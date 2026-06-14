# Windows Server 2022 Active Directory Home Lab

A fully functional Active Directory domain lab built on a personal laptop using VirtualBox. This project documents the end-to-end setup of a Windows Server 2022 Domain Controller and a Windows 11 client machine — including all errors encountered and how they were resolved.

---

## Table of Contents

- [Lab Overview](#lab-overview)
- [Tools & Requirements](#tools--requirements)
- [Network Architecture](#network-architecture)
- [Phase 1 — VirtualBox Network Setup](#phase-1--virtualbox-network-setup)
- [Phase 2 — DC01 VM Creation](#phase-2--dc01-vm-creation)
- [Phase 3 — Windows Server Installation](#phase-3--windows-server-installation)
- [Phase 4 — Static IP Configuration](#phase-4--static-ip-configuration)
- [Phase 5 — Installing AD DS, DNS, DHCP](#phase-5--installing-ad-ds-dns-dhcp)
- [Phase 6 — Promoting to Domain Controller](#phase-6--promoting-to-domain-controller)
- [Phase 7 — DHCP Scope Configuration](#phase-7--dhcp-scope-configuration)
- [Phase 8 — PC01 VM Creation & Windows 11 Install](#phase-8--pc01-vm-creation--windows-11-install)
- [Phase 9 — Joining PC01 to Domain](#phase-9--joining-pc01-to-domain)
- [Phase 10 — Active Directory Configuration](#phase-10--active-directory-configuration)
- [Phase 11 — Group Policy Objects](#phase-11--group-policy-objects)
- [Phase 12 — Shared Folder & Permissions](#phase-12--shared-folder--permissions)
- [Phase 13 — Testing & Verification](#phase-13--testing--verification)
- [Errors & Fixes](#errors--fixes)

---

## Lab Overview

| Component | Details |
|---|---|
| Host OS | Windows 11 (Lenovo laptop) |
| Hypervisor | Oracle VirtualBox |
| Server VM | Windows Server 2022 Standard Evaluation (Desktop Experience) |
| Client VM | Windows 11 Enterprise LTSC Evaluation |
| Domain | company.local |
| Network | NAT Network — 192.168.10.0/24 |

**What this lab demonstrates:**
- Active Directory Domain Services deployment
- DNS and DHCP server configuration
- Organizational Unit and user management
- Group Policy Object creation and enforcement
- SMB shared folder with role-based access
- Domain client join and authentication

---

## Tools & Requirements

- Oracle VirtualBox (free) — [virtualbox.org](https://www.virtualbox.org)
- Windows Server 2022 Evaluation ISO — [Microsoft Eval Center](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2022)
- Windows 11 Enterprise LTSC Evaluation ISO — [Microsoft Eval Center](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-11-enterprise)
- Minimum 8 GB RAM on host (16 GB recommended)
- Minimum 120 GB free disk space

---

## Network Architecture

```
Host Laptop (Windows 11)
└── VirtualBox NAT Network: LabNAT (192.168.10.0/24)
    ├── DC01 — Windows Server 2022
    │   IP: 192.168.10.10 (static)
    │   Roles: AD DS, DNS, DHCP
    │   Domain: company.local
    │
    └── PC01 — Windows 11 Enterprise
        IP: 192.168.10.100+ (via DHCP from DC01)
        Joined to: company.local
```

Gateway: 192.168.10.1 (VirtualBox NAT)
DHCP Pool: 192.168.10.100 — 192.168.10.200

---

## Phase 1 — VirtualBox Network Setup

Before creating any VMs, a shared NAT Network must be created so both VMs can communicate with each other.

**Steps:**
1. Open VirtualBox → click **Tools** in the left sidebar
2. Click **NAT Networks** tab → click **Create**
3. Configure:

```
Name:        LabNAT
IPv4 Prefix: 192.168.10.0/24
DHCP:        Disabled (DC01 will handle DHCP)
```

4. Click **Apply**

> **Why NAT Network and not plain NAT?**
> Plain NAT gives each VM its own isolated network — VMs cannot see each other. NAT Network puts all VMs on the same subnet, allowing DC01 and PC01 to communicate while still sharing the host's internet connection.

---

## Phase 2 — DC01 VM Creation

**Settings used:**

| Setting | Value |
|---|---|
| Name | DC01 |
| ISO | SERVER\_EVAL\_x64FRE\_en-us.iso |
| Type | Microsoft Windows |
| Version | Windows Server 2022 (64-bit) |
| RAM | 4096 MB |
| CPUs | 2 |
| Disk | 60 GB (VDI, dynamically allocated) |
| Skip Unattended Install | ✅ Checked |

**After creation — set network adapter:**
Settings → Network → Adapter 1:
```
Attached to: NAT Network
Name:        LabNAT
```

> **Why skip unattended installation?**
> Unattended install lets VirtualBox auto-configure the OS but often causes issues with Server editions. Manual install gives full control over edition selection (Core vs Desktop Experience).

---

## Phase 3 — Windows Server Installation

1. Start DC01 → Windows Setup launches
2. Select edition: **Windows Server 2022 Standard Evaluation (Desktop Experience)**

> The Desktop Experience option includes the full GUI. Without it, you get Server Core — command line only. For a learning lab, Desktop Experience is preferred.

3. Installation type: **Custom (clean install)**
4. Select the unallocated disk → Next
5. Wait ~15 minutes for installation
6. Set Administrator password when prompted: use a complex password (e.g. `Admin@123`)

**After first login — remove ISO from virtual drive:**

Settings → Storage → click the ISO → Remove Disk from Virtual Drive

Then set boot order: Settings → System → Motherboard → move **Hard Disk** above **Optical**.

---

## Phase 4 — Static IP Configuration

A Domain Controller must have a fixed IP. If the IP changes, clients lose DNS and cannot authenticate to the domain.

Open PowerShell as Administrator on DC01:

```powershell
# Assign static IP
New-NetIPAddress `
  -InterfaceAlias "Ethernet" `
  -IPAddress 192.168.10.10 `
  -PrefixLength 24 `
  -DefaultGateway 192.168.10.1

# Point DNS to itself (DC will run its own DNS after AD install)
Set-DnsClientServerAddress `
  -InterfaceAlias "Ethernet" `
  -ServerAddresses 127.0.0.1
```

**Verify:**
```powershell
ipconfig /all
```

Expected output:
```
IPv4 Address:    192.168.10.10
Subnet Mask:     255.255.255.0
Default Gateway: 192.168.10.1
DNS Servers:     127.0.0.1
```

> **Why DNS points to 127.0.0.1?**
> Once AD DS is installed, this machine runs its own DNS server. Pointing to itself ensures AD-related DNS records (like `company.local`) resolve correctly. If pointed to an external DNS (e.g. 8.8.8.8), domain lookups would fail.

---

## Phase 5 — Installing AD DS, DNS, DHCP

```powershell
Install-WindowsFeature -Name AD-Domain-Services,DNS,DHCP -IncludeManagementTools
```

> **Note:** Commas between feature names must have no spaces, and `-IncludeManagementTools` must not be preceded by a comma — this was a syntax error encountered during setup (see Errors & Fixes).

**What each role does:**

| Role | Purpose |
|---|---|
| AD-Domain-Services | Core engine for Active Directory — users, groups, OUs, GPOs |
| DNS | Translates domain names like `company.local` to IPs. Required by AD |
| DHCP | Automatically assigns IPs to client machines joining the network |
| -IncludeManagementTools | Installs GUI tools (AD Users & Computers, DNS Manager, DHCP Console) |

Expected output: `Success: True`

---

## Phase 6 — Promoting to Domain Controller

```powershell
Install-ADDSForest -DomainName "company.local" -InstallDns
```

- Enter a DSRM (Directory Services Restore Mode) password when prompted
- Type `Y` to confirm
- Server reboots automatically

**Verify after reboot:**
```powershell
Get-ADDomain
```

Key fields to confirm:
```
DNSRoot:     company.local
Forest:      company.local
NetBIOSName: COMPANY
```

> After promotion, login changes from `Administrator` to `COMPANY\Administrator` — this confirms the machine is now a Domain Controller.

---

## Phase 7 — DHCP Scope Configuration

```powershell
# Fix DHCP service startup
Set-Service -Name DHCPServer -StartupType Automatic
Start-Service DHCPServer

# Authorize DHCP server in Active Directory
Add-DhcpServerInDC -DnsName "DC01.company.local" -IPAddress 192.168.10.10

# Import DHCP module
Import-Module DhcpServer

# Create IP scope
Add-DhcpServerv4Scope `
  -Name "LabScope" `
  -StartRange 192.168.10.100 `
  -EndRange 192.168.10.200 `
  -SubnetMask 255.255.255.0

# Set default gateway and DNS for clients
Set-DhcpServerv4OptionValue `
  -ScopeId 192.168.10.0 `
  -Router 192.168.10.1 `
  -DnsServer 192.168.10.10
```

**Verify:**
```powershell
Get-DhcpServerv4Scope
Get-DhcpServerv4OptionValue -ScopeId 192.168.10.0
```

Expected:
```
ScopeId       State   StartRange        EndRange
192.168.10.0  Active  192.168.10.100    192.168.10.200

OptionId  Name        Value
3         Router      192.168.10.1
6         DNS Servers 192.168.10.10
```

---

## Phase 8 — PC01 VM Creation & Windows 11 Install

**VM Settings:**

| Setting | Value |
|---|---|
| Name | PC01 |
| ISO | 26100.1742...CLIENT\_LTSC\_EVAL\_x64FRE\_en-us.iso |
| Version | Windows 11 (64-bit) |
| RAM | 4096 MB |
| CPUs | 2 |
| Disk | **64 GB** (minimum for Windows 11) |
| EFI | Enabled |
| TPM | v2.0 |
| Secure Boot | Enabled |

**Windows 11 TPM bypass (required for VMs):**

Windows 11 checks for TPM, Secure Boot, and disk size during install. In VMs these checks often fail even with correct settings. Bypass via registry before installer runs:

Press `Shift+F10` at the first setup screen to open command prompt, then:

```cmd
reg add HKLM\SYSTEM\Setup\LabConfig /v BypassTPMCheck /t REG_DWORD /d 1 /f
reg add HKLM\SYSTEM\Setup\LabConfig /v BypassSecureBootCheck /t REG_DWORD /d 1 /f
reg add HKLM\SYSTEM\Setup\LabConfig /v BypassRAMCheck /t REG_DWORD /d 1 /f
reg add HKLM\SYSTEM\Setup\LabConfig /v BypassCPUCheck /t REG_DWORD /d 1 /f
```

Close cmd → proceed with installation.

**During OOBE (Out of Box Experience):**
- Network: click **"I don't have internet"**
- Account name: `LocalUser` (temporary local account)
- Password: leave blank
- All privacy settings: **No/Decline**

---

## Phase 9 — Joining PC01 to Domain

Ensure DC01 is running first. Then on PC01, open PowerShell as Administrator:

```powershell
Add-Computer -DomainName "company.local" -Credential COMPANY\Administrator -Restart
```

Enter DC01 Administrator password when prompted. PC01 reboots automatically.

**Verify network connectivity before joining:**
```powershell
ping 192.168.10.10
```
Should return replies — if it times out, check both VMs are on LabNAT network.

**After reboot — login with domain account:**
- Click **Other user** on login screen
- Username: `COMPANY\Administrator`
- Password: DC01 admin password

---

## Phase 10 — Active Directory Configuration

Run on DC01 PowerShell:

```powershell
# Create Organizational Units
New-ADOrganizationalUnit -Name "IT"      -Path "DC=company,DC=local"
New-ADOrganizationalUnit -Name "HR"      -Path "DC=company,DC=local"
New-ADOrganizationalUnit -Name "Finance" -Path "DC=company,DC=local"

# Create domain users
New-ADUser `
  -Name "John Doe" `
  -SamAccountName "john.doe" `
  -UserPrincipalName "john.doe@company.local" `
  -Path "OU=HR,DC=company,DC=local" `
  -AccountPassword (ConvertTo-SecureString "User@123" -AsPlainText -Force) `
  -Enabled $true

New-ADUser `
  -Name "Jane Smith" `
  -SamAccountName "jane.smith" `
  -UserPrincipalName "jane.smith@company.local" `
  -Path "OU=IT,DC=company,DC=local" `
  -AccountPassword (ConvertTo-SecureString "User@123" -AsPlainText -Force) `
  -Enabled $true

New-ADUser `
  -Name "Test User" `
  -SamAccountName "test.user" `
  -UserPrincipalName "test.user@company.local" `
  -Path "OU=Finance,DC=company,DC=local" `
  -AccountPassword (ConvertTo-SecureString "User@123" -AsPlainText -Force) `
  -Enabled $true
```

---

## Phase 11 — Group Policy Objects

### Password Policy (domain-wide)

```powershell
New-GPO -Name "Password Policy" | New-GPLink -Target "DC=company,DC=local"

Set-GPRegistryValue `
  -Name "Password Policy" `
  -Key "HKLM\SYSTEM\CurrentControlSet\Services\Netlogon\Parameters" `
  -ValueName "RequireStrongKey" `
  -Type DWORD -Value 1

# Set domain password policy
Set-ADDefaultDomainPasswordPolicy `
  -Identity "company.local" `
  -MinPasswordLength 8 `
  -ComplexityEnabled $true `
  -MaxPasswordAge 90.00:00:00 `
  -MinPasswordAge 1.00:00:00 `
  -PasswordHistoryCount 5
```

### Block USB Storage (domain-wide)

```powershell
New-GPO -Name "Block USB" | New-GPLink -Target "DC=company,DC=local"

Set-GPRegistryValue `
  -Name "Block USB" `
  -Key "HKLM\SYSTEM\CurrentControlSet\Services\USBSTOR" `
  -ValueName "Start" `
  -Type DWORD -Value 4
```

> Setting USBSTOR Start value to 4 disables the USB storage driver. Value 3 = enabled, value 4 = disabled.

### Disable Control Panel (HR OU only)

```powershell
New-GPO -Name "Disable Control Panel" | New-GPLink -Target "OU=HR,DC=company,DC=local"

Set-GPRegistryValue `
  -Name "Disable Control Panel" `
  -Key "HKCU\Software\Microsoft\Windows\CurrentVersion\Policies\Explorer" `
  -ValueName "NoControlPanel" `
  -Type DWORD -Value 1
```

> This GPO is linked only to the HR OU — IT and Finance users are unaffected. This demonstrates per-OU policy scoping.

---

## Phase 12 — Shared Folder & Permissions

```powershell
# Create IT security group
New-ADGroup -Name "IT" -GroupScope Global -Path "OU=IT,DC=company,DC=local"

# Create shared folder
New-Item -Path "C:\Shares\IT_Share" -ItemType Directory

# Create SMB share with permissions
New-SmbShare `
  -Name "IT_Share" `
  -Path "C:\Shares\IT_Share" `
  -FullAccess "COMPANY\Domain Admins" `
  -ReadAccess "COMPANY\IT"
```

| Group | Permission |
|---|---|
| COMPANY\Domain Admins | Full Control |
| COMPANY\IT | Read |

---

## Phase 13 — Testing & Verification

**Test 1 — Domain user login on PC01:**
- Sign out of PC01
- Click Other User
- Login as `COMPANY\john.doe` / `User@123`
- ✅ Successfully logged in as domain user

**Test 2 — Network connectivity:**
```powershell
ping 192.168.10.10   # from PC01 — should reply
```

**Test 3 — Verify AD objects on DC01:**
```powershell
Get-ADUser -Filter * | Select Name, SamAccountName, Enabled
Get-ADOrganizationalUnit -Filter * | Select Name
Get-GPO -All | Select DisplayName, GpoStatus
```

---

## Errors & Fixes

### Error 1 — Install-WindowsFeature syntax error
**Command:** `Install-WindowsFeature -Name AD-Domain-Services, DNS, DHCP, -IncludeManagementTools`
**Error:** `The role, role service, or feature name is not valid: '-IncludeManagementTools'`
**Cause:** Comma before `-IncludeManagementTools` made PowerShell treat it as a feature name. Spaces between feature names also caused issues.
**Fix:** Remove all spaces between feature names and remove the comma before the flag:
```powershell
Install-WindowsFeature -Name AD-Domain-Services,DNS,DHCP -IncludeManagementTools
```

---

### Error 2 — DHCP RPC server unavailable (Error 1722)
**Error:** `Failed to initiate the authorization check on the DHCP server. Error: The RPC server is unavailable. (1722)`
**Cause:** DHCP service hadn't fully started after AD DS promotion and reboot.
**Fix:**
```powershell
Set-Service -Name DHCPServer -StartupType Automatic
Start-Service DHCPServer
Restart-Service DHCPServer
```

---

### Error 3 — Add-DhcpServerv4Scope not recognized
**Error:** `The term 'Add-DhcpServer4Scope' is not recognized`
**Cause:** DHCP PowerShell module not loaded in current session.
**Fix:**
```powershell
Import-Module DhcpServer
```

---

### Error 4 — DC01 booting from ISO after reboot
**Problem:** After Windows Server installation, VM kept booting back into the setup ISO.
**Cause:** Boot order had Optical drive above Hard Disk, and ISO was still attached.
**Fix:**
1. Settings → Storage → Remove ISO from optical drive
2. Settings → System → Motherboard → Move Hard Disk above Optical in boot order

---

### Error 5 — Windows 11 TPM/system requirements error
**Error:** `This PC doesn't currently meet Windows 11 system requirements`
**Cause:** Windows 11 installer checks for TPM 2.0, Secure Boot, and minimum 52 GB disk. VM environment triggers false negatives on these checks.
**Fix:** Press `Shift+F10` at first setup screen → add registry bypass keys:
```cmd
reg add HKLM\SYSTEM\Setup\LabConfig /v BypassTPMCheck /t REG_DWORD /d 1 /f
reg add HKLM\SYSTEM\Setup\LabConfig /v BypassSecureBootCheck /t REG_DWORD /d 1 /f
reg add HKLM\SYSTEM\Setup\LabConfig /v BypassRAMCheck /t REG_DWORD /d 1 /f
reg add HKLM\SYSTEM\Setup\LabConfig /v BypassCPUCheck /t REG_DWORD /d 1 /f
```
> Note: These keys must be added every boot as the installer environment is fresh each time.

---

### Error 6 — Windows 11 disk size requirement (52 GB minimum)
**Error:** `The system drive needs to be at least 52 GB or larger`
**Cause:** Initial disk size was set to 50 GB. Windows 11 requires minimum 52 GB.
**Fix:** Deleted VM and recreated with **64 GB** disk.

---

### Error 7 — New-SmbShare account mapping error
**Error:** `No mapping between account names and security IDs was done`
**Cause:** Referenced `COMPANY\IT` security group in share permissions before the group existed in AD.
**Fix:** Create the AD group first, then create the share:
```powershell
New-ADGroup -Name "IT" -GroupScope Global -Path "OU=IT,DC=company,DC=local"
New-SmbShare -Name "IT_Share" -Path "C:\Shares\IT_Share" -FullAccess "COMPANY\Domain Admins" -ReadAccess "COMPANY\IT"
```

---

### Error 8 — Domain join "username or password incorrect"
**Error:** `Computer failed to join domain. The user name or password is incorrect`
**Cause:** Password typed incorrectly in VM due to keyboard input lag — the `@` character in `Admin@123` was mistyped.
**Fix:** Typed password slowly and carefully. Verified password on DC01 first via `net user Administrator`.

---

## Screenshots

| Screenshot | Description |
|---|---|
| `screenshots/01-natnetwork.png` | LabNAT NAT Network configured in VirtualBox |
| `screenshots/02-dc01-ipconfig.png` | Static IP verified on DC01 |
| `screenshots/03-addomain.png` | Get-ADDomain output confirming company.local |
| `screenshots/04-dhcp-scope.png` | DHCP scope active with correct options |
| `screenshots/05-pc01-joined.png` | PC01 domain join success |
| `screenshots/06-john-doe-login.png` | john.doe logged into PC01 as domain user |
| `screenshots/07-gpo-list.png` | GPOs created and linked |

---

*Lab completed: June 2026 | Tools: VirtualBox, Windows Server 2022, Windows 11 Enterprise LTSC*
