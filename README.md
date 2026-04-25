# Lab 03: Security Groups · Group Policy Objects · Enforcement
 
![Platform](https://img.shields.io/badge/Platform-Windows%20Server%202025-blue?style=flat-square)
![Duration](https://img.shields.io/badge/Duration-45--90%20min-orange?style=flat-square)
![Cost](https://img.shields.io/badge/Estimated%20Cost-%240-brightgreen?style=flat-square)
 
---
 
## Overview
 
This section covers the handoff between identity and policy — the point where the security groups you built in last lab become the targets for centrally-enforced Group Policy Objects. A GPO without a target is inert. A target without a GPO has no enforcement. The last lab and this lab work together to create a domain where access is controlled and settings are consistent across every joined machine.
 
Group Policy is one of the most powerful — and most abused — tools in an Active Directory environment. Done correctly, it enforces security baselines across thousands of machines from a single place. Done carelessly, it introduces privilege escalation paths that attackers actively look for.
 
---
 
## Architecture
 
```
Security groups (RBAC)
┌─────────────────────────────────────────────────────────┐
│  IT-Admins (group)    HR-Staff (group)   Finance-Staff  │
│  OU=IT                OU=HR              OU=Finance      │
└───────────────────┬──────────────────────────┬──────────┘
                    │  groups are the scope     │
                    │  of policy application    │
                    ▼                           ▼
Step 6 — GPOs linked to OUs
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  Password Policy GPO          Workstation Lockout GPO   │
│  Linked to: domain root       Linked to: Workstations   │
│  ┌──────────────────┐         ┌──────────────────────┐  │
│  │ Min 12 chars     │         │ 10 min inactivity    │  │
│  │ 90-day max age   │         │ Screen lock enforced │  │
│  │ 10 pwd history   │         └──────────────────────┘  │
│  │ Complexity on    │                                   │
│  └──────────────────┘         Software Restriction GPO  │
│                               Linked to: Finance OU     │
│                               ┌──────────────────────┐  │
│                               │ Block unauthorized   │  │
│                               │ app execution        │  │
│                               └──────────────────────┘  │
└──────────────────────────────────────┬──────────────────┘
                                       │
                               gpupdate /force
                                       │
                                       ▼
                    Domain-joined machines receive policy
                    Settings enforced on next login
```
 
> **Why GPO scope matters:** A GPO linked to the `Finance` OU applies to every object inside that OU — and only objects inside that OU. A GPO linked to the domain root applies to every object in the domain. Scope is the most common thing to get wrong when troubleshooting GPO problems.
 
---
 
## Security Groups (Recap and Confirmation)
 
Before proceeding to GPO creation, confirm your security groups and OU structure are in place. GPOs are linked to OUs — if the OUs do not exist, the GPO links will fail.
 
```powershell
# Confirm OUs exist
Get-ADOrganizationalUnit -Filter * | Select Name, DistinguishedName
 
# Confirm groups exist
Get-ADGroup -Filter * | Select Name, GroupScope, DistinguishedName
 
# Confirm users are in their groups
Get-ADGroupMember "IT-Admins"     | Select Name
Get-ADGroupMember "HR-Staff"      | Select Name
Get-ADGroupMember "Finance-Staff" | Select Name
```
 
All five OUs and three security groups should be present before you proceed.
 
---
 
## Configure Group Policy Objects
 
> **Open Group Policy Management:** Server Manager → **Tools → Group Policy Management**. This is a completely separate application from Active Directory Users and Computers. You will not find GPO settings inside ADUC.
 
### Password Policy GPO
 
The password policy GPO applies to all user accounts across the entire domain. Link it at the domain root so it covers everyone.
 
#### Create and edit via GPMC
 
1. In the **Group Policy Management** tree, right-click **Group Policy Objects → New**
2. Name it: `Lab - Password Policy`
3. Right-click the new GPO → **Edit**
4. Navigate to: `Computer Configuration → Policies → Windows Settings → Security Settings → Account Policies → Password Policy`
5. Configure each setting in the table below
6. Close the editor
7. Right-click `lab.local` (domain root) in the tree → **Link an Existing GPO → Lab - Password Policy**
| Setting | Value | Rationale |
|---|---|---|
| Minimum password length | 12 characters | Industry baseline; long passwords resist brute force more than complex short ones |
| Password must meet complexity requirements | Enabled | Enforces mixed character types (uppercase, lowercase, number, symbol) |
| Maximum password age | 90 days | Limits the exposure window if a credential is silently compromised |
| Minimum password age | 1 day | Prevents users from cycling through the history immediately |
| Enforce password history | 10 passwords | Prevents immediate reuse of recent passwords |
| Store passwords using reversible encryption | Disabled | Never enable this — it stores passwords in a reversible format attackers can recover |
 
#### Set password policy via PowerShell
 
```powershell
# Set the default domain password policy
Set-ADDefaultDomainPasswordPolicy `
  -Identity "lab.local" `
  -MinPasswordLength 12 `
  -ComplexityEnabled $true `
  -MaxPasswordAge (New-TimeSpan -Days 90) `
  -MinPasswordAge (New-TimeSpan -Days 1) `
  -PasswordHistoryCount 10 `
  -ReversibleEncryptionEnabled $false
```
 
---
 
### Workstation Lockout GPO
 
The screen lock policy applies to all domain-joined workstations. Link it to the `Workstations` OU — every computer object moved into that OU automatically inherits this policy.
 
#### Create via GPMC
 
1. **Group Policy Objects → New** → Name: `Lab - Workstation Lockout`
2. Right-click → **Edit**
3. Navigate to: `Computer Configuration → Policies → Windows Settings → Security Settings → Local Policies → Security Options`
4. Set **Interactive logon: Machine inactivity limit** to `600` seconds (10 minutes)
5. Close editor → right-click `Workstations` OU → **Link an Existing GPO → Lab - Workstation Lockout**
#### Set via PowerShell
 
```powershell
New-GPO -Name "Lab - Workstation Lockout" | `
  New-GPLink -Target "OU=Workstations,DC=lab,DC=local"
 
Set-GPRegistryValue `
  -Name "Lab - Workstation Lockout" `
  -Key "HKLM\Software\Microsoft\Windows\CurrentVersion\Policies\System" `
  -ValueName "InactivityTimeoutSecs" `
  -Type DWord `
  -Value 600
```
 
---
 
### Software Restriction GPO (Finance OU)
 
The Finance department gets a stricter policy — only approved applications can run. Link this to the `Finance` OU only.
 
#### Create via GPMC
 
1. **Group Policy Objects → New** → Name: `Lab - Finance Software Restriction`
2. Right-click → **Edit**
3. Navigate to: `User Configuration → Policies → Windows Settings → Security Settings → Software Restriction Policies`
4. Right-click **Software Restriction Policies → New Software Restriction Policies**
5. Set **Default Security Level** to **Disallowed** — all software blocked unless explicitly permitted
6. Add exceptions for: `C:\Windows\*`, `C:\Program Files\*`, `C:\Program Files (x86)\*`
7. Close editor → right-click `Finance` OU → **Link an Existing GPO → Lab - Finance Software Restriction**
> **Important:** Test software restriction policies on a non-critical machine before linking them to production OUs. An overly restrictive policy can block legitimate applications including the ones users need to log in.
 
---
 
## Push and Verify Policy Delivery
 
Once GPOs are linked, policy is delivered on the next Group Policy refresh cycle (typically every 90 minutes for computers, every 90 minutes with a 30-minute random offset for users). To test immediately:
 
```powershell
# Run on the domain-joined machine you want to test
gpupdate /force
 
# Verify which GPOs are applied to the current machine
gpresult /r
 
# Get a detailed HTML report of applied policy
gpresult /h C:\GPO-Report.html
```
 
Open `C:\GPO-Report.html` in a browser for a full breakdown of which GPOs applied, which were filtered out, and what settings are active.
 
---
 
## GPO Security — What Attackers Look For
 
Understanding GPO security is important because misconfigured GPOs are a common privilege escalation path in real environments.
 
| Risk | What it means | How to prevent it |
|---|---|---|
| Excessive GPO write permissions | A user with write access to a GPO can modify its settings — including adding a malicious script to all machines in scope | Audit GPO permissions in GPMC regularly. Only Domain Admins should have write access to GPOs |
| GPO link over-scoping | A GPO linked to the domain root applies to every object in the domain, including DCs | Link GPOs to the most specific OU that covers the intended target. Never link non-DC policies to the Domain Controllers OU |
| Startup/logon scripts in GPOs | Attackers who can write to a GPO can add a startup script that runs as SYSTEM on every machine in scope | Monitor GPO changes with audit logging enabled |
 
---
 
## Verification Checklist
 
```powershell
# Confirm GPOs exist
Get-GPO -All | Select DisplayName, GpoStatus
 
# Check which GPOs are linked to the domain root
(Get-GPInheritance -Target "DC=lab,DC=local").GpoLinks
 
# Check which GPOs are linked to the Workstations OU
(Get-GPInheritance -Target "OU=Workstations,DC=lab,DC=local").GpoLinks
 
# Check the password policy is active
Get-ADDefaultDomainPasswordPolicy
```
 
---
 
## Troubleshooting
 
| Issue | Cause | Fix |
|---|---|---|
| GPO not applying to machines | GPO linked but `gpupdate` not run | Run `gpupdate /force` on the target machine |
| `gpresult /r` shows GPO was filtered | Security filtering excludes the account | Check that **Authenticated Users** has Read + Apply Group Policy on the GPO |
| Password policy change not effective | Domain-level policy takes precedence | The default domain password policy overrides OU-level password policies. Use Fine-Grained Password Policies (FGPP) for per-group password rules |
| Software restriction blocks legitimate apps | Path rules too narrow | Add the application's installation folder as an explicit **Unrestricted** path rule |
| GPO appears in GPMC but settings not showing | GPO opened before GPMC fully loaded | Close and reopen the GPO editor |
 
---
 
## Key Concepts
 
| Term | Definition |
|---|---|
| Group Policy Object (GPO) | A collection of settings applied to users or computers within a defined scope |
| GPO Link | The connection between a GPO and an OU, site, or domain that determines scope |
| GPO Inheritance | Child OUs inherit GPOs from parent OUs unless inheritance is blocked |
| Security Filtering | Controls which users/groups a GPO applies to within the linked scope |
| `gpupdate /force` | Forces immediate re-application of all applicable Group Policy settings |
| `gpresult` | Reports which GPOs are applied to a user or computer and their resultant settings |
| Software Restriction Policy | GPO feature that controls which executables are allowed to run |
| Fine-Grained Password Policy | Per-group password policy that overrides the domain default |
