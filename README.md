# Windows PsExec-Style Execution DFIR Lab

## Overview

This lab investigates a **PsExec-style Windows service execution scenario** from a DFIR perspective.

The investigation focuses on how service-based execution can leave artifacts across Windows Service Control Manager, the Windows Registry, System Event Logs, Sysmon, Windows Security logs, and Wazuh.

The lab does **not** claim to reproduce a real remote PsExec session. Instead, it creates a controlled service installation artifact and investigates whether the available telemetry is sufficient to establish successful execution or remote activity.

The key principle is:

> **A service installation artifact is evidence of service installation, not automatic proof of PsExec usage or remote execution.**

---

## Objectives

- Understand the forensic significance of Windows service installation.
- Investigate Service Control Manager artifacts.
- Examine Windows Event ID 7045.
- Examine service configuration through PowerShell and `sc.exe`.
- Investigate the corresponding service Registry key.
- Examine the configured service account and binary path.
- Investigate Sysmon Process Create telemetry.
- Check Windows authentication events.
- Check Sysmon network telemetry.
- Correlate available evidence through Wazuh.
- Identify telemetry gaps without fabricating events.
- Build an evidence-based execution timeline.
- Distinguish PsExec-style activity from confirmed PsExec usage.

---

## Concept

PsExec-style execution commonly relies on Windows service functionality to execute a process on a target system.

From a DFIR perspective, an analyst should investigate multiple artifacts rather than relying on a single indicator.

A useful investigation model is:

```text
Service Installation
        |
        v
Service Configuration
        |
        v
Service Account
        |
        v
Service Start / Failure
        |
        v
Process Creation
        |
        v
Authentication
        |
        v
Network Activity
        |
        v
Wazuh Correlation
```

The presence of a newly installed service can be suspicious, but legitimate software, administrators, installers, and Windows components can also create services.

Therefore:

```text
7045 != PsExec proof
Service creation != Remote execution proof
Service name != Maliciousness proof
```

---

## Lab Scenario

A SOC analyst notices an unfamiliar Windows service on a workstation.

The service name is:

```text
PsExecStyleLabSvc
```

Because service-based execution is associated with tools such as PsExec, the analyst wants to determine:

1. When the service was installed.
2. What executable was configured.
3. Which account was configured to run it.
4. Whether the service was successfully started.
5. Whether a corresponding process was created.
6. Whether authentication evidence exists.
7. Whether network evidence supports remote activity.
8. Whether the evidence is sufficient to attribute the activity to PsExec.

A controlled laboratory service is created to reproduce the service-installation artifact.

---

## Environment

- Windows VM
- PowerShell 7.6.6
- Sysmon
- Wazuh Agent
- Windows Event Logs

Environment observed during the investigation:

```text
Hostname: DESKTOP-9MMM37V
User: desktop-9mmm37v\dell
PowerShell: 7.6.6
Sysmon: Running
WazuhSvc: Running
```

---

## Lab Directory

```text
C:\PsExecStyleLab
└── Evidence
```

---

## Investigation Workflow

### 1. Establish the Environment

```powershell
hostname
whoami
$PSVersionTable.PSVersion
Get-Service Sysmon64
Get-Service WazuhSvc
```

---

### 2. Create the Evidence Directory

```powershell
$LabPath = "C:\PsExecStyleLab"

New-Item -ItemType Directory -Path "$LabPath\Evidence" -Force
```

---

### 3. Capture a Service Baseline

```powershell
Get-Service |
    Sort-Object Name |
    Select-Object Status, Name, DisplayName |
    Export-Csv "$LabPath\Evidence\Services-Baseline.csv" -NoTypeInformation
```

Record the time:

```powershell
Get-Date
```

---

### 4. Create the Controlled Service Artifact

The controlled service was created with:

```powershell
$ServiceName = "PsExecStyleLabSvc"

New-Service `
    -Name $ServiceName `
    -BinaryPathName "$env:windir\System32\cmd.exe /c exit" `
    -DisplayName "PsExec Style Investigation Lab" `
    -Description "Temporary DFIR laboratory service" `
    -StartupType Manual
```

The service was successfully created and initially appeared as:

```text
Status   Name               DisplayName
------   ----               -----------
Stopped  PsExecStyleLabSvc  PsExec Style Investigation Lab
```

---

### 5. Investigate Service Configuration

```powershell
Get-CimInstance Win32_Service -Filter "Name='PsExecStyleLabSvc'" |
    Select-Object Name, State, StartMode, StartName, PathName
