---
layout: default
title: Windows System Enumeration
---

# Windows System Enumeration

Understanding the target's Windows version, architecture, installed updates, and available administrative utilities is an important part of post-compromise situational awareness. This information helps an operator understand the environment, identify relevant security controls, and determine whether known vulnerabilities may apply.

The commands in this guide are intended for authorized assessments and controlled lab environments.

## 1. Identify the Windows Version

Start by collecting the operating system name, version, build number, and architecture.

```cmd
ver
systeminfo
```

`systeminfo` provides a broader summary, including the OS name, version, build, system type, installation date, and installed hotfixes. On some systems, access to individual fields or output may be restricted.

PowerShell provides structured output:

```powershell
Get-ComputerInfo | Select-Object WindowsProductName, WindowsVersion, OsBuildNumber, OsArchitecture
```

A lighter-weight alternative is to query the operating-system object through CIM:

```powershell
Get-CimInstance Win32_OperatingSystem |
    Select-Object Caption, Version, BuildNumber, OSArchitecture
```

You can also inspect the version information stored in the registry:

```cmd
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion" /v ProductName
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion" /v DisplayVersion
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion" /v CurrentBuild
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion" /v UBR
```

`CurrentBuild` identifies the base build, while `UBR` is the update build revision. Do not rely on a single value when assessing patch state.

## 2. Check the System Architecture

Architecture affects binary compatibility and which software or vulnerability conditions may be relevant.

```cmd
echo %PROCESSOR_ARCHITECTURE%
wmic os get osarchitecture
```

`wmic` is deprecated on recent Windows releases and may not be installed. Prefer PowerShell where available:

```powershell
(Get-CimInstance Win32_OperatingSystem).OSArchitecture
```

Be aware that a 32-bit process running on 64-bit Windows may report its process architecture rather than the underlying OS architecture in some contexts. Use OS-level information when recording the host's architecture.

## 3. Review Installed Updates and Hotfixes

The installed update set can help establish whether a known issue has been addressed.

```cmd
systeminfo
```

PowerShell:

```powershell
Get-HotFix | Sort-Object InstalledOn -Descending
```

For a focused query:

```powershell
Get-HotFix -Id KB503xxxx
```

Replace the example identifier with the update you are checking. If PowerShell reports `GetHotFixNoEntriesFound`, the requested KB was not found in the `Win32_QuickFixEngineering` inventory exposed by `Get-HotFix`. This does not conclusively prove that the update is absent: `Get-HotFix` does not provide a complete inventory of every update mechanism or component servicing state. Correlate its output with the OS build, update history, and other available patch-management evidence.

## 4. Research the Build Number

After collecting the edition, version, build, architecture, and patch information:

1. Identify the exact Windows release and servicing branch.
2. Compare the build and update revision with Microsoft's release and update history.
3. Determine whether the relevant security update is installed.
4. Search vulnerability databases for issues affecting that release.
5. Validate all prerequisites and mitigations before considering a finding applicable.

Useful references include:

