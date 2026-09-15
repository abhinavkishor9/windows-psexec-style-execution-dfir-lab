# Timeline — PsExec-Style Execution Investigation

## Host

```text
DESKTOP-9MMM37V
```

## User

```text
desktop-9mmm37v\dell
```

## Lab

```text
PsExecStyleLabSvc
```

---

## Chronological Timeline

| Time | Source | Event | Interpretation |
|---|---|---|---|
| 07:53 | PowerShell | `C:\PsExecStyleLab` created | Lab infrastructure established |
| 07:54 | PowerShell | `Evidence` directory created | Evidence collection initialized |
| 07:54:41 | PowerShell | Service baseline captured | Baseline established |
| 07:54+ | PowerShell | `PsExecStyleLabSvc` created | Controlled service artifact created |
| 07:54+ | Registry | Service Registry key created | Service configuration artifact |
| 07:54+ | PowerShell | Service observed as `Stopped` | Service had not been started |
| 07:54+ | `sc.exe` | Service configuration queried | Service configuration validated |
| 07:38:45* | System | Event 7009 | Service start timeout from earlier attempt |
| 07:38:45* | System | Event 7000 | Service start failure from earlier attempt |
| 08:01–08:10 | Sysmon | Event ID 3 | General network telemetry observed |
| 08:08–08:10 | Sysmon | Event ID 1 | General process telemetry observed |
| Investigation | Security | 4624 query | No matching events returned |
| Investigation | Security | 4625 query | No matching events returned |
| Investigation | Security | 4697 query | No matching events returned |
| Cleanup | Service Control Manager | Service deletion requested | Lab artifact removed |
| Cleanup | Registry | `PsExecStyleLabSvc` key absent | Registry cleanup verified |

> `*` These timestamps correspond to the earlier failed service-start attempt captured during the laboratory troubleshooting process. They should not be interpreted as a successful execution event.

---

## Detailed Timeline

### 07:53 — Lab Directory Created

```text
C:\PsExecStyleLab
```

The laboratory workspace was created.

---

### 07:54 — Evidence Directory Created

```text
C:\PsExecStyleLab\Evidence
```

This directory was used for investigation output.

---

### 07:54:41 — Service Baseline Captured

The existing Windows services were exported:

```powershell
Get-Service |
    Sort-Object Name |
    Select-Object Status, Name, DisplayName |
    Export-Csv "$LabPath\Evidence\Services-Baseline.csv" -NoTypeInformation
```

This established the pre-lab service state.

---

### Service Creation — `PsExecStyleLabSvc`

The controlled service was created with:

```text
Service Name: PsExecStyleLabSvc
Display Name: PsExec Style Investigation Lab
Start Type: Manual
Account: LocalSystem
```

Configured path:

```text
C:\WINDOWS\System32\cmd.exe /c exit
```

---

### Service Configuration — Registry

The following Registry location was created:

```text
HKLM\SYSTEM\CurrentControlSet\Services\PsExecStyleLabSvc
```

Relevant values:

```text
ImagePath   = C:\WINDOWS\System32\cmd.exe /c exit
ObjectName  = LocalSystem
Start       = 3
Type        = 16
```

---

### Service Start Attempt — Earlier Troubleshooting

The artificial service was previously started.

The attempt resulted in:

```text
7009 — Service start timeout
7000 — Service failed to start
```

This demonstrates that the simplified service configuration did not successfully initialize as a Windows service.

---

### Sysmon Process Telemetry

Sysmon Event ID 1 was queried.

Multiple process creation events were available around:

```text
08:08–08:10
```

However, the summarized data did not establish that these processes were created by `PsExecStyleLabSvc`.

Therefore, they remain general host telemetry rather than confirmed service-execution evidence.

---

### Sysmon Network Telemetry

Sysmon Event ID 3 was queried.

Multiple network events were available around:

```text
08:01–08:10
```

However, no summarized evidence established a network connection specifically associated with the laboratory service.

Therefore, remote activity was not established.

---

### Windows Security Telemetry

Queries for:

```text
4624 — Successful Logon
4625 — Failed Logon
4697 — Service Installation
```

returned no matching events.

This represents a telemetry limitation for the investigation.

---

## Cleanup

The service was deleted using:

```powershell
sc.exe delete PsExecStyleLabSvc
```

The service was temporarily observed as marked for deletion during troubleshooting.

After cleanup and environment reset, the Registry was checked:

```powershell
Test-Path "HKLM:\SYSTEM\CurrentControlSet\Services\PsExecStyleLabSvc"
```

Final result:

```text
False
```

This confirms that the service Registry artifact was removed.

---

# Timeline Summary

```text
07:53
Lab workspace created
        |
        v
07:54
Evidence directory created
        |
        v
07:54:41
Service baseline captured
        |
        v
Service created
PsExecStyleLabSvc
        |
        v
Service Registry configuration created
        |
        v
Service start attempted
        |
        v
7009 — Start timeout
        |
        v
7000 — Start failure
        |
        v
Sysmon investigation
        |
        v
Security telemetry investigation
        |
        v
Wazuh investigation
        |
        v
Service deleted
        |
        v
Registry cleanup verified
```

---

# Final Timeline Assessment

The timeline establishes the creation and configuration of a controlled Windows service.

It also establishes that the attempted service startup failed.

The timeline does **not** establish:

- Successful service execution
- A service-created child process
- Remote authentication
- Remote network execution
- Actual PsExec usage

## Final Verdict

**Controlled PsExec-style service installation reproduced and investigated. Successful PsExec-style execution and remote execution were not established from the available evidence.**
