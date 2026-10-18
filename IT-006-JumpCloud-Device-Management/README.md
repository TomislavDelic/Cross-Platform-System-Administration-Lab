# IT-006 – JumpCloud Windows Device Management

## Overview

This lab demonstrates centralized Windows 11 device management using JumpCloud MDM and JumpCloud Agent.

The project covers MDM enrollment, Windows configuration policies, local user provisioning, and temporary administrator privilege management.

A significant part of the lab involved troubleshooting residual Microsoft Intune enrollment data that prevented successful JumpCloud MDM enrollment.

All testing was performed in a Hyper-V virtual machine.

## Lab Environment

| Component | Configuration |
|---|---|
| Hypervisor | Microsoft Hyper-V |
| Operating System | Windows 11 Enterprise LTSC |
| Virtual Machine | WIN11-LAB |
| MDM Platform | JumpCloud MDM (EU tenant) |
| Endpoint Agent | JumpCloud Agent 2.192.0 |
| Previous MDM | Microsoft Intune |
| Test Account | lukak |
| Recovery Account | Local Windows administrator |

## Objectives

- Enroll Windows 11 into JumpCloud MDM.
- Understand the difference between JumpCloud MDM and JumpCloud Agent.
- Deploy and verify Windows MDM policies.
- Provision a local Windows account through JumpCloud.
- Grant temporary local administrator privileges.
- Revoke administrator privileges and verify the result.
- Troubleshoot MDM enrollment and configuration issues.

---

## Implementation

### 1. Windows 11 VM Preparation

Created a Hyper-V checkpoint before modifying device management settings.

Configured a separate local administrator account for recovery and verified that it could successfully sign in and perform administrative actions.

Removed the previous Microsoft Entra ID connection and Microsoft Intune MDM enrollment.

Verified the device registration status:

```powershell
dsregcmd /status
```

Results:

```text
AzureAdJoined    : NO
EnterpriseJoined : NO
DomainJoined     : NO
WorkplaceJoined  : NO
```

The device was no longer joined to Microsoft Entra ID or an on-premises Active Directory domain.

### 2. JumpCloud Agent Installation

Installed JumpCloud Agent version 2.192.0.

The Windows device appeared in the JumpCloud Admin Portal:

- Device: WIN11-LAB
- Agent Status: Active
- MDM Status: NOT ENROLLED

This demonstrated that installing the JumpCloud Agent and enrolling a device into MDM are separate operations.

### 3. Windows MDM Enrollment

After resolving conflicts with the previous Microsoft MDM configuration, enabled manual enrollment through JumpCloud User Portal.

Navigation:

JumpCloud User Portal → Security → Device Enrollment → Windows

Selected Start MDM Enrollment and completed the Windows device setup.

Verified the connection in Windows:

Settings → Accounts → Access work or school

Windows displayed:

```text
Connected to JumpCloud MDM
```

Confirmed the final status in JumpCloud Admin Portal:

```text
Device: WIN11-LAB
Agent Status: Active
MDM Status: ENROLLED
```

### 4. MDM Policy – Config Refresh

Created a Windows MDM policy:

**Policy Name:** IT-006 - Windows MDM Config Refresh

**Configuration:**

- Policy Type: MDM
- Configuration Refresh: 30 minutes
- Assigned Device: WIN11-LAB

Verified the deployment in JumpCloud Policy Results.

Result:

```text
SUCCESS
Exit Status: 0
ConfigRefresh/Enabled: 200 - OK
ConfigRefresh/Cadence: 200 - OK
```

The result confirmed that Windows accepted the MDM configuration.

### 5. MDM Policy – Taskbar Customization

Created another Windows MDM policy:

**Policy Name:** IT-006 - Windows MDM Taskbar Test

**Configuration:**

- Policy Type: MDM
- Setting: Hide the Task View button
- Assigned Device: WIN11-LAB

**Verification:**

Before the policy was applied, the Task View button was visible on the Windows 11 Taskbar.

After deployment:

- JumpCloud reported SUCCESS.
- The Task View button disappeared from the Taskbar.

This provided direct evidence that JumpCloud MDM successfully modified a Windows configuration setting.

The policy assignment was subsequently removed.

After signing out and signing back into Windows, the Task View button was restored, confirming that the policy removal took effect after refreshing the user session.

### 6. Local User Provisioning

Bound the JumpCloud test user `lukak` to WIN11-LAB.

JumpCloud Agent created the corresponding local Windows account.

Initial configuration:

