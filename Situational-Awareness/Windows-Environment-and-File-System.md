---
layout: default
title: Windows Environment and File System
---

# Windows Environment and File System

Windows environment variables, user-profile locations, logical drives, filesystem metadata, and registry artifacts can reveal useful information about a compromised host. During post-exploitation, these details help map the current execution context, identify accessible data and application locations, discover mounted or previously accessed resources, and find paths that may support further privilege escalation or lateral movement.

This guide focuses on practical Windows enumeration from an offensive security perspective.

## 1. Environment Variables

Environment variables are name-value pairs made available to processes. They expose commonly used paths and configuration values without requiring applications to hard-code them.

Variables can be defined at process, user, and machine scopes. A process generally receives an environment block when it starts, so changes made elsewhere may not be reflected in an already-running process.

### 1.1 Display Environment Variables

Command Prompt:

```cmd
set
```

Display individual values:

```cmd
echo %PATH%
echo %USERPROFILE%
echo %APPDATA%
echo %TEMP%
```

PowerShell:

```powershell
Get-ChildItem Env:
```

Display specific values:

```powershell
$env:PATH
$env:USERPROFILE
$env:APPDATA
$env:TEMP
```

### 1.2 Common Environment Variables

| Variable | Typical purpose |
|---|---|
| `%SystemRoot%` | Windows installation directory, commonly `C:\Windows` |
| `%windir%` | Another reference to the Windows directory |
| `%USERPROFILE%` | Current user's profile directory |
| `%APPDATA%` | Current user's roaming application-data directory |
| `%LOCALAPPDATA%` | Current user's local application-data directory |
| `%TEMP%` / `%TMP%` | Temporary directory for the current user or process |
| `%ProgramData%` | Application data shared across users |
| `%ProgramFiles%` | Program Files directory for native applications |
| `%ProgramFiles(x86)%` | 32-bit Program Files directory on 64-bit Windows |
| `%PATH%` | Directories searched when resolving commands without an explicit path |
| `%COMPUTERNAME%` | Computer's host name |
| `%USERDOMAIN%` | Domain or computer context associated with the current user |

Paths can vary because of installation choices, localization, redirection, and system configuration. Use the values observed on the target rather than assuming every machine uses the same layout.

### 1.3 User-Level and Machine-Level Values

PowerShell can display the environment variables available to the current process:

```powershell
Get-ChildItem Env:
```

To inspect persistent environment-variable definitions at different scopes:

```powershell
[Environment]::GetEnvironmentVariables('User')
[Environment]::GetEnvironmentVariables('Machine')
```

The process scope represents the environment block available to the current process. User and machine scopes are persistent configuration scopes.

### 1.4 Offensive Relevance

Environment variables can help identify:

- The current user's profile and application-data locations.
- Temporary directories used by applications, installers, and scripts.
- The Windows installation directory and architecture-related paths.
- Shared application-data locations.
- Directories included in command lookup through `PATH`.

Pay particular attention to `PATH` when investigating command execution. A directory appearing in `PATH` is not automatically exploitable; determine whether the relevant process searches that directory, whether the directory is writable in the current context, and whether the application invokes the command in a way that permits path hijacking.

## 2. User Profiles and Application Data

A typical user profile is located under:

```text
C:\Users\<username>\
```

Common subdirectories include:

| Location | Typical content |
|---|---|
| `Desktop` | User desktop files |
| `Documents` | User documents |
| `Downloads` | Downloaded files |
| `AppData\Roaming` | Roaming application settings and data |
| `AppData\Local` | Local application data and caches |
| `AppData\Local\Temp` | Temporary files for the user |

Useful commands:

```cmd
echo %USERPROFILE%
echo %APPDATA%
echo %LOCALAPPDATA%
dir "%USERPROFILE%"
```

PowerShell:

```powershell
Get-ChildItem -Force $env:USERPROFILE
Get-ChildItem -Force $env:APPDATA
Get-ChildItem -Force $env:LOCALAPPDATA
```

`-Force` includes hidden items in the listing; it does not bypass filesystem permissions.

### 2.1 Enumerate Other User Profiles

List profile directories:

```cmd
dir C:\Users
```

PowerShell can enumerate registered local profiles:

```powershell
Get-CimInstance Win32_UserProfile |
    Select-Object LocalPath, SID, Loaded, Special
```

