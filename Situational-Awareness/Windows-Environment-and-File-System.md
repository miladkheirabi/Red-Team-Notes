---
layout: default
title: Windows Environment and File System
---

# Windows Environment and File System

Environment variables and standard filesystem locations provide useful context during Windows enumeration. They can reveal user-profile paths, temporary directories, application data locations, and files that are relevant to system configuration.

The commands in this guide are intended for authorized assessments and controlled lab environments.

## 1. Environment Variables

An environment variable is a named value made available to processes. Windows uses environment variables to expose commonly needed paths and configuration values without requiring applications to hard-code them.

Variables can be defined at different scopes, including user-level and system-level. A process generally receives an environment block when it starts; changes made elsewhere may not be reflected in an already-running process.

### 1.1 Display Environment Variables

Command Prompt:

```cmd
set
```

Display one variable:

```cmd
echo %PATH%
echo %USERPROFILE%
```

PowerShell:

```powershell
Get-ChildItem Env:
```

Display a specific variable:

```powershell
$env:PATH
$env:USERPROFILE
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
| `%ProgramFiles%` | Program Files directory for native 64-bit applications on 64-bit Windows |
| `%ProgramFiles(x86)%` | 32-bit Program Files directory on 64-bit Windows |
| `%PATH%` | Directories searched when resolving commands without an explicit path |
| `%COMPUTERNAME%` | Computer's host name |
| `%USERDOMAIN%` | Domain or computer context associated with the current user |

Paths can vary because of installation choices, localization, redirection, and system configuration. Treat the values observed on the target as authoritative rather than assuming every machine uses the same layout.

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

The process scope represents the environment block available to the current process. User and machine scopes are persistent configuration scopes; changing them may require a new process or sign-in before applications see the updated values.

### 1.4 Why Environment Variables Matter

During enumeration, environment variables can help identify:

- The current user's profile and application-data locations.
- Temporary directories used by applications and scripts.
- The Windows installation directory.
- Shared application-data locations.
- Directories included in command lookup through `PATH`.

A directory appearing in `PATH` is not automatically writable or exploitable. Check its ACLs and the way the relevant application resolves commands before drawing conclusions.

## 2. Windows User Profiles and Application Data

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
```

`-Force` includes hidden items in the listing; it does not bypass filesystem permissions.

## 3. Enumerate Available Drives

Start with a quick list of drive letters visible to the system:

```cmd
fsutil fsinfo drives
```

Example output:

```text
Drives: C:\ D:\ E:\ Z:\
```

For additional information about logical disks, WMIC can query the `Win32_LogicalDisk` class:

```cmd
wmic logicaldisk get caption,description,drivetype,filesystem,freespace,size,volumename
```

`wmic` is deprecated and may not be available on newer Windows installations. The PowerShell/CIM equivalent is:

```powershell
Get-CimInstance Win32_LogicalDisk |
    Select-Object DeviceID, Description, DriveType,
                  FileSystem, FreeSpace, Size, VolumeName
```

The `DriveType` field commonly uses these values:

| Value | Meaning |
|---:|---|
| `0` | Unknown |
| `1` | No root directory |
| `2` | Removable disk |
| `3` | Local disk |
| `4` | Network drive |
| `5` | Compact disc |
| `6` | RAM disk |

`fsutil fsinfo drives` is useful for quickly identifying drive letters. The logical-disk query provides more detail, but neither command should be treated as a complete inventory of every storage device or volume configuration.

## 4. Important Windows Directories and Files

The following locations are commonly useful when understanding a Windows installation.

### 3.1 Windows System Directory

```text
C:\Windows\System32\
```

Contains many Windows system executables, libraries, and configuration components. On 64-bit Windows, filesystem redirection can affect how 32-bit processes access system directories.

### 3.2 Drivers and Hosts File

```text
C:\Windows\System32\drivers\
C:\Windows\System32\drivers\etc\hosts
```

The `hosts` file provides local hostname-to-address mappings that are checked as part of name resolution. Inspect it as configuration data; entries may be legitimate, stale, or suspicious depending on context.

```cmd
type C:\Windows\System32\drivers\etc\hosts
```

Access may be restricted by file permissions or endpoint controls.

### 3.3 Registry Hive Files

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

These files are protected operating-system resources. Their presence and paths can be documented during enumeration, but ordinary users generally cannot read them directly. Do not assume that knowing a path grants access. Access is governed by Windows protections, file permissions, and system state.

### 3.4 Program Files and Shared Application Data

```text
C:\Program Files\
C:\Program Files (x86)\
C:\ProgramData\
```

`Program Files` commonly contains installed application binaries. `Program Files (x86)` is used for 32-bit applications on 64-bit Windows. `ProgramData` stores application data shared across users.

Inspect permissions before treating a directory as writable:

```cmd
icacls "C:\ProgramData"
icacls "C:\Program Files"
```

### 3.5 Startup Folders

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

The presence of a shortcut or file does not by itself establish that it executes successfully. Consider the file type, target, permissions, user context, and startup behavior.

## 4. Inspect Files and Permissions

Basic directory listing:

```cmd
dir
dir /a
```

`/a` includes entries with attributes such as hidden and system.

Inspect a file or directory's access-control list:

```cmd
icacls "C:\Path\To\File"
```

PowerShell:

```powershell
Get-Acl "C:\Path\To\File" | Format-List
```

Access checks depend on the current security token, inherited permissions, explicit deny entries, ownership, and other Windows security mechanisms. A directory being visible does not mean it is readable or writable.

## 5. Practical Enumeration Checklist

During an authorized assessment, record:

- Current values of `USERPROFILE`, `APPDATA`, `LOCALAPPDATA`, `TEMP`, `ProgramData`, and `PATH`.
- User profile directories and relevant application-data locations.
- Windows system and application installation directories.
- Hosts-file contents, if access is authorized.
- Relevant Startup folder contents.
- ACLs on directories that appear security-relevant.
- Whether protected system files are merely present or actually accessible.

Avoid collecting or copying sensitive data unless it is explicitly within the assessment scope. Document access restrictions instead of attempting to bypass them.
