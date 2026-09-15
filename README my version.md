# windows-psexec-style-execution-dfir-lab
## Overview
PsExec-style execution is commonly associated with Windows service-based execution. A remote administration or execution tool can create a service on a target system, configure what that service should execute, and then use the Service Control Manager to launch it.

From a DFIR perspective, the important question is not simply:

"Was PsExec used?"

Instead, investigate the evidence chain:

Service installation
        ↓
Service configuration
        ↓
Service account
        ↓
Service start/stop activity
        ↓
Process creation
        ↓
Authentication
        ↓
Network activity
        ↓
Wazuh correlation

A service named something suspicious is only a lead. Likewise, Event ID 7045 does not prove PsExec was used. Other administrative tools and legitimate software can install services.

This lab therefore focuses on identifying and correlating the artifacts associated with PsExec-style service execution.

This lab investigates a **PsExec-style Windows service execution scenario** from a DFIR perspective.

The investigation focuses on how service-based execution can leave artifacts across Windows Service Control Manager, the Windows Registry, System Event Logs, Sysmon, Windows Security logs, and Wazuh.

The lab does **not** claim to reproduce a real remote PsExec session. Instead, it creates a controlled service installation artifact and investigates whether the available telemetry is sufficient to establish successful execution or remote activity.

The key principle is:

> **A service installation artifact is evidence of service installation, not automatic proof of PsExec usage or remote execution.**

---

## Objectives

- Identify the Windows artifacts generated when a new service is installed.
- Examine Service Control Manager events associated with service creation and startup failure.
- Analyze Event ID 7045 as evidence of service installation.
- Investigate Event IDs 7000 and 7009 to understand service startup failures and timeouts.
- Examine the service's Registry configuration under HKLM\SYSTEM\CurrentControlSet\Services.
- Validate service properties such as ImagePath, StartMode, StartName, DisplayName, and Service Type.
- Compare service configuration obtained through PowerShell, CIM, and sc.exe.
- Use Sysmon Process Create (Event ID 1) and Network Connection (Event ID 3) as supporting telemetry.
- Determine whether available Sysmon events can be confidently attributed to the investigated service.
- Check Windows Security logs for available authentication and service-installation telemetry without assuming missing events.
- Investigate how Wazuh can correlate Service Control Manager and Sysmon telemetry.
- Identify telemetry gaps that prevent confirming PsExec usage or remote execution.
- Build a chronological investigation timeline from the available host artifacts.
- Distinguish between confirmed service installation, service execution, and PsExec/remote execution claims.
- Practice evidence-based SOC reasoning by separating confirmed findings from unsupported assumptions.

---

## Lab Scenario

An unfamiliar Windows service, PsExecStyleLabSvc, was identified on the endpoint. The service was configured to run as LocalSystem with a command-line executable.

The investigation focused on determining:

- How the service was created and configured.
- What Windows and Sysmon artifacts were generated.
- Whether the service successfully executed.
- Whether the activity could actually be attributed to PsExec or remote execution.
- What evidence was available in Wazuh and what telemetry was missing.

The goal was to establish a defensible conclusion from the available artifacts, rather than treating service creation alone as proof of PsExec activity.

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