This can reveal profile paths and SIDs even when the current user cannot access every profile's contents. Check permissions before attempting to enumerate individual directories.

## 3. Logical Drives and Filesystem Information

Drive enumeration helps identify the host's storage layout, available volumes, removable media, and potentially interesting mounted locations.

### 3.1 List Available Drive Letters

```cmd
fsutil fsinfo drives
```

This displays drive letters recognized by the filesystem.

A WMIC-based alternative:

```cmd
wmic logicaldisk get caption,description,drivetype,filesystem,freespace,size,volumename
```

PowerShell:

```powershell
Get-CimInstance Win32_LogicalDisk |
    Select-Object DeviceID, Description, DriveType,
                  FileSystem, FreeSpace, Size, VolumeName
```

WMIC is deprecated on recent Windows versions and may not be installed. Prefer CIM/PowerShell when WMIC is unavailable.

### 3.2 Understand Drive Types

The `DriveType` property commonly uses these values:

| Value | Type |
|---:|---|
| `0` | Unknown |
| `1` | No root directory |
| `2` | Removable disk |
| `3` | Local disk |
| `4` | Network drive |
| `5` | CD-ROM |
| `6` | RAM disk |

Network drives and removable volumes may expose additional paths, user activity, or data sources relevant to the engagement.

### 3.3 Inspect Volumes and Mount Points

PowerShell:

```powershell
Get-Volume |
    Select-Object DriveLetter, FileSystemLabel, FileSystem,
                  DriveType, HealthStatus, Size, SizeRemaining
```

List volume mount points:

```cmd
mountvol
```

A volume may be mounted under a directory instead of having a drive letter. When a path looks like an ordinary directory but contains a mounted volume, account for that when exploring the filesystem.

## 4. Important Windows Directories and Files

The following locations are commonly useful when understanding a Windows installation and locating application data.

### 4.1 Windows System Directory

```text
C:\Windows\System32\
```

Contains many Windows system executables, libraries, and configuration components. On 64-bit Windows, filesystem redirection can affect how 32-bit processes access system directories.

### 4.2 Drivers and Hosts File

```text
C:\Windows\System32\drivers\
C:\Windows\System32\drivers\etc\hosts
```

The `hosts` file provides local hostname-to-address mappings that participate in name resolution.

```cmd
type C:\Windows\System32\drivers\etc\hosts
```

Host mappings may reveal internal naming conventions, development systems, or locally configured destinations. Validate discovered names through other available network and DNS evidence.

### 4.3 Registry Hive Files

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

These are protected operating-system resources. Their presence and paths are useful context, but ordinary users generally cannot read them directly. Do not assume that knowing a path grants access.

### 4.4 Program Files and Shared Application Data

```text
C:\Program Files\
C:\Program Files (x86)\
C:\ProgramData\
```

`Program Files` commonly contains installed application binaries. `Program Files (x86)` is used for 32-bit applications on 64-bit Windows. `ProgramData` stores application data shared across users.

Inspect permissions on relevant directories:

```cmd
icacls "C:\ProgramData"
icacls "C:\Program Files"
icacls "C:\Program Files (x86)"
```

For offensive assessment, identify application directories that are writable by the current user and determine whether privileged processes load executables, DLLs, plugins, scripts, or configuration files from those locations. A writable directory alone does not establish an exploitable condition; the privileged execution path must also be validated.

