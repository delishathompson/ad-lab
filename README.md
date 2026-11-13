# AD Lab: A Small Windows Domain Built on My Own Laptop

I built a small company-style network inside VirtualBox: one server that manages the users and computers, and one client PC that joins it. Then I practiced the problems a help desk sees every day (locked accounts, password resets, broken DNS, shared-folder access) and wrote up how I fixed each one.

**Cost:** $0. VirtualBox is free, and both Windows copies are Microsoft evaluation versions.

---

## What I built

| Machine | Role | Address |
|---|---|---|
| **DC01** | Windows Server 2022. Runs Active Directory, DNS, and DHCP | 192.168.50.10 (fixed) |
| **PC02** | Windows 10 Enterprise (evaluation) client PC | Gets an address from DHCP (192.168.50.100 to .200) |

Both machines sit on one private VirtualBox network called **labnet**, so the lab can't interfere with my home network.

The domain is called **corp.lab**.

## Network layout

All traffic stays on **labnet**, a private VirtualBox network (192.168.50.0/24) with no internet.

| Machine | Address | Where the address comes from | DNS server it uses | Network |
|---|---|---|---|---|
| DC01 | 192.168.50.10 | Set by hand (fixed) | 192.168.50.10 (itself) | labnet |
| PC02 | 192.168.50.100 to .200 | DHCP on DC01 | 192.168.50.10 (DC01) | labnet |

---

## The pieces in plain terms

- **Active Directory (AD):** The company phone book and security guard in one. It stores every user, group, and computer, and decides who may sign in.
- **DNS:** Turns names into addresses (corp.lab into 192.168.50.10). AD depends on it, because PCs find the domain controller by name.
- **DHCP:** Hands out addresses automatically so nobody types them by hand.
- **OUs (organizational units):** Folders inside AD. I made Corp > Users, Workstations, and Groups. Policies attach to OUs.
- **Groups:** I give access to a group and put people in the group. That keeps access easy to add and remove.
- **Group Policy (GPO):** Rules applied automatically to users or computers in an OU.
- **File share:** A shared folder (Sales) only the Sales group can open. Two layers decide access: share permissions and NTFS permissions, and the stricter one wins.

---

## How I built it, in order

Each block shows the commands I used, with a plain-language note on what each does. No passwords are stored here.

### 1. Name the server and give it a fixed address (DC01)

```powershell
# Show the network adapter name (mine was Ethernet)
Get-NetAdapter

# Give the server a fixed address
New-NetIPAddress -InterfaceAlias 'Ethernet' -IPAddress 192.168.50.10 -PrefixLength 24

# Make the server use itself for DNS
Set-DnsClientServerAddress -InterfaceAlias 'Ethernet' -ServerAddresses 192.168.50.10

# Rename the computer and restart
Rename-Computer -NewName DC01 -Restart
```

### 2. Install Active Directory and create the domain

```powershell
# Install the Active Directory software
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools

# Turn this server into the first domain controller of a new domain called corp.lab
Install-ADDSForest -DomainName 'corp.lab' -DomainNetbiosName 'CORP' -InstallDns

# After the restart, check it works
Get-ADDomain
Get-Service NTDS,DNS,Netlogon
nslookup corp.lab
```

### 3. Set up DHCP

```powershell
# Install DHCP and approve it in AD (an unapproved DHCP server does nothing)
Install-WindowsFeature DHCP -IncludeManagementTools
Add-DhcpServerInDC -DnsName DC01.corp.lab -IPAddress 192.168.50.10
Add-DhcpServerSecurityGroup
Restart-Service dhcpserver

# Create the pool of addresses (.100 to .200) and tell clients which DNS to use
Add-DhcpServerv4Scope -Name 'LabScope' -StartRange 192.168.50.100 -EndRange 192.168.50.200 -SubnetMask 255.255.255.0 -State Active
Set-DhcpServerv4OptionValue -ScopeId 192.168.50.0 -DnsServer 192.168.50.10 -DnsDomain corp.lab
```

### 4. Create the OUs, groups, and users

