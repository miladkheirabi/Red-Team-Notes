---
layout: default
title: Windows Registry Autoruns
---

# Windows Registry Autoruns

Windows provides several mechanisms for launching applications automatically during sign-in or system startup. Registry-based autorun locations are particularly relevant when reviewing persistence configuration and understanding which programs are configured to start for a user or across the machine.

This guide focuses on read-only enumeration and defensive assessment. Any validation that changes startup configuration must be explicitly authorized.

## 1. Registry Structure

The Windows Registry is a hierarchical configuration database. It is organized into root keys, subkeys, and values.

Common root keys include:

| Root key | General purpose |
|---|---|
| `HKEY_LOCAL_MACHINE` (`HKLM`) | Machine-wide configuration |
| `HKEY_CURRENT_USER` (`HKCU`) | Configuration for the current user |
| `HKEY_USERS` (`HKU`) | Loaded user profiles |
| `HKEY_CLASSES_ROOT` (`HKCR`) | File associations and COM-related registration view |
| `HKEY_CURRENT_CONFIG` (`HKCC`) | Current hardware profile information |

The same configuration area may be exposed through different registry views or aliases. `HKCU` represents the current user's loaded profile; it is not a machine-wide location.

## 2. Run and RunOnce Keys

The most familiar registry autorun locations are the `Run` and `RunOnce` keys.

### 2.1 Current User

```text
HKCU\Software\Microsoft\Windows\CurrentVersion\Run
HKCU\Software\Microsoft\Windows\CurrentVersion\RunOnce
```

These locations apply to the current user. Entries are commonly processed when that user signs in.

### 2.2 Local Machine

```text
HKLM\Software\Microsoft\Windows\CurrentVersion\Run
HKLM\Software\Microsoft\Windows\CurrentVersion\RunOnce
```

These locations are machine-wide and are generally relevant to programs configured to run for users of the computer.

`RunOnce` entries are intended for one-time execution and are normally removed after they are processed successfully, subject to the entry's behavior and Windows processing rules.

## 3. Query Common Autorun Locations

Use `reg query` to inspect a key:

```cmd
reg query "HKCU\Software\Microsoft\Windows\CurrentVersion\Run"
reg query "HKCU\Software\Microsoft\Windows\CurrentVersion\RunOnce"
reg query "HKLM\Software\Microsoft\Windows\CurrentVersion\Run"
reg query "HKLM\Software\Microsoft\Windows\CurrentVersion\RunOnce"
```

To inspect the machine-wide key recursively:

```cmd
reg query "HKLM\Software\Microsoft\Windows\CurrentVersion" /s
```

A recursive query can produce a large amount of output. Prefer targeted queries when possible.

PowerShell:

```powershell
Get-ItemProperty 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Run'
Get-ItemProperty 'HKLM:\Software\Microsoft\Windows\CurrentVersion\Run'
```

A missing key or value is not necessarily an error; it may simply mean no entries are configured at that location.

## 4. Additional Registry Locations Worth Reviewing

The following locations can contain startup-related or logon-related configuration. Their behavior differs, and not every entry is a conventional `Run` autorun.

| Registry location | General purpose |
|---|---|
| `HKCU\Software\Microsoft\Windows\CurrentVersion\Run` | Current-user logon programs |
| `HKCU\Software\Microsoft\Windows\CurrentVersion\RunOnce` | Current-user one-time logon entries |
| `HKLM\Software\Microsoft\Windows\CurrentVersion\Run` | Machine-wide logon programs |
| `HKLM\Software\Microsoft\Windows\CurrentVersion\RunOnce` | Machine-wide one-time entries |
| `HKLM\Software\WOW6432Node\Microsoft\Windows\CurrentVersion\Run` | 32-bit registry view on 64-bit Windows |
| `HKLM\Software\WOW6432Node\Microsoft\Windows\CurrentVersion\RunOnce` | 32-bit one-time entries on 64-bit Windows |
| `HKCU\Software\Microsoft\Windows\CurrentVersion\Policies\Explorer\Run` | Policy-related current-user logon configuration |
| `HKLM\Software\Microsoft\Windows\CurrentVersion\Policies\Explorer\Run` | Policy-related machine-wide logon configuration |
| `HKLM\System\CurrentControlSet\Services` | Service configuration; services are a separate startup mechanism |

The 32-bit registry view is relevant on 64-bit Windows. Which view a process sees can depend on its architecture and the registry APIs it uses. Do not assume that querying one path always exposes every relevant entry.

The `Services` key is not a `Run` key. Services are managed by the Service Control Manager and have their own configuration and startup semantics. See the Windows Services Enumeration guide for details.

## 5. Enumerate Startup Folder Entries

Startup folders are filesystem-based rather than registry-based, but they are useful to review alongside registry autoruns.

Current user:

```cmd
dir "%APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup"
```

All users:

```cmd
dir "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp"
```

Review shortcuts and their targets carefully. A shortcut's presence does not prove that its target exists or will execute successfully.

## 6. Interpret Autorun Entries

For each entry, record:

- Registry path and value name.
- Configured command or executable path.
- Whether the location is user-specific or machine-wide.
- Whether the referenced file exists.
- File signature and publisher, where relevant.
- File and parent-directory permissions.
- Whether the entry is expected for the installed software.
- Whether the entry is actually processed in the relevant sign-in or startup context.

A registry value may contain an executable path, arguments, a script, or a command interpreted by another program. Parse the full command carefully rather than assuming the value is only a filename.

An unfamiliar entry is not automatically malicious. Compare it with software inventory, vendor documentation, file metadata, signatures, and the organization's baseline.

## 7. Permissions and Security Context

The ability to read a registry key does not imply the ability to modify it. Registry keys have access-control lists, and write access varies by location and system configuration.

Inspect a key's permissions through Registry Editor or PowerShell where appropriate. For example:

```powershell
Get-Acl 'Registry::HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Run'
```

Do not attempt to alter autorun values as part of routine enumeration. If a controlled validation is explicitly authorized, document the original state and follow the engagement's change-control and rollback procedures.

## 8. Practical Enumeration Checklist

- Review current-user `Run` and `RunOnce`.
- Review machine-wide `Run` and `RunOnce`.
- Consider the 32-bit registry view on 64-bit Windows.
- Review policy-related `Explorer\Run` locations.
- Review user and all-users Startup folders.
- Record full commands, paths, and value names.
- Validate file existence, publisher, signature, and ACLs.
- Distinguish normal software behavior from confirmed suspicious or unauthorized persistence.

Registry autoruns are only one part of Windows startup behavior. A complete assessment may also need to consider services, scheduled tasks, startup folders, logon scripts, drivers, and other mechanisms within the agreed scope.
