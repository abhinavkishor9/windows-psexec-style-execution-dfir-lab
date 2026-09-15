# Troubleshooting Notes

## Issue 1 — Service Already Exists

### Error

```text
The specified service already exists.
```

### Cause

`PsExecStyleLabSvc` had already been created during an earlier attempt.

### Resolution

The service was deleted using:

```powershell
sc.exe delete PsExecStyleLabSvc
```

---

## Issue 2 — Service Failed to Start

### Command

```powershell
Start-Service PsExecStyleLabSvc
```

### Error

```text
Cannot start service 'PsExecStyleLabSvc'
```

### Investigation

The service configuration was checked:

```powershell
sc.exe qc PsExecStyleLabSvc
```

Observed:

```text
BINARY_PATH_NAME   : C:\WINDOWS\System32\cmd.exe /c exit
SERVICE_START_NAME : LocalSystem
```

The service was configured to launch:

```text
cmd.exe /c exit
```

A normal `cmd.exe` process is not a complete Windows service implementation. Service Control Manager therefore waited for the expected service startup response.

---

## Issue 3 — Service Start Timeout

System events showed:

```text
7009 — Service start timeout
7000 — Service failed to start
```

### Interpretation

The service installation succeeded, but the service did not successfully initialize as a Windows service.

This was retained as useful investigation evidence.

---

## Issue 4 — Service Became Stuck in START_PENDING

After attempting to start the service and subsequently deleting it, the service temporarily remained visible as:

```text
STATE : 2  START_PENDING
```

The deletion operation then returned:

```text
The specified service has been marked for deletion.
```

### Cause

Windows had marked the service for deletion but still had an active service/process handle.

### Resolution

The service was not repeatedly recreated or manipulated while deletion was pending.

The Windows VM was restarted to allow the pending deletion to complete.

After cleanup:

```powershell
Test-Path "HKLM:\SYSTEM\CurrentControlSet\Services\PsExecStyleLabSvc"
```

returned:

```text
False
```

The service was therefore successfully removed.

---

## Issue 5 — Security Event ID 4697 Not Available

### Command

```powershell
Get-WinEvent -FilterHashtable @{
    LogName = "Security"
    Id = 4697
} -MaxEvents 20
```

### Result

```text
No events were found that match the specified selection criteria.
```

### Resolution

No 4697 event was fabricated.

The investigation instead relied on:

- Service configuration
- Service Registry artifact
- System service-related telemetry where available
- Sysmon
- Wazuh

### DFIR Principle

> Missing telemetry should be documented as an evidence gap rather than replaced with an assumed event.

---

## Issue 6 — Security Events 4624 and 4625 Not Available

The following queries returned no matching events:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName = "Security"
    Id = 4624
} -MaxEvents 50
```

```powershell
Get-WinEvent -FilterHashtable @{
    LogName = "Security"
    Id = 4625
} -MaxEvents 50
```

### Interpretation

The queried Security telemetry did not contain the requested events.

This means authentication-based attribution could not be established from those queries.

---

## Issue 7 — `$LabPath` Became Invalid

Some commands produced paths such as:

```text
C:\Evidence\ServiceConfiguration.txt
```

instead of:

```text
C:\PsExecStyleLab\Evidence\ServiceConfiguration.txt
```

### Cause

The PowerShell session no longer contained the expected `$LabPath` variable.

The variable had previously been:

```powershell
$LabPath = "C:\PsExecStyleLab"
```

but was unavailable in the later session.

### Resolution

Reinitialize the variable:

```powershell
$LabPath = "C:\PsExecStyleLab"
```

Then verify:

```powershell
$LabPath
```

Expected:

```text
C:\PsExecStyleLab
```

Create the evidence directory if necessary:

```powershell
New-Item -ItemType Directory -Path "$LabPath\Evidence" -Force
```

### Best Practice

For reproducible DFIR work, initialize important path variables at the beginning of each PowerShell session.

---

## Issue 8 — Evidence Directory Did Not Initially Exist

A cleanup command returned:

```text
Cannot find path 'C:\PsExecStyleLab'
```

### Cause

The laboratory directory had not yet been recreated in that session.

### Resolution

```powershell
$LabPath = "C:\PsExecStyleLab"

New-Item -ItemType Directory -Path "$LabPath\Evidence" -Force
```

---

## Issue 9 — Sysmon Returned Many Events

Sysmon Event ID 1 returned numerous process creation events:

```text
15-09-2026 08:10:42
15-09-2026 08:10:37
15-09-2026 08:10:32
...
```

Sysmon Event ID 3 also returned numerous network events.

### Important

The presence of these events does **not** automatically connect them to `PsExecStyleLabSvc`.

The summarized output did not contain enough information to establish that relationship.

### Correct approach

Inspect relevant Sysmon fields:

```text
Image
CommandLine
ParentImage
ParentCommandLine
User
ProcessId
ParentProcessId
SourceIp
DestinationIp
DestinationPort
```

Only attribute an event to the lab service when the available evidence supports that conclusion.

---

## Issue 10 — Do Not Repeatedly Attempt to Start the Artificial Service

The original laboratory service used:

```text
cmd.exe /c exit
```

This was useful for creating a service artifact but was not a suitable standalone Windows service implementation.

Repeatedly attempting to start it produced unnecessary timeout/failure behavior.