```powershell
# OUs (folders in AD)
New-ADOrganizationalUnit -Name 'Corp' -Path 'DC=corp,DC=lab'
New-ADOrganizationalUnit -Name 'Users' -Path 'OU=Corp,DC=corp,DC=lab'
New-ADOrganizationalUnit -Name 'Workstations' -Path 'OU=Corp,DC=corp,DC=lab'
New-ADOrganizationalUnit -Name 'Groups' -Path 'OU=Corp,DC=corp,DC=lab'

# Groups
New-ADGroup -Name 'Sales' -GroupScope Global -GroupCategory Security -Path 'OU=Groups,OU=Corp,DC=corp,DC=lab'
New-ADGroup -Name 'HelpDesk' -GroupScope Global -GroupCategory Security -Path 'OU=Groups,OU=Corp,DC=corp,DC=lab'

# Users (four test users, created with a loop; the password was typed at a hidden prompt)
# Example for one user:
New-ADUser -Name 'Ava Stone' -SamAccountName astone -UserPrincipalName astone@corp.lab -Path 'OU=Users,OU=Corp,DC=corp,DC=lab' -AccountPassword $pw -Enabled $true

# Put people into groups
Add-ADGroupMember -Identity 'Sales' -Members 'astone','bortiz'
Add-ADGroupMember -Identity 'HelpDesk' -Members 'clee'
```

### 5. Build the client PC and join it to the domain

On the client I checked that it received a lab address and could find the domain, then joined it.

```powershell
# Did I get an address from DHCP, and can I find the domain?
ipconfig /all
nslookup corp.lab

# Join the domain (asks for the domain admin login, administrator@corp.lab)
Add-Computer -DomainName corp.lab -Credential (Get-Credential) -Restart
```

On DC01, I found the new computer and moved it into the Workstations OU. The PC had joined under a random Windows name (DESKTOP-SNN1...), so I found it by that prefix, then renamed it.

```powershell
# On DC01: list every computer in the domain
Get-ADComputer -Filter * | Select-Object Name,DistinguishedName

# On DC01: move the PC into the Workstations OU
Get-ADComputer -Filter "Name -like 'DESKTOP-*'" | Move-ADObject -TargetPath 'OU=Workstations,OU=Corp,DC=corp,DC=lab'

# On the PC: rename it to PC02 (uses domain admin credentials)
Rename-Computer -NewName PC02 -DomainCredential (Get-Credential) -Restart
```

### 6. Create the shared folder

```powershell
New-Item -Path 'C:\Shares\Sales' -ItemType Directory -Force
New-SmbShare -Name 'Sales' -Path 'C:\Shares\Sales' -FullAccess 'CORP\Domain Admins' -ChangeAccess 'CORP\Sales'
icacls 'C:\Shares\Sales' /grant 'CORP\Sales:(OI)(CI)M'
```

### 7. Group Policies

I created these in the Group Policy Management tool (gpmc.msc) and linked each one to an OU.

| Policy | Linked to | What it does | How I checked it |
|---|---|---|---|
| User - Map Sales Drive | Users OU | Gives members of Sales an S: drive pointing to the Sales share | Signed in as a Sales user and saw the S: drive; a non-Sales user did not |
| User - Block Control Panel | Users OU (filtered to Sales) | Stops Sales users from opening Control Panel | Signed in as a Sales user and got the restriction message |
| Computer - Screen Lock | Workstations OU | Locks the screen after 10 minutes of inactivity | `gpresult /r` on PC02 listed it under Computer Settings |

To check which policies reached a PC, I ran this on the PC after signing in:

```powershell
gpupdate /force
gpresult /r
```

### 8. Account lockout rule

By default AD never locks accounts. I set the Default Domain Policy to lock an account after 5 wrong passwords, so I could practice unlocking one.

---

## Screenshots

TODO: take these on PC02 and put the files in a `screenshots` folder next to this README. (Press Win+Shift+S to capture.)

- `ipconfig /all` showing the DHCP address (192.168.50.100 to .200), DNS server 192.168.50.10, and the corp.lab suffix:

  ![ipconfig output on PC02](screenshots/ipconfig-all.png)

- `gpresult /r` showing the three policies applied:

  ![gpresult output on PC02](screenshots/gpresult-r.png)

---

## Help desk tickets

Each row is a problem I hit or practiced, with how I fixed it.