### 4.5 Startup Folders

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
dir /a "%APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup"
```

Inspect the common Startup folder:

```cmd
dir /a "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp"
```

Startup folders are relevant when assessing user-level persistence and writable execution locations. Review shortcut targets, file permissions, and the account context under which the entries execute. See the Windows Registry Autoruns guide for additional logon and startup mechanisms.

## 5. Inspect Files, Hidden Items, and Permissions

### 5.1 List Files and Directories

Basic directory listing:

```cmd
dir
```

Include hidden and system entries:

```cmd
dir /a
```

List a specific location recursively:

```cmd
dir /a /s "C:\Path\To\Directory"
```

PowerShell:

```powershell
Get-ChildItem -Force "C:\Path\To\Directory"
Get-ChildItem -Force -Recurse "C:\Path\To\Directory"
```

Recursive enumeration can produce substantial output and may encounter access-denied errors. Narrow the target when possible.

### 5.2 Inspect File and Directory ACLs

```cmd
icacls "C:\Path\To\File"
icacls "C:\Path\To\Directory"
```

PowerShell:

```powershell
Get-Acl "C:\Path\To\File" | Format-List
```

When investigating a potentially interesting path, inspect both the object and its parent directories. The ability to replace or create a file depends on the effective permissions along the relevant path, not merely on whether the file can be read.

### 5.3 Search for Potentially Interesting Files

Examples of targeted searches:

```cmd
dir /a /s C:\Users\*.kdbx
dir /a /s C:\Users\*.config
dir /a /s C:\Users\*.ini
```

PowerShell:

```powershell
Get-ChildItem C:\Users -Include *.kdbx,*.config,*.ini -File -Recurse -Force -ErrorAction SilentlyContinue
```

Search results should be triaged against the engagement scope. File names and extensions are only clues; they do not prove that a file contains credentials or other useful information.

## 6. Enumerate MountPoints2

Windows stores per-user artifacts about previously accessed volumes and locations under the `MountPoints2` registry key.

Query the current user's key:

```cmd
reg query "HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\MountPoints2"
```

To include subkeys and their values:

```cmd
reg query "HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\MountPoints2" /s
```

### 6.1 Offensive Relevance

During post-exploitation enumeration, `MountPoints2` can provide clues about resources the current user has previously accessed. Depending on the artifacts present, these may include:

- Previously accessed network shares.
- Removable storage devices.
- Volume identifiers and resource references associated with the user.

These artifacts can help build a picture of the user's activity and identify network locations or storage devices worth investigating further.

### 6.2 User Context Matters

`HKCU` refers to the registry hive of the account executing the command. If the command runs as a service account, SYSTEM, or another user, it queries that account's hive—not necessarily the interactive user's hive.

To inspect another user's artifacts, identify the relevant SID and determine whether that user's hive is currently loaded. A loaded user hive may be available under `HKEY_USERS\<SID>`. For example:

```cmd
reg query "HKEY_USERS\<SID>\Software\Microsoft\Windows\CurrentVersion\Explorer\MountPoints2"
```

Replace `<SID>` with the actual SID. If the hive is not loaded, the key will not be available at that path unless the hive is loaded through an authorized procedure.

### 6.3 Correlate with Current Connections

`MountPoints2` is not a complete inventory of currently mounted drives or active SMB connections. Correlate it with current-session information:

```cmd
net use
```

PowerShell:

```powershell
Get-SmbConnection
Get-PSDrive -PSProvider FileSystem
```

Use the distinction carefully:

- `MountPoints2` provides per-user historical artifacts and references.
- `net use` shows network connections and mappings in the current logon context.
- `Get-SmbConnection` reports SMB client connections visible to the current context.
- `Get-PSDrive` lists PowerShell drives, which may include filesystem drives and mapped locations.

A resource appearing in `MountPoints2` does not prove that it is currently reachable or that the current token can access it.

## 7. Practical Offensive Enumeration Workflow

A compact workflow for this topic:

1. Dump the current process environment and note `USERPROFILE`, `APPDATA`, `LOCALAPPDATA`, `TEMP`, `ProgramData`, and `PATH`.
2. Identify local profiles and determine which user context the session is running under.
3. Enumerate drive letters, drive types, volumes, and mount points.
4. Inspect common application, system, and user-data directories.
5. Review hidden files, Startup folders, and ACLs on potentially interesting paths.
6. Query `MountPoints2` in the relevant user context.
7. Correlate historical artifacts with current network mappings and SMB connections.
8. Record paths, permissions, user context, and any follow-up leads for privilege-escalation or lateral-movement analysis.

## 8. Quick Reference

| Objective | Command |
|---|---|
| List environment variables | `set` |
| List environment variables in PowerShell | `Get-ChildItem Env:` |
| Show current profile | `echo %USERPROFILE%` |
| List user profiles | `dir C:\Users` |
| List logical drives | `fsutil fsinfo drives` |
| Get logical disk details | `wmic logicaldisk get caption,description,drivetype,filesystem,freespace,size,volumename` |
| Get volume details | `Get-Volume` |
| List volume mount points | `mountvol` |
| Include hidden/system files | `dir /a` |
| Inspect ACLs | `icacls <path>` |
| Query MountPoints2 | `reg query "HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\MountPoints2" /s` |
| Show mapped network drives | `net use` |
| Show SMB client connections | `Get-SmbConnection` |