```

The observed configuration was:

```text
Name      : PsExecStyleLabSvc
State     : Stopped
StartMode : Manual
StartName : LocalSystem
PathName  : C:\WINDOWS\System32\cmd.exe /c exit
```

The configuration was also queried using:

```powershell
sc.exe qc PsExecStyleLabSvc
```

Important fields included:

```text
TYPE               : 10  WIN32_OWN_PROCESS
START_TYPE         : 3   DEMAND_START
BINARY_PATH_NAME   : C:\WINDOWS\System32\cmd.exe /c exit
SERVICE_START_NAME : LocalSystem
```

---

### 6. Investigate the Service Registry Artifact

Windows service configuration can be examined under:

```text
HKLM\SYSTEM\CurrentControlSet\Services
```

The lab service was queried with:

```powershell
Get-ItemProperty `
    "HKLM:\SYSTEM\CurrentControlSet\Services\PsExecStyleLabSvc"
```

Observed values included:

```text
Type         : 16
Start        : 3
ErrorControl : 1
ImagePath    : C:\WINDOWS\System32\cmd.exe /c exit
DisplayName  : PsExec Style Investigation Lab
ObjectName   : LocalSystem
Description  : Temporary DFIR laboratory service
```

---

### 7. Investigate Service Start Behavior

The original service configuration was intentionally simple and was not a valid standalone Windows service implementation.

An attempted start produced:

```text
7009 - Service start timeout
7000 - Service failed to start
```

The service therefore did not provide reliable evidence of successful service-based process execution.

The investigation was changed to focus on the **service installation and configuration artifacts** instead of repeatedly attempting to start the artificial service.

---

### 8. Investigate Sysmon

Sysmon Event ID 1 was queried:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName = "Microsoft-Windows-Sysmon/Operational"
    Id = 1
} -MaxEvents 100 |
Select-Object TimeCreated, Id, Message
```

Sysmon Event ID 3 was also queried:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName = "Microsoft-Windows-Sysmon/Operational"
    Id = 3
} -MaxEvents 100 |
Select-Object TimeCreated, Message
```

The environment contained numerous process-creation and network-connection events.

However, the summarized output did not establish that these events were generated by `PsExecStyleLabSvc`.

Therefore, the events were not treated as proof of PsExec-style execution.

---

### 9. Investigate Windows Security Events

Successful logons:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName = "Security"
    Id = 4624
} -MaxEvents 50
```

Failed logons:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName = "Security"
    Id = 4625
} -MaxEvents 50
```

Service installation auditing:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName = "Security"
    Id = 4697
} -MaxEvents 20
```

The environment returned no matching events for the queried Security event IDs.

These telemetry gaps were recorded rather than replaced with assumed events.

---

## Wazuh Investigation

Wazuh was used as a centralized investigation source.

Suggested searches include:

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

and, where archived events are available:

```text
wazuh-archives-*
```

The investigation should determine whether Windows service, Sysmon, and authentication telemetry is actually being collected by the Wazuh agent.

---

## Evidence Correlation

| Artifact | Finding | Interpretation |
|---|---|---|
| Service object | `PsExecStyleLabSvc` created | Confirms controlled service creation |
| Service configuration | `cmd.exe /c exit` | Shows configured executable |
| Service Registry | Matching `ImagePath` and account | Independent configuration evidence |
| Service state | Initially stopped | Service was created but not yet executing |
| 7009 | Start timeout | Service did not report successful startup |
| 7000 | Start failure | Service execution attempt failed |
| Sysmon 1 | Process telemetry available | General process visibility exists |
| Sysmon 3 | Network telemetry available | General network visibility exists |
| Security 4624/4625 | No matching events returned | Authentication telemetry gap |
| Security 4697 | No matching events returned | Service-installation audit telemetry unavailable |
| Wazuh | Used for centralized searches | Correlation depends on collected telemetry |

---

## Investigation Verdict

**Verdict: Controlled service installation confirmed; PsExec usage and remote execution not established.**

The investigation confirmed the creation and configuration of a controlled Windows service named `PsExecStyleLabSvc`.

The service Registry configuration and Service Control Manager configuration were consistent with the created laboratory artifact.

The service did not successfully execute because the simplified `cmd.exe /c exit` configuration did not behave as a proper Windows service. Consequently, the lab does not establish a successful service-based process execution chain.

No matching Security 4624, 4625, or 4697 events were available from the queried telemetry.

Therefore:

```text
Service installation       CONFIRMED
Service configuration      CONFIRMED
Successful service start   NOT ESTABLISHED
Process execution by lab service  NOT ESTABLISHED
Remote authentication     NOT ESTABLISHED
Remote execution          NOT ESTABLISHED
PsExec attribution        NOT ESTABLISHED
```

---

## DFIR Takeaway

The main lesson from this lab is that **PsExec-style indicators must be correlated rather than interpreted in isolation**.

A service installation can provide a valuable investigative lead, but attribution requires additional evidence such as:

- Service creation
- Service configuration
- Process creation
- Account context
- Authentication
- Network activity
- Consistent timestamps

The correct DFIR approach is:

> **Follow the evidence, not the assumption.**
