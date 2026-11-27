---
layout: default
title: Windows Network and Share Enumeration
---

# Windows Network and Share Enumeration

Windows hosts may expose resources to other systems through SMB shares and may also maintain connections to remote shares. Enumerating these resources helps build a picture of the host's network role, identify potentially sensitive shared directories, and understand which remote systems are interacting with the machine.

This guide focuses on read-only enumeration for authorized assessments and controlled lab environments.

## 1. Understand the Different Share and Session Views

Several Windows commands expose different aspects of SMB and network-share activity:

| Command | What it shows |
|---|---|
| `net share` | Shares hosted by the local computer |
| `net session` | Remote SMB sessions connected to the local computer |
| `net use` | Network connections and mapped drives associated with the current logon session |
| `Get-SmbShare` | Local SMB shares, through the SMB PowerShell module |
| `Get-SmbSession` | SMB client sessions connected to the local SMB server |
| `Get-SmbConnection` | Outbound SMB connections from the local computer to remote SMB servers |

These commands are related, but they are not interchangeable. In particular, a mapped drive, a locally hosted share, and an inbound SMB session describe different things.

## 2. Enumerate Shares Hosted by the Local Computer

Use `net share` to list the shares that the local computer makes available to other systems:

```cmd
net share
```

Example output:

```text
Share name   Resource                         Remark
-------------------------------------------------------------------------------
ADMIN$       C:\Windows                       Remote Admin
C$           C:\                              Default share
IPC$                                          Remote IPC
Public       C:\Users\Public                  Public folder
```

The exact output depends on the Windows edition, configuration, installed roles, and permissions.

To inspect a specific share:

```cmd
net share Public
```

This can reveal the local filesystem path associated with the share and its configured information. It does not, by itself, establish which remote users can successfully access the underlying files. Share permissions and filesystem ACLs both affect effective access.

### PowerShell equivalent

On systems with the SMB PowerShell module:

```powershell
Get-SmbShare
```

Select useful fields:

```powershell
Get-SmbShare |
    Select-Object Name, Path, Description, ScopeName, Special
```

To inspect permissions assigned at the share level:

```powershell
Get-SmbShareAccess -Name "Public"
```

Share-level permissions are only one part of the access decision. Also inspect the NTFS permissions on the backing directory:

```cmd
icacls "C:\Users\Public"
```

Do not infer effective access from share permissions alone.

## 3. Enumerate Remote Sessions Connected to the Local Computer

Use `net session` to view remote SMB sessions established with the local computer:

```cmd
net session
```

The output can include the remote computer, user name, client type, number of open files, and session duration or idle information, depending on the Windows version.

This command is useful when assessing a host that may act as a file server or otherwise accept inbound SMB connections. It can help identify which remote systems and accounts are currently connected.

### Important permission and service considerations

Querying sessions commonly requires administrative privileges. The command may fail with an access-denied error when run from a standard user context.

The local Server service must also be available for the computer to provide SMB server functionality. If the service is disabled or the host is not accepting SMB connections, there may be no sessions to display.

An empty result does not prove that the host has never been accessed over SMB; it only reflects the sessions visible at the time of the query.

### PowerShell equivalent

```powershell
Get-SmbSession
```

For a more focused view:

```powershell
Get-SmbSession |
    Select-Object ClientComputerName, ClientUserName,
                  NumOpens, Dialect, SecondsExists, SecondsIdle
```

Availability and returned properties can vary by Windows version and SMB module support.

## 4. Enumerate Outbound Network Share Connections

`net use` displays network connections associated with the current logon session:

```cmd
net use
```

It can show mapped drive letters and connections to remote resources, for example:

```text
Status       Local     Remote
-------------------------------------------------------------------------------
OK           Z:        \\FILESERVER\Shared
Unavailable  Y:        \\BACKUP01\Archive
```

A connection may exist without a persistent drive-letter mapping. Therefore, do not treat `net use` output as a complete history of all SMB resources the user has ever accessed.

PowerShell can show SMB connections maintained by the local SMB client:

```powershell
Get-SmbConnection
```

Select useful fields:

```powershell
Get-SmbConnection |
    Select-Object ServerName, ShareName, UserName,
                  Credential, Dialect, NumOpens
```

These are outbound connections from the local computer to remote SMB servers. They are different from the inbound sessions shown by `net session` or `Get-SmbSession`.

## 5. Understand Special and Administrative Shares

Windows commonly creates default administrative shares, depending on the system role and configuration:

| Share | Typical purpose |
|---|---|
| `C$`, `D$`, etc. | Administrative access to the root of a volume |
| `ADMIN$` | Remote administration through the Windows directory |
| `IPC$` | Inter-process communication and named-pipe access |

Their presence is not automatically evidence of a vulnerability. Whether they can be accessed remotely depends on authentication, account privileges, policy, firewall rules, and other configuration.

Record unexpected or unnecessarily exposed shares, but validate access using only the credentials and methods permitted by the engagement scope.

## 6. Practical Enumeration Workflow

A useful read-only workflow is:

1. Run `net share` to identify shares hosted by the local computer.
2. Use `Get-SmbShare` when the SMB PowerShell module is available.
3. Review share permissions with `Get-SmbShareAccess`.
4. Review the backing directory's NTFS ACLs with `icacls`.
5. Run `net session` or `Get-SmbSession` to inspect inbound SMB sessions, if permissions allow.
6. Run `net use` to identify mapped drives and connections in the current logon session.
7. Use `Get-SmbConnection` to inspect outbound SMB connections.
8. Record the execution context and any access-denied or service-related limitations.

## 7. Interpretation Notes

Keep the following distinctions clear when documenting results:

- A **share** is a resource exported by a computer.
- A **mapped drive** is a drive-letter mapping to a remote resource in a particular logon context.
- An **inbound SMB session** is a remote client connected to the local SMB server.
- An **outbound SMB connection** is the local computer connected to a remote SMB server.
- Share permissions and NTFS permissions jointly influence access to files and directories.

The commands above provide current configuration or connection information; they are not a complete historical record of every share or SMB connection.

## 8. Reporting Notes

For each relevant finding, record:

- Hostname and assessment context.
- Share name and backing path.
- Whether the share is administrative or custom.
- Share-level permissions, where available.
- Filesystem ACLs on the backing path.
- Relevant inbound sessions or outbound connections.
- The account and privilege context used to run the commands.
- Any limitations, such as access denied, unavailable SMB cmdlets, or inactive Server service.

Distinguish an exposed share from a confirmed access-control weakness. Do not access, copy, or modify files outside the authorized scope.