| Symptom reported | What I checked | Cause | Fix | How I confirmed |
|---|---|---|---|---|
| Joining the domain with `Add-Computer` gave "access denied" | Whether PowerShell said Administrator in the title bar, which credentials I typed, and whether `nslookup corp.lab` worked | TODO: write the one that applied (not running as administrator, wrong credentials, or DNS not pointing at DC01) | Reopened PowerShell as administrator and used `administrator@corp.lab` with the domain Administrator password | The PC restarted and appeared in Active Directory |
| Could not find the PC to move it into the Workstations OU | `Get-ADComputer -Filter * \| Select-Object Name,DistinguishedName` on DC01 | The PC had joined under a random name (DESKTOP-SNN1...), not PC02 | Moved it using the DESKTOP- prefix, then renamed it to PC02 | `Get-ADComputer PC02` showed it under OU=Workstations |
| Creating the file share failed with a positional-parameter error and a "1332" error | The red error text, `Get-ADGroup -Identity Sales`, and `(Get-ADDomain).NetBIOSName` | Several commands were pasted onto one line, so PowerShell read `icacls` as part of `New-SmbShare`. Error 1332 means Windows could not match a group name to a real account. TODO: write whether the group or the domain name was the cause | Ran each command on its own line and confirmed the group existed | `Get-SmbShare` listed Sales and `icacls` reported success |
| Could not sign in to PC02 with the local admin account | `Get-LocalUser` and `Get-LocalGroupMember -Group Administrators` after signing in as `CORP\Administrator` | The local admin password was forgotten. Local and domain accounts are separate | Reset it with `Set-LocalUser -Name localadmin -Password (Read-Host -AsSecureString 'New password')` | Signed in as `.\localadmin` and `whoami` showed `pc02\localadmin` |
| `Remove-Computer` gave "access denied" during the missing-account drill | Whether PowerShell was elevated and whether the PC's account still existed in AD | The computer's AD account had been deleted, so the normal unjoin could not contact it | Left the domain with the WMI method `(Get-WmiObject Win32_ComputerSystem).UnjoinDomainOrWorkgroup($null,$null,0)`, restarted, then rejoined | `Test-ComputerSecureChannel` and a domain sign-in worked after rejoining |
| User cannot sign in after several tries | Account status in AD | Account locked after 5 bad passwords | `Unlock-ADAccount -Identity bortiz` | User signed in successfully |
| User forgot their password | Confirmed who the user was | Forgotten password | `Set-ADAccountPassword -Identity astone -Reset` and `Set-ADUser -Identity astone -ChangePasswordAtLogon $true` | User signed in and was asked to choose a new password |
| PC cannot reach the domain | `nslookup corp.lab` and `gpupdate /force` on the PC | I pointed the PC's DNS at a wrong address (192.168.50.99) | `Set-DnsClientServerAddress -InterfaceAlias 'Ethernet' -ResetServerAddresses`, then `ipconfig /renew` | `nslookup corp.lab` answered 192.168.50.10 and `gpupdate` worked |
| User lost access to the Sales drive after leaving the group | Group membership in AD and whether the user had signed out | Group changes only reach a user the next time they sign in | Removed the user from Sales, then signed out and back in | S: drive was gone and the share said Access Denied |
| User's account stopped working | Whether the account was disabled in AD | Account was disabled (this is the first step of offboarding) | `Enable-ADAccount -Identity dpatel` | User signed in successfully |

TODO: delete any row for something you did not actually do, and edit any detail that differs from what happened. Everything here should be something you can explain in an interview.

---

## What I learned

- A domain depends on DNS. Most "can't sign in" problems turned out to be name resolution.
- A computer account and its domain share a secret. Delete one side and the trust breaks.
- Local accounts and domain accounts are separate, so keep a working local admin as a way back in.
- Share permissions and NTFS permissions both apply, and the stricter one wins.
- Group membership is read at sign-in, so a user must sign out and back in to see a change.
- Snapshots in VirtualBox let me break things on purpose and roll back.
- When a command fails, the exact error text, plus one command that checks the assumption, finds the cause faster than guessing.

## Cheat sheet of checks I now use

| Question | Command |
|---|---|
| Did I get an address, and from where? | `ipconfig /all` |
| Can I find the domain? | `nslookup corp.lab` |
| Which policies applied? | `gpresult /r` |
| Is an account locked? | `Search-ADAccount -LockedOut` |
| Is the PC trusted by the domain? | `Test-ComputerSecureChannel` |
| Who am I signed in as? | `whoami` |
| Where is a computer in AD? | `Get-ADComputer <name> \| Select-Object DistinguishedName` |
