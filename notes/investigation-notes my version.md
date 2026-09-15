# Investigation Notes

---

## Initial Environment Validation

The following commands confirmed the laboratory environment:

```powershell
hostname
whoami
$PSVersionTable.PSVersion
Get-Service Sysmon64
Get-Service WazuhSvc
```

Observed:

```text
Hostname: DESKTOP-9MMM37V
User: desktop-9mmm37v\dell
PowerShell: 7.6.6
Sysmon: Running
Wazuh: Running
```

---

## Service Artifact

The controlled service was created as:

```text
Service Name: PsExecStyleLabSvc
Display Name: PsExec Style Investigation Lab
Startup Type: Manual
Service Account: LocalSystem
```

Configured image path:

```text
C:\WINDOWS\System32\cmd.exe /c exit
```

---

## Service Registry Evidence

Registry location:

```text
HKLM\SYSTEM\CurrentControlSet\Services\PsExecStyleLabSvc
```

Observed:

```text
Type         : 16
Start        : 3
ErrorControl : 1
ImagePath    : C:\WINDOWS\System32\cmd.exe /c exit
DisplayName  : PsExec Style Investigation Lab
ObjectName   : LocalSystem
Description  : Temporary DFIR laboratory service
```

This independently confirms the service configuration.

---

## Service Control Manager Evidence

The service configuration was validated with:

```powershell
sc.exe qc PsExecStyleLabSvc
```

Observed:

```text
TYPE               : 10  WIN32_OWN_PROCESS
START_TYPE         : 3   DEMAND_START
BINARY_PATH_NAME   : C:\WINDOWS\System32\cmd.exe /c exit
SERVICE_START_NAME : LocalSystem
```

---

## Service Start Investigation

An attempt was made to start the original controlled service.

The service did not successfully start.

The associated Windows System telemetry included:

```text
7009 — Service start timeout
7000 — Service failed to start
```

Interpretation:

```text
Service installed
      |
      v
Start attempted
      |
      v
Service did not report successful startup
      |
      v
Start timeout / failure
```

This does not establish successful service-based execution.

---

## Sysmon Investigation

Sysmon Event ID 1 was queried to investigate process creation:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName = "Microsoft-Windows-Sysmon/Operational"
    Id = 1
} -MaxEvents 100 |
Select-Object TimeCreated, Id, Message
```

The system contained numerous Sysmon Event ID 1 records.

However, the summarized output did not provide sufficient information to attribute those process events to `PsExecStyleLabSvc`.

Therefore, the available Sysmon Event ID 1 data was treated as **general process telemetry**, not as confirmed execution evidence for the lab service.

---

## Network Investigation

Sysmon Event ID 3 was queried:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName = "Microsoft-Windows-Sysmon/Operational"
    Id = 3
} -MaxEvents 100 |
Select-Object TimeCreated, Message
```

Numerous network events were present.

However, the summarized results did not establish a network connection associated with `PsExecStyleLabSvc`.

Therefore:

```text
Network telemetry available: YES
Network evidence for this service: NOT ESTABLISHED
```

---

## Windows Security Investigation

### Event ID 4624

```powershell
Get-WinEvent -FilterHashtable @{
    LogName = "Security"
    Id = 4624
} -MaxEvents 50
```

Result:

```text
No events were found that match the specified selection criteria.
```

### Event ID 4625

```powershell
Get-WinEvent -FilterHashtable @{
    LogName = "Security"
    Id = 4625
} -MaxEvents 50
```

Result:

```text
No events were found that match the specified selection criteria.
```

### Event ID 4697

```powershell
Get-WinEvent -FilterHashtable @{
    LogName = "Security"
    Id = 4697
} -MaxEvents 20
```

Result:

```text
No events were found that match the specified selection criteria.
```

These results were treated as telemetry gaps.

No Security events were fabricated.

---

## Wazuh Investigation

Wazuh was available and running during the investigation.

Relevant investigation terms:

```text
PsExecStyleLabSvc
```

```text
7045
```

```text
7000
```

```text
7009
```

```text
cmd.exe
```

Archived events can be investigated using:

```text
wazuh-archives-*
```

The purpose of the Wazuh investigation was to determine whether the Windows artifacts observed locally were also available through centralized telemetry.

---

## Evidence Assessment

| Evidence | Status | Assessment |
|---|---|---|
| Controlled service created | Confirmed | Direct laboratory observation |
| Service configuration | Confirmed | PowerShell / `sc.exe` |
| Service Registry configuration | Confirmed | Registry evidence |
| Service start | Failed | 7000/7009 |
| Sysmon process telemetry | Available | Not attributed to lab service |
| Sysmon network telemetry | Available | Not attributed to lab service |
| 4624 | Unavailable | Telemetry gap |
| 4625 | Unavailable | Telemetry gap |
| 4697 | Unavailable | Telemetry gap |
| Remote execution | Not established | Insufficient evidence |
| PsExec attribution | Not established | Insufficient evidence |

---

## Final Assessment

The investigation confirms that a controlled Windows service named `PsExecStyleLabSvc` was created and configured to run under `LocalSystem`.

The Registry and Service Control Manager views provide consistent evidence of that configuration.

The service did not successfully start because the simplified laboratory service configuration did not implement a complete Windows service interface. Consequently, no successful service-execution chain was established.

The available Security telemetry did not provide 4624, 4625, or 4697 events.

Sysmon provided process and network telemetry, but the summarized events did not establish a causal relationship with the laboratory service.

### Verdict

**Confirmed controlled service installation. PsExec usage and remote execution not established.**

---

## DFIR Lesson

The investigation demonstrates why a SOC analyst should avoid jumping from:

```text
New service
```

directly to:

```text
PsExec attack
```

A defensible conclusion requires correlation between:

```text
Service
+
Process
+
Account
+
Authentication
+
Network
+
Timeline
```

Where telemetry is missing, the correct conclusion is an **evidence gap**, not an invented finding.