```text
User: lukak
User Type: Standard
Permissions: No Elevated Permissions
Password Sync: Yes
```

Verified the account using:

Computer Management → Local Users and Groups → Users

Confirmed that `lukak` was not a member of the local Administrators group.

### 7. Temporary Administrator Privileges

Simulated a scenario where a user temporarily requires administrator access.

In JumpCloud Admin Portal, changed the test user's permissions to:

```text
Administrator/Sudo Access
Duration: 1 Hour
```

Saved the device configuration.

Verified that `lukak` appeared in the local Windows Administrators group.

Signed into WIN11-LAB using the test account and launched Command Prompt with Run as administrator.

Windows displayed a UAC consent prompt without requiring another administrator's credentials.

Verified the user identity:

```cmd
whoami
```

Result:

```text
win11-lab\lukak
```

The elevated Command Prompt confirmed that temporary administrator privileges were functional.

### 8. Administrator Privilege Revocation

Changed the user's JumpCloud permissions back to:

```text
No Elevated Permissions
```

Saved the changes.

Signed out and signed back into Windows to refresh the user's security token.

Attempted to launch Command Prompt as administrator.

Windows requested credentials from a separate administrator account.

This confirmed that the test user no longer had usable local administrator privileges.

The privileges were manually revoked before the one-hour period expired. Automatic expiration was not tested.


---

### 9. macOS VM Preparation

Prepared a dedicated macOS virtual machine using UTM on an Apple Silicon MacBook Pro.

The purpose of this environment is to extend the existing Windows JumpCloud device management lab to macOS while keeping testing isolated from the host operating system.

**Environment:**

| Component | Configuration |
|---|---|
| Hypervisor | UTM (Apple Virtualization) |
| Operating System | macOS Tahoe 26.1 |
| Architecture | Apple Silicon (ARM64) |
| Virtual CPU | Apple M5 (Virtual) |
| Memory | 4 GB |
| Local Administrator | labadmin |
| Guest Integration | UTM Guest Tools |

**Implementation:**

1. Created a macOS virtual machine in UTM.
2. Downloaded the macOS Tahoe 26.1 IPSW installation image.
3. Installed macOS using Apple Virtualization.
4. Created a dedicated local administrator account.
5. Installed UTM Guest Tools.
6. Enabled clipboard sharing between the host and guest operating systems.

**Verification:**

- macOS Tahoe 26.1 booted successfully.
- The local administrator account was accessible.
- UTM Guest Tools installation completed successfully.
- Clipboard sharing between host and guest was tested successfully.

The macOS virtual machine is ready for JumpCloud device management testing.


---

## Verification and Results

| Test | Result |
|---|---|
| JumpCloud Agent installation | Passed |
| Windows MDM enrollment | Passed |
| MDM Config Refresh policy | Passed |
| MDM Taskbar Customization | Passed |
| Visible Task View restriction | Passed |
| Local user provisioning | Passed |
| Initial standard user permissions | Passed |
| Temporary administrator access | Passed |
| Elevated Command Prompt verification | Passed |
| Administrator privilege revocation | Passed |
| Standard user UAC verification | Passed |
| Automatic Task View restoration | Passed |
| Automatic administrator privilege expiry | Not tested |

---

## Issues Encountered

### Issue 1 – Previous Microsoft Intune Enrollment Blocked JumpCloud MDM

**Problem**

JumpCloud Agent was active, but Windows MDM enrollment did not complete.

JumpCloud reported:

```text
1 Device enrolled with another vendor
```

This happened even after the existing Microsoft Entra ID and Intune connections had been removed from Windows.

**Investigation**

Inspected Windows Event Viewer:

```text
Applications and Services Logs
  Microsoft
    Windows
      DeviceManagement-Enterprise-Diagnostics-Provider
```

Reviewed MDM-related events, including:

```text
Event ID: 404
MDM ConfigurationManager: Command failure status
Result: The system cannot find the file specified.
```

The event occurred during the previous Microsoft MDM configuration and was not attributed directly to JumpCloud.

Additional enrollment events included:

```text
Event ID: 4022
Failed to enroll MMP-C for dual enrollment mode.
Error: 0x8018000b
```

These events were investigated as diagnostic evidence, without assuming they identified the exact cause of the JumpCloud failure.

Inspected Task Scheduler:

```text
Task Scheduler Library
  Microsoft
    Windows
      EnterpriseMgmt
```

Found scheduled tasks associated with the previous MDM enrollment.

Inspected the Windows Registry:

```text
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Enrollments
```

