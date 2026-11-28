---
layout: default
title: Windows Process and Service Enumeration
---

# Windows Process and Service Enumeration

Process and service enumeration reveals running applications, their owners, executable paths, command-line arguments, and the services hosted by each process.

## 1. Process Enumeration

### 1.1 Map Processes to Services

```cmd
tasklist /svc
```

`tasklist` lists running processes. The `/svc` switch adds the services hosted by each process, where applicable. The output includes process names, PIDs, and associated service names.

### 1.2 Understand svchost.exe

`svchost.exe` (Service Host) hosts Windows services implemented as DLLs rather than standalone executables.

Multiple `svchost.exe` processes are normal. Windows separates services into different host processes for isolation, reliability, and security. Depending on the Windows version and configuration, one instance may host one service or a group of services.

Use `tasklist /svc` to map service names to the PID hosting them.

### 1.3 Query an Individual Service

For example, `wuauserv` is the Windows Update service:

```cmd
sc.exe query wuauserv
```

Inspect its configuration:

```cmd
sc.exe qc wuauserv
```

`query` displays the service state; `qc` displays configuration such as the binary path and service account.

In PowerShell, use `sc.exe` explicitly. `sc` may resolve to the built-in `Set-Content` alias.

### 1.4 Identify Processes Running Outside SYSTEM

```cmd
tasklist /FI "USERNAME ne NT AUTHORITY\SYSTEM" /FI "STATUS eq RUNNING" /V
```

`/V` displays verbose details, including username and window title where available.

This is a first pass for finding processes running in user contexts. It does not prove that every result was launched interactively by a human: processes may also run as `LOCAL SERVICE`, `NETWORK SERVICE`, domain accounts, or virtual service accounts.

Before terminating a process, identify its owner and purpose. Killing an application a user is actively working with may interrupt their work and reveal the operator's presence.

### 1.5 Identify Executable Paths

```cmd
wmic process get name,executablepath,processid
```

PowerShell alternative:

```powershell
Get-CimInstance Win32_Process |
    Select-Object Name, ExecutablePath, ProcessId
```

The executable path helps identify where a process was launched from. Access to some paths may require elevated privileges. WMIC is deprecated and may be unavailable on newer Windows versions.

### 1.6 Search Processes by Name

```cmd
tasklist /v | findstr /i "<String>"
```

`findstr` returns lines containing the specified text. `/i` makes the search case-insensitive. A match can occur in any displayed column, not only the process name.

### 1.7 Inspect Process Command Lines

```cmd
wmic process get processid,commandline
```

PowerShell alternative:

```powershell
Get-CimInstance Win32_Process |
    Select-Object ProcessId, CommandLine
```

Command lines reveal executable arguments, scripts, and paths that are not visible in a basic process listing.

An encoded PowerShell command deserves closer inspection, particularly when combined with unusual parent processes, paths, or execution context. Encoding alone does not prove malicious activity; legitimate administration tools also use encoded commands. Command lines may contain passwords or tokens, so handle collected output carefully.

### 1.8 Enumerate Registered Antivirus Products

On Windows client systems where the Security Center provider is available:

```powershell
Get-CimInstance -Namespace "root\SecurityCenter2" -ClassName AntiVirusProduct -ErrorAction Stop
```

This queries antivirus products registered with Windows Security Center. The namespace may be unavailable on Windows Server or systems without the provider, and the results may not include every EDR or security component.

## 2. Service Enumeration

### 2.1 List Services

List services and their states:

```cmd
sc.exe query state= all
```

The space after `state=` is required by the `sc.exe` parser: use `state= all`, not `state=all`.

PowerShell:

```powershell
Get-Service
```

List services with additional configuration details:

```powershell
Get-CimInstance Win32_Service |
    Select-Object Name, DisplayName, State, StartMode, StartName, PathName
```

### 2.2 Query a Specific Service

```cmd
sc.exe query <service_name>
sc.exe qc <service_name>
```

For example:

```cmd
sc.exe query wuauserv
sc.exe qc wuauserv
```

The service name may differ from its display name. `query` shows the current state; `qc` shows configuration.

PowerShell:

```powershell
Get-CimInstance Win32_Service -Filter "Name='wuauserv'" |
    Format-List Name, DisplayName, State, StartMode, StartName, PathName
```

### 2.3 Service Startup Types

Service configuration is stored under:

```text
HKLM\SYSTEM\CurrentControlSet\Services\<ServiceName>
```

The `Start` value commonly means:

| Value | Startup type |
|---:|---|
| `0` | Boot |
| `1` | System |
| `2` | Automatic |
| `3` | Demand (Manual) |
| `4` | Disabled |

Query the value:

```cmd
reg query "HKLM\SYSTEM\CurrentControlSet\Services\wuauserv" /v Start
```

Startup type and current state are different: an automatic service may be stopped, while a manual service may be running.

### 2.4 Executable Path and Service Account

Use `sc qc` or `Win32_Service` to inspect the binary path and account:

```cmd
sc.exe qc <service_name>
```

```powershell
Get-CimInstance Win32_Service -Filter "Name='Spooler'" |
    Format-List Name, StartName, PathName
```

When reviewing a service, note its executable path, arguments, service account, and whether the executable exists. Check the executable and parent-directory permissions as well.

### 2.5 Service and File Permissions

Inspect the service object's security descriptor:

```cmd
sc.exe sdshow <service_name>
```

Inspect executable and parent-directory ACLs:

```cmd
icacls "C:\Path\To\Service.exe"
icacls "C:\Path\To"
```

Service-object permissions and filesystem permissions are separate. A service may have a weakly protected executable path even if its service configuration is not modifiable, or vice versa.

## 3. Quick Reference

| Objective | Command |
|---|---|
| Map processes to services | `tasklist /svc` |
| List non-SYSTEM running processes | `tasklist /FI "USERNAME ne NT AUTHORITY\SYSTEM" /FI "STATUS eq RUNNING" /V` |
| Get process executable paths | `wmic process get name,executablepath,processid` |
| Search process output | `tasklist /v \| findstr /i "<String>"` |
| Get process command lines | `wmic process get processid,commandline` |
| Query a service | `sc.exe query <service>` |
| Inspect service configuration | `sc.exe qc <service>` |
| List all services | `sc.exe query state= all` |
| Inspect service permissions | `sc.exe sdshow <service>` |
