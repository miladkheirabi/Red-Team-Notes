---
layout: default
title: Windows Environment and File System
---

# Windows Environment and File System

This guide covers useful Windows environment, user-profile, drive, filesystem, and registry artifacts encountered during post-exploitation enumeration.

## 1. Environment Variables

List environment variables:

```cmd
set
```

PowerShell:

```powershell
Get-ChildItem Env:
```

Useful variables:

| Variable | Purpose |
|---|---|
| `%SystemRoot%` | Windows directory |
| `%USERPROFILE%` | Current user's profile |
| `%APPDATA%` | Roaming application data |
| `%LOCALAPPDATA%` | Local application data |
| `%TEMP%` / `%TMP%` | Temporary directory |
| `%ProgramData%` | Shared application data |
| `%ProgramFiles%` | Installed applications |
| `%PATH%` | Directories searched for executable commands |
| `%COMPUTERNAME%` | Hostname |
| `%USERDOMAIN%` | User's domain or computer context |

Inspect persistent user and machine variables:

```powershell
[Environment]::GetEnvironmentVariables('User')
[Environment]::GetEnvironmentVariables('Machine')
```

Check `PATH` when investigating command execution and possible path hijacking. A writable directory in `PATH` matters only if the target process actually searches it.

## 2. User Profiles

List profile directories:

```cmd
dir C:\Users
```

Get registered profiles and their SIDs:

```powershell
Get-CimInstance Win32_UserProfile |
    Select-Object LocalPath, SID, Loaded
```

Common profile locations:

| Path | Contents |
|---|---|
| `Desktop` | Desktop files |
| `Documents` | User documents |
| `Downloads` | Downloaded files |
| `AppData\Roaming` | Roaming application data |
| `AppData\Local` | Local application data and caches |
| `AppData\Local\Temp` | Temporary files |

Inspect the current profile:

```cmd
echo %USERPROFILE%
dir /a "%USERPROFILE%"
```

## 3. Logical Drives and Volumes

List available drive letters:

```cmd
fsutil fsinfo drives
```

Get logical disk details:

```cmd
wmic logicaldisk get caption,description,drivetype,filesystem,freespace,size,volumename
```

PowerShell alternative:

```powershell
Get-CimInstance Win32_LogicalDisk |
    Select-Object DeviceID, Description, DriveType, FileSystem,
                  FreeSpace, Size, VolumeName
```

Common `DriveType` values:

| Value | Type |
|---:|---|
| `2` | Removable |
| `3` | Local disk |
| `4` | Network drive |
| `5` | CD-ROM |

WMIC is deprecated and may be unavailable on newer Windows versions. Use CIM when necessary.

List volumes and mount points:

```powershell
Get-Volume
```

```cmd
mountvol
```

## 4. Useful Windows Paths

| Path | Purpose |
|---|---|
| `C:\Windows\System32\` | Windows system binaries and libraries |
| `C:\Windows\System32\drivers\` | Driver files |
| `C:\Windows\System32\drivers\etc\hosts` | Local hostname mappings |
| `C:\Windows\System32\config\` | Registry hive files |
| `C:\Program Files\` | Installed applications |
| `C:\Program Files (x86)\` | 32-bit applications on 64-bit Windows |
| `C:\ProgramData\` | Shared application data |

Read the hosts file:

```cmd
type C:\Windows\System32\drivers\etc\hosts
```

Inspect application-directory permissions:

```cmd
icacls "C:\ProgramData"
icacls "C:\Program Files"
```

For privilege-escalation research, check whether a privileged process loads files from directories writable by the current user.

### Registry Hive Files

The registry's on-disk hive files are commonly stored in:

```text
C:\Windows\System32\config\
```

Important hive names include:

| Hive file | General scope |
|---|---|
| `SAM` | Local account database |
| `SYSTEM` | System configuration, including service and control-set data |
| `SECURITY` | Local security policy and security-related data |
| `SOFTWARE` | Machine-wide software and Windows configuration |
| `DEFAULT` | Default user profile settings used by the system |

These files are protected operating-system resources. Their presence and paths are useful context, but ordinary users generally cannot read them directly. Do not assume that knowing a path grants access.

### Startup Folders

Windows supports per-user and all-users Startup folders.

Current user's Startup folder:

```text
%APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup
```

Common all-users Startup folder:

```text
C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp
```

Inspect the current user's folder:

```cmd
dir "%APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup"
```

Inspect the common Startup folder:

```cmd
dir "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp"
```

These locations are relevant when enumerating programs configured to run at user sign-in. A shortcut or file's presence does not by itself prove that it executes successfully; inspect its target, file type, permissions, and execution context.

## 5. File and Permission Enumeration

List files, including hidden and system items:

```cmd
dir /a
```

Recursively list a directory:

```cmd
dir /a /s "C:\Path\To\Directory"
```

PowerShell:

```powershell
Get-ChildItem -Force "C:\Path\To\Directory"
```

Inspect file or directory permissions:

```cmd
icacls "C:\Path\To\File"
icacls "C:\Path\To\Directory"
```

Search for selected file types:

```cmd
dir /a /s C:\Users\*.kdbx
dir /a /s C:\Users\*.config
dir /a /s C:\Users\*.ini
```

## 6. MountPoints2

`MountPoints2` stores per-user artifacts related to previously accessed volumes and locations. It can provide clues about network shares and removable devices accessed by a user.

Query the current user's key:

```cmd
reg query "HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\MountPoints2" /s
```

`HKCU` refers to the account running the command. To query another user's hive, use its SID if the hive is loaded:

```cmd
reg query "HKEY_USERS\<SID>\Software\Microsoft\Windows\CurrentVersion\Explorer\MountPoints2" /s
```

These entries are historical artifacts, not a reliable list of currently connected or accessible resources.

## 7. Quick Reference

| Task | Command |
|---|---|
| List environment variables | `set` |
| List environment variables in PowerShell | `Get-ChildItem Env:` |
| Show current user profile | `echo %USERPROFILE%` |
| List user profiles and SIDs | `Get-CimInstance Win32_UserProfile \| Select-Object LocalPath, SID, Loaded` |
| List logical drives | `fsutil fsinfo drives` |
| Get logical disk details | `Get-CimInstance Win32_LogicalDisk` |
| List volumes and mount points | `Get-Volume` / `mountvol` |
| List hidden and system files | `dir /a` |
| Inspect file or directory ACLs | `icacls "C:\Path\To\File"` |
| Query MountPoints2 | `reg query "HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\MountPoints2" /s` |
| List current user's Startup folder | `dir "%APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup"` |
| List all-users Startup folder | `dir "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp"` |

## 8. Enumeration Checklist

- Record relevant environment variables, especially `USERPROFILE`, `APPDATA`, `LOCALAPPDATA`, `TEMP`, `ProgramData`, and `PATH`.
- Enumerate user profiles and their SIDs.
- Identify available logical drives, volume types, filesystems, and mount points.
- Review relevant Windows and application directories.
- Inspect the hosts file when it is within scope.
- Identify the Registry hive files and their locations; do not assume they are readable.
- Enumerate per-user and all-users Startup folders.
- Review hidden files and relevant filesystem ACLs.
- Query `MountPoints2` for historical volume and location artifacts.
- Distinguish between an artifact's presence and confirmed accessibility or execution.
