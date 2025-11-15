# Complete SCCM PXE Boot Setup and Troubleshooting Guide

**Created:** 14 November 2025
**Environment:** SCCM Current Branch, Windows 11, Proxmox VMs
**Reference Videos:**
- [SCCM PXE Boot Setup](https://youtu.be/wCElfLBLyX0?si=vx9XmuajHViV0hk7)
- [SCCM Configuration](https://youtu.be/BPcy_nOQZoI?si=zXGUO319AdeGMFr2)

---

## Table of Contents

1. [Prerequisites and Downloads](#prerequisites-and-downloads)
2. [Initial SCCM Setup](#initial-sccm-setup)
3. [DHCP Configuration for PXE Boot](#dhcp-configuration-for-pxe-boot)
4. [Distribution Point and PXE Configuration](#distribution-point-and-pxe-configuration)
5. [Boot Image Configuration](#boot-image-configuration)
6. [Task Sequence Deployment](#task-sequence-deployment)
7. [Troubleshooting PXE Boot Issues](#troubleshooting-pxe-boot-issues)
8. [Proxmox-Specific Configuration](#proxmox-specific-configuration)
9. [Common Errors and Solutions](#common-errors-and-solutions)

---

## Prerequisites and Downloads

### Required Software Components

#### 1. Windows ADK (Assessment and Deployment Kit)

**CRITICAL: ADK version MUST match your Windows version and SCCM version!**

- **Windows 11 ADK**: Download from [Microsoft ADK Downloads](https://learn.microsoft.com/en-us/windows-hardware/get-started/adk-install)
- **WinPE Add-on for ADK**: REQUIRED - separate download from same page

**Version Compatibility:**
- SCCM Current Branch 2309 → ADK for Windows 11, version 22H2 or later
- SCCM Current Branch 2303 → ADK for Windows 11, version 22H2
- Check [SCCM ADK Support](https://learn.microsoft.com/en-us/mem/configmgr/core/plan-design/configs/support-for-windows-adk)

**Installation Order:**
1. Install Windows ADK first
2. Install WinPE Add-on second
3. **Restart SCCM server** after installing both

**Components to Install:**
- Deployment Tools
- Windows Preinstallation Environment (Windows PE)
- User State Migration Tool (USMT) - optional but recommended

#### 2. Server Manager Roles and Features

**On SCCM Server (Distribution Point):**

Open Server Manager → Add Roles and Features:

**Roles:**
- ✅ Web Server (IIS)
  - Web Server → Common HTTP Features → Static Content
  - Web Server → Common HTTP Features → Default Document
  - Web Server → Application Development → ASP.NET 4.8
  - Web Server → Application Development → ISAPI Extensions
  - Web Server → Application Development → ISAPI Filters
  - Web Server → Security → Windows Authentication
  - Management Tools → IIS Management Console

**Features:**
- ✅ .NET Framework 3.5 (includes .NET 2.0 and 3.0)
- ✅ .NET Framework 4.8 Advanced Services
- ✅ Background Intelligent Transfer Service (BITS)
  - IIS Server Extension
  - Compact Server
- ✅ Remote Differential Compression

**On Domain Controller (if separate from SCCM):**

- ✅ DHCP Server role
- ✅ DNS Server role
- ✅ Active Directory Domain Services

#### 3. VirtIO Drivers for Proxmox (if using Proxmox VMs)

Download from: [Fedora VirtIO Drivers](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso)

Or install on a working Windows VM and extract drivers using the provided script.

**Critical Drivers Needed:**
- **NetKVM** - Red Hat VirtIO Ethernet Adapter (network)
- **viostor** - Red Hat VirtIO SCSI controller (boot-critical disk driver)
- **vioscsi** - Red Hat VirtIO SCSI pass-through (alternative disk driver)

---

## Initial SCCM Setup

### 1. Install SCCM Current Branch

Follow Microsoft documentation or reference videos above.

**Key configuration items:**
- Site code: Choose 3 characters (e.g., HLL, ABC, XYZ)
- Site server: Install on a domain-joined Windows Server
- SQL Server: Required (Express or full edition)

### 2. Verify ADK Installation in SCCM Console

**CRITICAL STEP - Often missed!**

After installing SCCM, verify ADK is recognized:

1. SCCM Console → Software Library → Operating Systems → Boot Images
2. Right-click on **Boot image (x64)** → Properties
3. Check if **Customization** and **Optional Components** tabs are visible

**If tabs are MISSING:**
- ADK version mismatch detected!
- SCCM cannot find the correct ADK installation
- **Fix:** Update boot image to use installed ADK

**To fix missing tabs:**

1. Right-click boot image → **Update Distribution Points**
2. Select: ✅ **"Reload this boot image from the current Windows ADK"**
3. Click OK
4. Wait 5 minutes for boot image to rebuild
5. Refresh SCCM console
6. Re-open boot image properties → Tabs should now appear!

**Why this happens:**
- SCCM was installed with an older ADK version
- You later installed a newer ADK
- SCCM boot images still reference the old ADK path
- Forcing a reload rebuilds the boot.wim using the new ADK

---

## DHCP Configuration for PXE Boot

### Understanding DHCP Options for PXE

**Option 66** - Boot Server Host Name:
- Specifies the IP address of the PXE server
- Value: IP address of your SCCM Distribution Point (e.g., 192.168.1.20)

**Option 67** - Bootfile Name:
- Specifies which boot file the client should download
- **CRITICAL:** Different values for BIOS vs UEFI!

**DO NOT set Option 67 if using SCCM PXE!**
- SCCM PXE provider automatically selects the correct boot file based on client architecture
- Setting Option 67 can cause boot failures

### DHCP Configuration Steps

**On Domain Controller (PowerShell as Administrator):**

```powershell
# Get DHCP scope
Get-DhcpServerv4Scope

# Set Option 66 - Boot Server IP
Set-DhcpServerv4OptionValue -ScopeId ip address -OptionId 66 -Value "ip address"

# REMOVE Option 67 if it exists (let SCCM handle it)
Remove-DhcpServerv4OptionValue -ScopeId ip address -OptionId 67

# Verify
Get-DhcpServerv4OptionValue -ScopeId ip address | Where-Object {$_.OptionId -in 66,67}
```

**Expected Result:**
```
OptionId   Name                 Value
--------   ----                 -----
66         Boot Server Host...  {ip address}
```

**Option 67 should NOT be listed** - SCCM will provide the boot file dynamically.

### If You Must Set Option 67 (Non-SCCM PXE):

**For BIOS clients:**
```
smsboot\x64\wdsnbp.com
```

**For UEFI x64 clients:**
```
SMSBoot\x64\wdsmgfw.efi
```

**For UEFI HTTP Boot (Architecture 16):**
```
SMSBoot\x64\wdsmgfw.efi
```

---

## Distribution Point and PXE Configuration

### 1. Enable PXE on Distribution Point

**SCCM Console:**

1. Administration → Distribution Points
2. Right-click your server → Properties
3. **PXE** tab:
   - ✅ Enable PXE support for clients
   - ✅ Allow this distribution point to respond to incoming PXE requests
   - ✅ **Enable unknown computer support** ← CRITICAL!
   - ✅ Require a password when computers use PXE (optional)
   - PXE response delay: 0 seconds (unless you have multiple PXE servers)

4. Click OK → WDS will be installed and configured automatically

### 2. Verify WDS Service is Running

**PowerShell:**

```powershell
Get-Service WDSServer | Select-Object Name, Status, DisplayName
```

**Expected:**
```
Name        Status  DisplayName
----        ------  -----------
WDSServer   Running Windows Deployment Services Server
```

### 3. Configure TFTP ReadFilter

**CRITICAL:** By default, WDS TFTP only allows access to `\boot\*` and `\tmp\*` directories.
SCCM boot images are in `SMSBoot\` and `SMSImages\` - these are **BLOCKED by default!**

**Registry Fix:**

```powershell
# Check current ReadFilter
(Get-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Services\WDSServer\Providers\WDSTFTP').ReadFilter

# Add SMSBoot and SMSImages
$regPath = "HKLM:\SYSTEM\CurrentControlSet\Services\WDSServer\Providers\WDSTFTP"
$currentFilter = (Get-ItemProperty -Path $regPath).ReadFilter
$newFilter = $currentFilter + @("SMSBoot\*", "\SMSBoot\*", "SMSImages\*", "\SMSImages\*")
Set-ItemProperty -Path $regPath -Name "ReadFilter" -Value $newFilter -Type MultiString

# Restart WDS
Restart-Service WDSServer

# Verify
(Get-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Services\WDSServer\Providers\WDSTFTP').ReadFilter
```

**Expected ReadFilter entries (8 total):**
```
\boot\*
\tmp\*
boot\*
tmp\*
SMSBoot\*
\SMSBoot\*
SMSImages\*
\SMSImages\*
```

**Symptoms if ReadFilter is blocking:**
- Client downloads initial boot file (wdsmgfw.efi)
- Client tries to download BCD file → **Fails silently**
- Client falls back to default WDS boot image (not SCCM boot image)
- WinPE loads but immediately reboots (wrong drivers)

**Log evidence (smspxe.log):**
```
TFTP: 192.168.5.11: request for SMSBoot\HLL00002\x64\BCD.
[No "sending" line follows → File blocked by ReadFilter]
```

---

## Boot Image Configuration

### 1. Understanding Boot Images

**SCCM comes with 2 default boot images:**
- Boot image (x86) - For 32-bit BIOS clients
- Boot image (x64) - For 64-bit UEFI clients

**Boot Image Package IDs:**
- Format: [SiteCode]00001, [SiteCode]00002, etc.
- Example: HLL00001 (x86), HLL00002 (x64)

### 2. Creating a New Boot Image (Optional)

**When to create new boot image:**
- Need specific drivers for virtual/physical hardware
- Want custom WinPE components
- Testing without modifying default boot images

**Steps:**

1. SCCM Console → Software Library → Boot Images
2. Right-click → Create Boot Image using MDT
   - Or: Add Boot Image → Browse to ADK boot.wim
3. Source file: `C:\Program Files (x86)\Windows Kits\10\Assessment and Deployment Kit\Windows Preinstallation Environment\amd64\en-us\winpe.wim`
4. Name: Descriptive name (e.g., "Windows 11 x64 Boot Image - Proxmox")
5. Click Next through wizard

### 3. Adding Drivers to Boot Image

**CRITICAL: Boot images need drivers injected BEFORE they're distributed!**

#### For Physical Hardware:
- Usually no additional drivers needed
- Windows includes most network and storage drivers

#### For Proxmox/KVM Virtual Machines:
- **MUST add VirtIO drivers** for network and storage
- Without these, WinPE cannot access network or disk

**Steps to Add Drivers:**

1. **Import Drivers into SCCM:**

   - Software Library → Drivers
   - Right-click → Import Driver
   - Browse to VirtIO driver folder (extracted from virtio-win.iso or running VM)
   - Select all .inf files in:
     - NetKVM (network driver)
     - viostor (SCSI storage driver)
     - vioscsi (alternative SCSI driver)

2. **Add Drivers to Boot Image:**

   - Software Library → Boot Images
   - Right-click boot image → Properties
   - **Drivers** tab → Click **Add**
   - Select imported VirtIO drivers:
     - ✅ Red Hat VirtIO Ethernet Adapter (NetKVM)
     - ✅ Red Hat VirtIO SCSI controller (viostor)
     - ✅ Red Hat VirtIO SCSI pass-through (vioscsi)
   - Click OK

3. **Enable Command Support (F8 Debugging):**

   - **Customization** tab
   - ✅ Enable command support (F8)
   - This allows pressing F8 during WinPE boot for troubleshooting

4. **CRITICAL: Update Distribution Points**

   Adding drivers in SCCM Console does NOT automatically rebuild boot.wim!

   - Right-click boot image → **Update Distribution Points**
   - Select: ✅ **"Reload this boot image from its current source location"**
   - Click OK
   - **Wait 5-10 minutes for distribution to complete**

5. **Verify Boot Image Updated:**

   ```powershell
   # Check boot.wim timestamp
   Get-Item "C:\RemoteInstall\SMSImages\HLL00002\boot.HLL00002.wim" | Select-Object Name, Length, LastWriteTime
   ```

   **LastWriteTime should be RECENT (within last 10 minutes)**

   If timestamp is old:
   - Drivers weren't injected into boot.wim!
   - Update Distribution Points didn't complete
   - Check distmgr.log for errors

### 4. Extracting VirtIO Drivers from Working VM

If you have a working Windows 11 Proxmox VM with VirtIO drivers installed:

**On the working Windows 11 VM (as Administrator):**

```powershell
# Export all installed drivers
Export-WindowsDriver -Online -Destination "C:\Exported-VirtIO-Drivers"

# Find VirtIO drivers
Get-ChildItem "C:\Exported-VirtIO-Drivers" -Directory | Where-Object {
    $_.Name -like "*vio*" -or $_.Name -like "*netkvm*"
}
```

Copy the exported folders to SCCM server, then import into SCCM as described above.

**Script available on Desktop:** `extract-drivers-working.ps1`

---

## Task Sequence Deployment

### 1. Create or Verify Task Sequence

**SCCM Console:**

1. Software Library → Task Sequences
2. Verify you have a Windows deployment task sequence

**Key task sequence settings:**

- Boot image: Must specify which boot image to use
- Operating System Image: Windows 11 WIM/ESD file
- Apply Network Settings: Join domain or workgroup
- Apply Windows Settings: Computer name format

### 2. Deploy Task Sequence for PXE Boot

**CRITICAL: Task sequence must be deployed to "All Unknown Computers" collection!**

**Steps:**

1. Right-click task sequence → **Deploy**

2. **Collection:** Select **All Unknown Computers**
   - This is a built-in collection
   - Includes any machine that PXE boots but isn't in SCCM database

3. **Deployment Settings:**
   - Purpose: **Available** (not Required)
   - Make available to: ✅ **Only media and PXE**
     - This is CRITICAL - "Configuration Manager clients only" will NOT work for unknown machines!

4. **Scheduling:**
   - Available: As soon as possible

5. **User Experience:**
   - Show Task Sequence progress: Yes

6. Complete the wizard → Click OK

### 3. Verify Deployment

**PowerShell:**

```powershell
# Check task sequence deployments
Get-WmiObject -Namespace "root\SMS\site_HLL" -Class SMS_Advertisement |
  Where-Object {$_.CollectionID -eq "SMS00001"} |
  Select-Object AdvertisementName, PackageID, PresentTime
```

**Or check SCCM Console:**

- Monitoring → Deployments
- Filter by "All Unknown Computers"
- Verify task sequences are listed with "Available" and "Only media and PXE"

---

## Troubleshooting PXE Boot Issues

### Essential Log Files

**On SCCM Server:**

```
C:\Program Files\Microsoft Configuration Manager\Logs\
```

**Key logs:**

1. **smspxe.log** - PXE requests and responses
   - Shows client MAC address, architecture, boot image selected
   - TFTP file transfer status
   - Task sequence deployment lookups

2. **distmgr.log** - Distribution manager
   - Boot image distribution status
   - Package distribution to distribution points
   - WIM mounting and driver injection

3. **mpcontrol.log** - Management Point
   - MP installation and health
   - Client communication issues

4. **sitecomp.log** - Site component manager
   - Component installation and configuration
   - PXE provider status

**Monitoring smspxe.log in real-time:**

```powershell
Get-Content "C:\Program Files\Microsoft Configuration Manager\Logs\smspxe.log" -Wait -Tail 30
```

### Troubleshooting Workflow

#### Issue 1: Client Not Getting PXE Response

**Symptoms:**
- Client sits at "PXE-E51: No DHCP or proxyDHCP offers"
- Or: Client gets IP but no boot file

**Check:**

1. **DHCP Options:**
   ```powershell
   Get-DhcpServerv4OptionValue -ScopeId ip address
   ```
   - Verify Option 66 points to SCCM server IP
   - Verify Option 67 is NOT set (or set correctly for your architecture)

2. **PXE Service:**
   ```powershell
   Get-Service WDSServer
   ```
   - Must be "Running"

3. **Firewall:**
   ```powershell
   Get-NetFirewallRule | Where-Object {$_.DisplayName -like "*WDS*" -or $_.DisplayName -like "*TFTP*"}
   ```
   - WDS and TFTP rules must be enabled

4. **PXE Enabled on DP:**
   - SCCM Console → Distribution Points → Properties → PXE tab
   - Verify "Enable PXE support" is checked

**smspxe.log should show:**
```
PXE: [MAC]: Operation=1, MessageType=1, Architecture=7
PXE: [MAC]: Client is 64-bit, UEFI, Firmware
```

---

#### Issue 2: Boot File Downloads, Then Client Reboots

**Symptoms:**
- Client downloads boot.wim (large file transfer completes)
- Blue Windows logo appears briefly
- System immediately reboots

**This is a DRIVER issue - WinPE cannot access storage or network!**

**Check:**

1. **Boot Image Drivers:**

   - SCCM Console → Boot Images → Right-click → Properties → Drivers tab
   - Verify VirtIO drivers are listed (if using Proxmox)
   - Verify drivers match your hardware

2. **Boot Image Timestamp:**

   ```powershell
   Get-Item "C:\RemoteInstall\SMSImages\HLL00002\boot.HLL00002.wim" | Select LastWriteTime
   ```

   **If timestamp is OLD (before you added drivers):**
   - Drivers are in SCCM console but NOT in boot.wim file!
   - Boot image was never rebuilt after adding drivers!

   **Fix:**
   - Right-click boot image → Update Distribution Points
   - Select "Reload this boot image from its current source location"
   - Wait 10 minutes
   - Re-check timestamp - should be current

3. **Test on Physical Hardware:**

   - If Proxmox VM fails but physical laptop works → **Driver issue confirmed**
   - Physical hardware works with inbox drivers
   - Virtual hardware (Proxmox) needs VirtIO drivers

4. **Check distmgr.log:**

   ```powershell
   Get-Content "C:\Program Files\Microsoft Configuration Manager\Logs\distmgr.log" -Tail 50 |
     Select-String "HLL00002"
   ```

   Look for:
   - "Processing package HLL00002" → Distribution started
   - "WIMMountImage() failed" → Driver injection failed!
   - Error 0xc1420114 → WIM mount error (requires cleanup)

**Common WIM mount errors:**

```powershell
# Clean up stuck WIM mounts
dism /Cleanup-Wim

# Check mounted images
dism /Get-MountedWimInfo
```

---

#### Issue 3: TFTP "Cannot Open" Errors

**Symptoms in smspxe.log:**
```
TFTP: ip address: request for SMSBoot\HLL00002\x64\BCD.
TFTP: ip address: cannot open SMSBoot\HLL00002\x64\BCD.
```

**Cause: TFTP ReadFilter blocking SMSBoot directory**

**Fix:**

```powershell
# Add SMSBoot and SMSImages to ReadFilter
$regPath = "HKLM:\SYSTEM\CurrentControlSet\Services\WDSServer\Providers\WDSTFTP"
$currentFilter = (Get-ItemProperty -Path $regPath).ReadFilter
$newFilter = $currentFilter + @("SMSBoot\*", "\SMSBoot\*", "SMSImages\*", "\SMSImages\*")
Set-ItemProperty -Path $regPath -Name "ReadFilter" -Value $newFilter -Type MultiString
Restart-Service WDSServer
```

Verify in logs:
```
TFTP: ip address: request for SMSBoot\HLL00002\x64\BCD.
TFTP: ip address: sending SMSBoot\HLL00002\x64\BCD.  ← Should see this!
```

---

#### Issue 4: "No Task Sequence Available"

**Symptoms:**
- WinPE loads successfully
- Blue background, Windows logo
- Error: "No task sequence is available for this computer"

**Causes:**

1. **Task sequence not deployed to "All Unknown Computers"**

   Check deployment:
   - Monitoring → Deployments
   - Filter Collection = "All Unknown Computers"
   - Verify task sequence is listed

   **Fix:**
   - Right-click task sequence → Deploy
   - Collection: **All Unknown Computers**
   - Available to: **Only media and PXE** ← CRITICAL!

2. **Unknown Computer Support not enabled**

   ```powershell
   $siteCode = "HLL"
   $dp = Get-WmiObject -Namespace "root\SMS\site_$siteCode" -Class SMS_SCI_SysResUse -Filter "RoleName='SMS Distribution Point'"
   $dp.Props | Where-Object {$_.PropertyName -eq "SupportUnknownMachines"}
   ```

   Value should be 1 (enabled)

   **Fix:**
   - Distribution Point Properties → PXE tab
   - ✅ Enable unknown computer support

3. **Boot image mismatch**

   Task sequence specifies HLL00002, but client is using HLL00003 (wrong architecture)

   **Check smspxe.log:**
   ```
   PXE: [MAC]:   HLL20009, HLL00002, 64-bit, optional, is valid.
   ```

   Boot image ID should match task sequence boot image

---

#### Issue 5: Management Point Not Accessible

**Symptoms in smspxe.log:**
```
PXE: [MAC]: Prioritizing local Management Point http://SERVER.domain.local.
PXE: [MAC]: Failed to connect to Management Point
```

**Check:**

1. **Management Point Role Installed:**

   - Administration → Site Configuration → Servers and Site System Roles
   - Verify "Management Point" role is listed

2. **MP Service Running:**

   ```powershell
   Get-Service SMS_EXECUTIVE
   ```

3. **HTTP/HTTPS Access:**

   ```powershell
   Invoke-WebRequest -Uri "http://VM3-APP1.homelab.local/sms_mp/.sms_aut?mplist" -UseBasicParsing
   ```

   Should return StatusCode: 200

4. **IIS Running:**

   ```powershell
   Get-Service W3SVC
   ```

**Fix if MP is broken:**

```powershell
# Create trigger file to reinstall MP
New-Item "C:\Program Files\Microsoft Configuration Manager\inboxes\sitecomp.box\sitecomp.ct0" -ItemType File -Force

# Wait 2 minutes, then check sitecomp.log
Get-Content "C:\Program Files\Microsoft Configuration Manager\Logs\sitecomp.log" -Tail 20
```

---

## Proxmox-Specific Configuration

### 1. Proxmox VM Settings for PXE Boot

**Recommended VM configuration:**

```bash
# On Proxmox host
qm set 999 -memory 4096      # 4GB RAM minimum for WinPE
qm set 999 -cores 2           # 2 CPU cores
qm set 999 -bios ovmf         # UEFI firmware
qm set 999 -boot order='net0' # Network boot first
```

**Disk Controller Options:**

1. **VirtIO SCSI (Recommended for Production):**
   - Best performance
   - **Requires VirtIO drivers in boot image!**
   - Controller: `virtio-scsi-pci`

2. **SATA/AHCI (Good Compatibility):**
   - Slower than VirtIO
   - Works with Windows inbox drivers (no custom drivers needed)
   - Controller: `ahci`

3. **IDE (Legacy, Slowest):**
   - Maximum compatibility
   - No drivers needed
   - Very slow performance
   - Use only for testing

**Change disk controller:**

Proxmox Web GUI:
1. VM → Hardware
2. Select Hard Disk → Detach
3. Add → Hard Disk
4. Bus/Device: Choose SATA or VirtIO SCSI
5. Select existing disk image

### 2. Network Configuration

**Proxmox VM Network:**

- Model: **VirtIO (paravirtualized)** - Requires NetKVM driver in boot image
- Alternative: **Intel E1000** - Works without drivers (slower)

```bash
# Change network adapter to VirtIO
qm set 999 -net0 virtio,bridge=vmbr0

# Or use E1000 for compatibility (no drivers needed)
qm set 999 -net0 e1000,bridge=vmbr0
```

### 3. UEFI and Secure Boot

**Disable Secure Boot for PXE:**

Secure Boot prevents loading unsigned drivers (VirtIO drivers are unsigned)

```bash
# Disable Secure Boot (recommended for SCCM deployment)
qm set 999 -efidisk0 local-lvm:vm-999-disk-0,efitype=4m,pre-enroll-keys=0
```

### 4. VirtIO Driver Installation

#### Method 1: Extract from Working VM

If you have a working Windows 11 Proxmox VM:

**On Windows 11 VM (as Administrator):**

```powershell
# Run the provided script
.\extract-drivers-working.ps1

# Drivers exported to: C:\Exported-VirtIO-Drivers
```

Copy to SCCM server:
```
\\VM3-APP1\C$\Temp\VirtIO-Drivers-Extracted
```

#### Method 2: Download VirtIO ISO

1. Download: https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso
2. Mount ISO on SCCM server
3. Copy driver folders to local disk:
   - `NetKVM\w11\amd64\` → Network driver
   - `viostor\w11\amd64\` → Storage driver
   - `vioscsi\w11\amd64\` → Alternative storage driver

#### Method 3: Install on Test VM, Then Extract

1. Create new Proxmox VM with Windows 11
2. During install, load VirtIO drivers when asked "Where do you want to install Windows?"
3. Complete Windows install
4. Run `extract-drivers-working.ps1` to export installed drivers

---

## Common Errors and Solutions

### Error 1: "TFTP cannot open file"

**Full Error in smspxe.log:**
```
TFTP: 192.168.5.11: request for SMSBoot\HLL00002\x64\BCD.
TFTP: 192.168.5.11: cannot open SMSBoot\HLL00002\x64\BCD.
```

**Cause:** TFTP ReadFilter blocking SMSBoot directory

**Solution:** Add SMSBoot and SMSImages to ReadFilter (see [TFTP ReadFilter section](#3-configure-tftp-readfilter))

**Impact:** Client cannot download boot configuration, falls back to default WDS image, WinPE reboots immediately

---

### Error 2: Boot Image Missing Tabs in SCCM Console

**Symptom:** When opening Boot Image Properties, "Customization" and "Optional Components" tabs are missing

**Cause:** ADK version mismatch - SCCM cannot find installed ADK

**Solution:**

1. Verify correct ADK version installed (must match Windows version)
2. Right-click boot image → Update Distribution Points
3. Select: ✅ "Reload this boot image from the current Windows ADK"
4. Wait 5 minutes
5. Refresh SCCM console
6. Re-open boot image properties → Tabs should appear

**Prevention:** Always install matching ADK version before creating boot images

---

### Error 3: "WIMMountImage() failed" Error 0xc1420114

**Full Error in distmgr.log:**
```
Failed to mount boot image. WIMMountImage() failed. Error = 0xc1420114
```

**Cause:**
- Previous WIM mount operation didn't clean up properly
- Corrupted WIM mount registry
- Boot.wim file is locked by another process

**Solution:**

```powershell
# 1. Check for mounted images
dism /Get-MountedWimInfo

# 2. Clean up all WIM mounts
dism /Cleanup-Wim

# 3. Restart SMS_EXECUTIVE service
Restart-Service SMS_EXECUTIVE

# 4. Try updating boot image again
```

**Prevention:** Don't interrupt distribution manager operations, wait for them to complete

---

### Error 4: Boot Image Timestamp Never Updates

**Symptom:**
```powershell
Get-Item "C:\RemoteInstall\SMSImages\HLL00002\boot.HLL00002.wim" | Select LastWriteTime
# Shows old date (before you added drivers)
```

**Cause:** Distribution manager not rebuilding boot.wim after driver changes

**Solution:**

1. **Force reload:**
   - Right-click boot image → Update Distribution Points
   - Select: ✅ "Reload this boot image from its current source location"

2. **Monitor distribution:**
   ```powershell
   Get-Content "C:\Program Files\Microsoft Configuration Manager\Logs\distmgr.log" -Wait -Tail 30
   ```
   Look for: "Processing package HLL00002"

3. **Verify completion:**
   - Wait 10 minutes
   - Re-check boot.wim timestamp
   - Should be current

**If still not updating:**
- Check distmgr.log for errors
- Check DISM errors: `dism /Cleanup-Wim`
- Restart SMS_EXECUTIVE service
- Try again

---

### Error 5: "No Task Sequence Available for This Computer"

**Full Error:** WinPE loads, shows error message "No task sequence is available for this computer"

**Causes and Solutions:**

**1. Task Sequence Not Deployed to Unknown Computers:**

Check:
```powershell
# Verify deployment
Get-WmiObject -Namespace "root\SMS\site_HLL" -Class SMS_Advertisement |
  Where-Object {$_.CollectionID -eq "SMS00001"}
```

Fix:
- Deploy task sequence to **All Unknown Computers** collection
- Set availability: **Only media and PXE**

**2. Unknown Computer Support Not Enabled:**

Check:
- Distribution Point Properties → PXE tab
- Verify: ✅ Enable unknown computer support

**3. Boot Image Architecture Mismatch:**

Check smspxe.log:
```
PXE: [MAC]:   HLL20009, HLL00002, 64-bit, optional, is valid.
```

Task sequence boot image (HLL00002) must match what PXE server is providing.

If mismatch:
- Task Sequence Properties → Advanced tab
- Change Boot Image to correct architecture (x64 for UEFI, x86 for BIOS)

---

### Error 6: Immediate Reboot After boot.wim Download (Proxmox)

**Symptom:**
- Client downloads boot.wim successfully (400+ MB)
- Blue Windows logo appears briefly
- System reboots immediately
- Works on physical laptop, fails on Proxmox VM

**Cause:** Missing VirtIO drivers in boot image - WinPE cannot access storage

**Solution:**

1. **Extract drivers from working Windows 11 VM:**
   ```powershell
   .\extract-drivers-working.ps1
   ```

2. **Import drivers into SCCM:**
   - Software Library → Drivers → Import Driver
   - Browse to extracted VirtIO folders
   - Select NetKVM, viostor, vioscsi

3. **Add drivers to boot image:**
   - Boot Images → Properties → Drivers tab
   - Add VirtIO drivers

4. **FORCE UPDATE distribution point:**
   - Update Distribution Points → Reload from source
   - Wait 10 minutes
   - Verify boot.wim timestamp changed

5. **Test PXE boot again**

**Alternative:** Change Proxmox VM disk controller from VirtIO SCSI to SATA (slower but works without drivers)

---

### Error 7: Architecture Detection Mismatch

**Symptom in smspxe.log:**
```
PXE: [MAC]: Operation=1, MessageType=1, Architecture=16
PXE: [MAC]: Client is 32-bit, BIOS, Firmware.  ← WRONG!
```

**Explanation:**
- Architecture 16 = UEFI x64 HTTP Boot
- Log incorrectly shows "32-bit, BIOS"
- This is a SCCM log parsing bug, NOT an actual error

**Impact:** None - ignore this cosmetic issue

**Real issue:** Check which boot file was served:
```
BootFile: smsboot\HLL00002\x64\wdsmgfw.efi  ← Correct for UEFI x64
```

As long as correct boot file is served, architecture detection warning can be ignored.

---

## Quick Reference: Boot Process Flow

### Successful PXE Boot Sequence

1. **Client broadcasts DHCP Discover**
   - Includes PXE vendor options
   - Architecture type in DHCP request

2. **DHCP server responds with:**
   - IP address assignment
   - Option 66: PXE server IP
   - (Option 67: Boot file - if set)

3. **Client contacts PXE server**
   - SCCM PXE provider receives request
   - Checks client architecture (BIOS/UEFI, x86/x64)
   - Selects appropriate boot file

4. **TFTP transfer begins:**
   - Download wdsmgfw.efi (UEFI boot manager)
   - Download bootmgfw.efi (Windows boot manager)
   - Request BCD (Boot Configuration Data)
   - Download boot.sdi (RAM disk image)
   - Download boot.wim (400+ MB WinPE image)

5. **WinPE boots:**
   - Load boot.wim into RAM
   - Initialize hardware drivers
   - Detect network adapters (needs network driver!)
   - Detect storage controllers (needs storage driver!)
   - Start WinPE shell

6. **Task Sequence wizard loads:**
   - Connect to Management Point
   - Query available task sequences
   - Filter by deployment type (PXE)
   - Filter by collection (All Unknown Computers)
   - Display available task sequences

7. **User selects task sequence:**
   - Download and execute steps
   - Format disk, apply OS image, install drivers, join domain, etc.

### Log Evidence of Each Step

**smspxe.log:**

```
[Step 1-2] Client DHCP request
PXE: macc address: Operation=1, MessageType=1, Architecture=7

[Step 3] PXE server lookup
PXE: macc address: Client machine is UNKNOWN.
PXE: macc address: Prioritizing local Management Point

[Step 4] Boot file selection
PXE: macc address:   HLL20009, HLL00002, 64-bit, optional, is valid.
BootFile: smsboot\HLL00002\x64\wdsmgfw.efi

[Step 5] TFTP transfers
TFTP: ip-address: request for smsboot\HLL00002\x64\wdsmgfw.efi.
TFTP: ip-address: sending smsboot\HLL00002\x64\wdsmgfw.efi
TFTP: ip-address: request for HLL00002.WIM.
TFTP: ip-address: sending HLL00002.WIM
TFTP: ip-address: end of file. 408129318  ← boot.wim fully transferred
```

**If any step fails, boot process stops at that point!**

---

## Checklist: Fresh SCCM PXE Setup

Use this checklist when setting up SCCM PXE from scratch:

### Prerequisites
- [ ] Install Windows Server 2019/2022 (domain-joined)
- [ ] Install SQL Server (Express or full)
- [ ] Install .NET Framework 3.5 and 4.8
- [ ] Install IIS with required features
- [ ] Install BITS

### ADK Installation
- [ ] Download Windows ADK matching your Windows version
- [ ] Download WinPE Add-on for ADK
- [ ] Install ADK (Deployment Tools + WinPE)
- [ ] Install WinPE Add-on
- [ ] Restart server

### SCCM Installation
- [ ] Install SCCM Current Branch
- [ ] Configure site code (3 characters)
- [ ] Configure SQL connection
- [ ] Complete SCCM setup wizard
- [ ] Verify console opens without errors

### Verify ADK Recognition
- [ ] Open SCCM Console → Boot Images
- [ ] Right-click Boot image (x64) → Properties
- [ ] Verify "Customization" and "Optional Components" tabs exist
- [ ] If missing: Update boot image → Reload from current ADK

### Distribution Point Setup
- [ ] Administration → Distribution Points
- [ ] Right-click server → Properties → PXE tab
- [ ] Enable PXE support for clients
- [ ] Enable unknown computer support
- [ ] Click OK (WDS will auto-install)
- [ ] Verify WDS service running

### TFTP ReadFilter Configuration
- [ ] Run PowerShell as Administrator
- [ ] Add SMSBoot and SMSImages to ReadFilter
- [ ] Restart WDS service
- [ ] Verify 8 entries in ReadFilter

### DHCP Configuration
- [ ] Set Option 66 to SCCM server IP
- [ ] Remove Option 67 (let SCCM handle it)
- [ ] Verify with Get-DhcpServerv4OptionValue

### Boot Image Configuration
- [ ] Decide: Use default boot images or create new?
- [ ] If Proxmox: Add VirtIO drivers to boot image
- [ ] Enable F8 command support
- [ ] Update Distribution Points → Reload from source
- [ ] Wait 10 minutes
- [ ] Verify boot.wim timestamp updated

### Task Sequence Deployment
- [ ] Create or import task sequence
- [ ] Deploy to "All Unknown Computers" collection
- [ ] Set availability: "Only media and PXE"
- [ ] Set purpose: Available
- [ ] Verify deployment in Monitoring

### Test PXE Boot
- [ ] Boot test client via PXE
- [ ] Verify DHCP offer received
- [ ] Verify boot files download
- [ ] Verify WinPE loads (blue Windows logo)
- [ ] Verify task sequence wizard appears
- [ ] If fails: Check logs and troubleshoot

---

## Scripts Reference

All scripts created during this session are available on the Desktop:

### Diagnostic Scripts

**check-dhcp-options.ps1**
- Check DHCP Option 66 and 67 configuration

**check-pxe-unknown-support.ps1**
- Verify Unknown Computer Support is enabled in WMI

**diagnose-winpe-reboot.md**
- Comprehensive troubleshooting guide for WinPE reboot issues

### Configuration Scripts

**fix-dhcp-for-uefi-httpboot.md**
- Guide for configuring DHCP for UEFI HTTP Boot

**fix-tftp-readfilter.ps1**
- Add SMSBoot and SMSImages to TFTP ReadFilter

### Boot Image Management

**copy-boot-files-to-remoteinstall.ps1**
- Manually copy boot files to RemoteInstall directory

**create-bcd-for-hll0000f.ps1**
- Create BCD file for boot image (if missing)

**force-rebuild-boot-image-with-drivers.ps1**
- Import VirtIO drivers and force boot image rebuild
- Monitor distribution process
- Verify boot.wim timestamp changes

### Driver Extraction

**extract-drivers-working.ps1**
- Extract VirtIO drivers from running Windows 11 Proxmox VM
- Run on working VM, copy drivers to SCCM server

---

## Additional Resources

### Microsoft Documentation
- [SCCM PXE Boot Configuration](https://learn.microsoft.com/en-us/mem/configmgr/osd/deploy-use/use-pxe-to-deploy-windows-over-the-network)
- [Windows ADK Support](https://learn.microsoft.com/en-us/mem/configmgr/core/plan-design/configs/support-for-windows-adk)
- [Boot Image Management](https://learn.microsoft.com/en-us/mem/configmgr/osd/get-started/manage-boot-images)

### Video Tutorials
- [SCCM PXE Boot Setup](https://youtu.be/wCElfLBLyX0?si=vx9XmuajHViV0hk7)
- [SCCM Configuration](https://youtu.be/BPcy_nOQZoI?si=zXGUO319AdeGMFr2)

### VirtIO Drivers
- [Fedora VirtIO Downloads](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/)
- [Proxmox VirtIO Documentation](https://pve.proxmox.com/wiki/Windows_VirtIO_Drivers)

---

## Lessons Learned from This Deployment

### Critical Issues Encountered

1. **DHCP Option 67 pointing to BIOS boot file**
   - Client was UEFI x64, Option 67 pointed to `wdsnbp.com` (BIOS)
   - Changed to `wdsmgfw.efi` for UEFI
   - Better: Remove Option 67 and let SCCM PXE handle it

2. **TFTP ReadFilter blocking SMSBoot**
   - Default WDS only allows `\boot\*` and `\tmp\*`
   - SCCM uses `SMSBoot\` and `SMSImages\` (blocked!)
   - Client fell back to wrong boot image, WinPE rebooted
   - **Fix:** Add SMSBoot and SMSImages to ReadFilter

3. **Boot image timestamp never updating**
   - Added VirtIO drivers in SCCM console
   - Clicked "Update Distribution Points" multiple times
   - **Boot.wim file never rebuilt!**
   - LastWriteTime remained old (before driver addition)
   - **Solution:** "Reload this boot image from its current source location"
   - This forces SCCM to rebuild boot.wim with injected drivers

4. **VirtIO drivers needed for Proxmox**
   - Boot image worked on laptop (physical hardware)
   - Failed on Proxmox VM (immediate reboot after boot.wim load)
   - WinPE couldn't access VirtIO SCSI storage
   - **Solution:** Extract drivers from working Windows 11 VM, inject into boot image

5. **Task sequences using wrong boot image**
   - Changed task sequence to use HLL00002
   - Policy didn't propagate immediately
   - Logs still showed HLL0000F being used
   - **Solution:** Wait 5-10 minutes for policy replication

### Best Practices Established

1. **Always verify boot.wim timestamp after changes**
   - Don't assume "Update Distribution Points" worked
   - Check file LastWriteTime
   - If old, force reload from source

2. **Test on physical hardware first**
   - Helps isolate driver vs. configuration issues
   - Physical hardware works with inbox drivers
   - Virtual hardware (Proxmox) needs special drivers

3. **Use VirtIO drivers from working VM**
   - More reliable than downloading ISO
   - Drivers are proven to work on your Proxmox setup
   - Export with `Export-WindowsDriver -Online`

4. **Monitor logs in real-time during testing**
   - `Get-Content smspxe.log -Wait -Tail 30`
   - See exactly what's happening
   - Identify issues immediately

5. **Enable F8 command support in boot images**
   - Critical for troubleshooting
   - Press F8 during WinPE load
   - Get command prompt
   - Test: `ipconfig`, `diskpart list disk`, `wpeinit`

---

## Final Notes

This guide documents a complete SCCM PXE boot setup with specific focus on:
- UEFI x64 clients
- Proxmox virtual machines
- Unknown computer deployment
- VirtIO driver integration

The troubleshooting section covers all issues encountered during this deployment, with detailed solutions and prevention strategies.

**Key takeaway:** Most PXE boot issues are NOT SCCM bugs - they're configuration mismatches:
- DHCP options vs. client architecture
- Boot image drivers vs. hardware type
- Distribution point updates vs. actual file timestamps
- TFTP filters vs. SCCM directory structure

When troubleshooting, always:
1. Check logs (smspxe.log is your friend!)
2. Verify file timestamps
3. Test on different hardware
4. Work systematically through the boot process

---

**Document Version:** 1.0
**Last Updated:** 14 November 2025
**Environment:** SCCM Current Branch, Windows Server 2022, Proxmox 8.x, Windows 11 25H2