The identified enrollment contained Microsoft management information:

```text
ProviderID: MS DM Server
AADResourceID: https://manage.microsoft.com/
```

The same enrollment identifier appeared in Registry records, scheduled tasks and MDM event logs.

**Resolution**

Created a new Hyper-V checkpoint and exported the relevant Registry key.

Used a reviewed and modified version of the official JumpCloud Windows MDM cleanup PowerShell script.

The modifications included:

- Targeting the identified previous Intune enrollment GUID.
- Disabling the broader ForcePrune invocation.
- Commenting out the identified certificate-deletion command.

Executed the modified script in an elevated PowerShell session.

The script reported:

```text
Targeted Cleanup Phase
No MDM enrollment keys found after cleanup.
Cleanup was successful!
```

Verified removal of the targeted Registry key:

```powershell
Test-Path "HKLM:\SOFTWARE\Microsoft\Enrollments\<ENROLLMENT-GUID>"
```

Result:

```text
False
```

Verified removal of the corresponding scheduled-task folder:

```powershell
Test-Path "C:\Windows\System32\Tasks\Microsoft\Windows\EnterpriseMgmt\<ENROLLMENT-GUID>"
```

Result:

```text
False
```

**Result**

JumpCloud no longer reported the device as enrolled with another vendor.

The device became eligible for a new MDM enrollment attempt.

The cleanup was performed only in the controlled lab VM with checkpoints and recovery access.

### Issue 2 – PowerShell Script Execution Blocked

**Problem**

Windows initially prevented execution of the MDM cleanup script:

```text
running scripts is disabled on this system
UnauthorizedAccess
```

**Investigation**

Checked PowerShell execution policies:

```powershell
Get-ExecutionPolicy -List
```

All scopes were reported as Undefined, resulting in a restrictive default execution policy.

The downloaded script was also unsigned and marked as originating from the internet.

**Resolution**

Temporarily enabled RemoteSigned for the current PowerShell process:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy RemoteSigned
```

Checked the script signature:

```powershell
Get-AuthenticodeSignature ".\remove_windowsMDM.ps1"
```

Result:

```text
NotSigned
```

After reviewing the script and its source, removed the downloaded-file block:

```powershell
Unblock-File -Path ".\remove_windowsMDM.ps1"
```

Executed the reviewed and modified script:

```powershell
.\remove_windowsMDM.ps1
```

**Result**

The cleanup script completed and reported success.

The execution-policy change was limited to the active PowerShell process rather than being applied system-wide.

### Issue 3 – Automatic Windows MDM Enrollment Did Not Complete

**Problem**

Even after the previous Intune enrollment data had been cleaned up, JumpCloud continued to display:

```text
MDM Status: NOT ENROLLED
```

The automatic Windows MDM enrollment option was enabled, but enrollment did not complete.

**Investigation**

Verified that:

- JumpCloud Agent was active.
- The device was no longer joined to Microsoft Entra ID.
- The previous Intune enrollment had been removed.
- The residual enrollment Registry key and scheduled tasks had been removed.
- JumpCloud no longer detected a competing MDM vendor.

**Resolution**

Enabled manual Windows MDM enrollment through the JumpCloud User Portal.

Completed the Windows enrollment wizard using the JumpCloud test account.

**Result**

Windows displayed:

```text
Connected to JumpCloud MDM
```

JumpCloud Admin Portal confirmed:

```text
MDM Status: ENROLLED
```

The exact reason automatic enrollment did not complete was not conclusively established.

---

## Key Takeaways

- JumpCloud Agent and Windows MDM provide different management capabilities.
- Installing an endpoint agent does not automatically confirm MDM enrollment.
- Residual MDM configuration can interfere with migration between management providers.
- Event Viewer, Task Scheduler, Registry and PowerShell are useful for Windows device-management troubleshooting.
- MDM policies should be verified both centrally and directly on the endpoint.
- JumpCloud Agent can provision and manage local Windows users.
- Temporary administrator privileges help support least-privilege access.
- Administrator privilege revocation must be verified after the user obtains an updated Windows security token.

---

## Conclusion

Successfully configured JumpCloud Windows MDM and Agent-based management on a Windows 11 Enterprise LTSC virtual machine.

The lab included troubleshooting a previous Microsoft Intune enrollment, removing residual MDM configuration, enrolling the device into JumpCloud MDM, deploying Windows configuration policies, provisioning a local user, and granting and revoking temporary local administrator privileges.

All major management actions were verified through JumpCloud Admin Portal and direct Windows endpoint testing.
