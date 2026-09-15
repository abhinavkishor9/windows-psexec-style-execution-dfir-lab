# Timeline 


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