- [Microsoft Windows release health](https://learn.microsoft.com/en-us/windows/release-health/)
- [Microsoft Update Catalog](https://www.catalog.update.microsoft.com/)
- [Microsoft Security Response Center](https://msrc.microsoft.com/)
- [NVD](https://nvd.nist.gov/)
- [Exploit-DB](https://www.exploit-db.com/)

A matching build number alone does **not** prove that a host is vulnerable. Exploit applicability may depend on the edition, architecture, installed updates, enabled features, configuration, and mitigations. Exploit-DB entries should be treated as research leads, not as proof of exploitability.

## 5. Common Windows Administrative Utilities

Windows includes administrative tools exposed through different executable types:

- `.exe` — ordinary executable programs.
- `.cpl` — Control Panel applets, commonly opened through Control Panel or `control.exe`.
- `.msc` — Microsoft Management Console snap-ins, commonly opened with `mmc.exe`.

Examples:

| Utility | Purpose |
|---|---|
| `ncpa.cpl` | Opens Network Connections |
| `appwiz.cpl` | Opens Programs and Features |
| `sysdm.cpl` | Opens System Properties |
| `desk.cpl` | Opens Display settings on supported versions |
| `firewall.cpl` | Opens Windows Defender Firewall |
| `regedit.exe` | Registry Editor |
| `taskmgr.exe` | Task Manager |
| `services.msc` | Services management console |
| `compmgmt.msc` | Computer Management |
| `eventvwr.msc` | Event Viewer |
| `devmgmt.msc` | Device Manager |
| `lusrmgr.msc` | Local Users and Groups; unavailable in some Windows editions |

The exact interface may vary between Windows versions. Some Control Panel applets are legacy interfaces that coexist with newer Settings pages.

You can launch a utility from Command Prompt or the Run dialog:

```cmd
ncpa.cpl
services.msc
eventvwr.msc
```

To locate an executable through the current search path:

```cmd
where regedit
where taskmgr
```

`where` searches directories listed in the current process's `PATH`; it is not a complete filesystem search.

## 6. Enumerate Running Processes

A process listing can reveal which applications and management tools are currently running.

Command Prompt:

```cmd
tasklist
tasklist /v
```

PowerShell:

```powershell
Get-Process | Sort-Object ProcessName
```

To inspect process ownership where permissions allow:

```cmd
tasklist /v
```

Or use CIM:

```powershell
Get-CimInstance Win32_Process |
    Select-Object ProcessId, ParentProcessId, Name
```

To inspect a specific process:

```powershell
Get-Process -Name regedit -ErrorAction SilentlyContinue
```

The presence of a process such as `regedit.exe`, `services.msc`, or `ncpa.cpl` may indicate that a user is performing administrative or configuration-related work. It does **not** prove that the current account is an Administrator. The process may have been launched by another user, through elevation, or in a different session.

## 7. Check the Last Boot Time

When access to a host is interrupted and a callback later resumes, checking whether the machine rebooted can help distinguish a possible restart from a network or agent-communication issue.

A quick filesystem-based check is:

```cmd
dir /a C:\pagefile.sys
```

The listing includes the file's last-modified timestamp. On many systems, the page file's timestamp can provide a rough clue about the last system start, but it is **not a definitive boot-time record**. Pagefile configuration, file recreation, system behavior, and timestamp changes can affect the result. The file may also be absent or configured differently.

A more direct method is to query the operating system:

```powershell
(Get-CimInstance Win32_OperatingSystem).LastBootUpTime
```

For a readable result:

```powershell
(Get-CimInstance Win32_OperatingSystem).LastBootUpTime.ToString("yyyy-MM-dd HH:mm:ss")
```

Treat the pagefile timestamp as a quick heuristic and `LastBootUpTime` as the preferred system-reported value. Neither alone proves why a callback stopped. Correlate the time with endpoint logs, agent state, network telemetry, and other available evidence.

## 8. Verify the Current Security Context

Do not infer privileges from the applications a user has open. Check the actual security context.

```cmd
whoami
whoami /user
whoami /groups
whoami /priv
```

PowerShell can also show the current identity:

```powershell
[System.Security.Principal.WindowsIdentity]::GetCurrent().Name
```

Useful indicators include membership in the local Administrators group, the integrity level, and enabled privileges. Membership alone may not tell the whole story because User Account Control (UAC), filtered administrator tokens, service accounts, and process elevation can affect effective access.

## User and Account Enumeration

Identifying the current user and local account privileges helps determine the security context of the current session and identify accounts with elevated access.

### Current User

Display the username associated with the current process:

```cmd
echo %USERNAME%
```

This returns the username stored in the `USERNAME` environment variable. It identifies the account context of the current process, which may differ from the user currently logged into the interactive desktop.

### 9. Network Login Information

Retrieve information about network login accounts:

```cmd
wmic netlogin get Name,NumberOfLogons
```

- `Name`: Domain and username associated with the network login account.
- `NumberOfLogons`: Number of times the account has logged on, as reported by the provider.

This queries the `Win32_NetworkLogin` class. It does not provide a complete list of local Windows accounts.

> **Note:** WMIC is deprecated and may not be available on newer Windows installations.

### Local Administrators

Enumerate members of the local Administrators group:

```cmd
net localgroup "Administrators"
```

The output lists local and domain accounts or groups that are members of the local Administrators group.

Membership in this group indicates administrative privileges on the local machine, subject to applicable security restrictions and token configuration.

## 10. Practical Enumeration Checklist

Record the following during an authorized assessment:

- Windows product name and edition.
- Version, build number, and update revision.
- OS architecture.
- Installed hotfixes and relevant security updates.
- Running processes and their owners, where available.
- Last boot time, including the method used to obtain it.
- Current user, group memberships, integrity level, and privileges.
- Relevant administrative tools and management consoles in use.

Keep raw command output when permitted. Distinguish observed facts from assumptions, and do not label a host vulnerable solely because its build number resembles one mentioned in an exploit write-up.
