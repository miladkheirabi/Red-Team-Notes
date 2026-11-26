---
layout: default
title: Windows Services Enumeration
---

# Windows Services Enumeration

Windows services are background components managed by the Service Control Manager (SCM). They may start automatically during boot, start when requested, or remain disabled. Enumerating services helps build an understanding of the host's software, operational configuration, and security boundaries.

The commands in this guide are intended for authorized assessments and controlled lab environments.

## 1. What Is a Windows Service?

A service is a program or component designed to run under the control of the Service Control Manager. Services can run under built-in service accounts, local user accounts, or domain accounts.

A service configuration commonly includes:

- Service name and display name.
- Executable path and command-line arguments.
- Service account (the account used to run the service).
- Startup type.
- Current state.
- Dependencies and recovery settings.
- Permissions on the service object and its executable files.

A service's existence does not imply that it is running, automatically started, or misconfigured.

## 2. Enumerate Services

### 2.1 Command Prompt

List services and their states:

```cmd
sc query
```

List configuration details for a service:

```cmd
sc qc <service_name>
```

Example:

```cmd
sc qc Spooler
```

Query the current state:

```cmd
sc query Spooler
```

The service name is not always identical to its display name. `sc` is the Service Control utility; it can query and manage service configuration, subject to permissions.

### 2.2 PowerShell

List services:

```powershell
Get-Service
```

Include services with additional CIM properties:

```powershell
Get-CimInstance Win32_Service |
    Select-Object Name, DisplayName, State, StartMode, StartName, PathName
```

Filter for running services:

```powershell
Get-Service | Where-Object Status -eq 'Running'
```

Find services configured for automatic startup:

```powershell
Get-CimInstance Win32_Service |
    Where-Object StartMode -eq 'Auto' |
    Select-Object Name, DisplayName, State, StartName, PathName
```

## 3. Understand Service Startup Types

The service configuration is represented in the registry under:

```text
HKLM\SYSTEM\CurrentControlSet\Services\<ServiceName>
```

The `Start` value commonly uses the following numbers:

| Value | Meaning |
|---:|---|
| `0` | Boot start |
| `1` | System start |
| `2` | Automatic start |
| `3` | Demand start (manual) |
| `4` | Disabled |

These values describe startup configuration, not the service's current running state. For example, an automatic service may currently be stopped, and a demand-start service may currently be running.

Query a service's registry configuration:

```cmd
reg query "HKLM\SYSTEM\CurrentControlSet\Services\Spooler" /v Start
```

Query the service's full registry key:

```cmd
reg query "HKLM\SYSTEM\CurrentControlSet\Services\Spooler"
```

The effective configuration may involve control sets and other service settings. Use the Service Control Manager interfaces as well as registry data when validating a finding.

## 4. Inspect the Executable Path and Service Account

Use `sc qc` to inspect a service's configured binary path and account:

```cmd
sc qc <service_name>
```

PowerShell can return these fields in a structured format:

```powershell
Get-CimInstance Win32_Service -Filter "Name='Spooler'" |
    Format-List Name, DisplayName, State, StartMode, StartName, PathName
```

When reviewing a binary path, consider:

- Whether the path is fully qualified.
- Whether arguments are present.
- Whether the executable exists.
- Which account runs the service.
- Whether the executable and its parent directories have appropriate permissions.
- Whether the service configuration is consistent with the software's expected behavior.

Do not conclude that a service is vulnerable merely because its executable is located in a particular directory. Validate the actual ACLs, quoting, execution context, and service behavior.

## 5. Inspect Service and File Permissions

Service object permissions and filesystem permissions are separate controls.

You can inspect the security descriptor exposed by `sc`:

```cmd
sc sdshow <service_name>
```

The output is a security descriptor in SDDL format. Interpreting it requires understanding the relevant access-control rights and trustees.

Inspect the executable and its parent directory ACLs:

```cmd
icacls "C:\Path\To\Service.exe"
icacls "C:\Path\To"
```

A service may be configured securely while its executable directory has weak permissions, or the reverse. Assess both the service object's permissions and the filesystem permissions relevant to the service.

Querying service information is often available to standard users, but changing service configuration, controlling certain services, or accessing protected files generally requires appropriate rights. Exact access depends on the service's security descriptor and the current token.

## 6. Useful Service Control Commands

```cmd
sc query <service_name>
sc qc <service_name>
sc sdshow <service_name>
```

These commands are useful for read-only enumeration. Commands that start, stop, create, delete, or reconfigure services change system state and should only be used when explicitly authorized and necessary for the assessment.

## 7. Common Enumeration Workflow

1. List installed services.
2. Identify running and stopped services.
3. Record startup modes.
4. Inspect service accounts.
5. Review executable paths and arguments.
6. Inspect service-object permissions.
7. Inspect executable and parent-directory ACLs.
8. Validate any suspected misconfiguration before reporting it.

## 8. Reporting Notes

A useful service finding should identify the service name, host, observed configuration, relevant permissions, security impact, and evidence. Clearly distinguish a confirmed weakness from a condition that merely deserves further review.

Do not make configuration changes during enumeration unless the rules of engagement explicitly permit them.
