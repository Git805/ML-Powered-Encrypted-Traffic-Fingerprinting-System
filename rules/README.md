# Detection Rules Library

Companion Sigma detection rules for the ML-Powered Encrypted Traffic 
Fingerprinting System. All rules target Zeek log sources and can be 
converted using sigmac for SIEM deployment.

## Rules Index

| Rule | Technique | Tactic | Level | Log Source | Status |
|---|---|---|---|---|---|
| [c2_beaconing_periodic.yml](c2_beaconing_periodic.yml) | T1071.001 | Command & Control | Medium | Zeek conn.log | Experimental |
| [tls_ja3_anomaly.yml](tls_ja3_anomaly.yml) | T1071.001, T1573 | Command & Control | High | Zeek ssl.log | Experimental |
| [high_entropy_dns.yml](high_entropy_dns.yml) | T1071.004, T1572 | Command & Control | Medium | Zeek dns.log | Experimental |
| [valid_accounts_credential_abuse.yml](valid_accounts_credential_abuse.yml) | T1078 | Initial Access, Persistence | Medium | Windows Security Events | Experimental |
| [lateral_movement_rdp.yml](lateral_movement_rdp.yml) | T1021.001 | Lateral Movement | Medium | Windows Security Events | Experimental |
| [process_injection.yml](process_injection.yml) | T1055 | Defence Evasion, Privilege Escalation | High | Sysmon EventID 8, 10 | Experimental |
| [masquerading.yml](masquerading.yml) | T1036 | Defence Evasion | High | Sysmon EventID 1 | Experimental |
| [powershell_suspicious_execution.yml](powershell_suspicious_execution.yml) | T1059.001, T1027 | Execution, Defence Evasion | High | Sysmon EventID 1, 4104 | Experimental |
| [impair_defences.yml](impair_defences.yml) | T1562, T1490 | Defence Evasion | Critical | Sysmon EventID 1, Windows Security | Experimental |

## MITRE ATT&CK Coverage

```
Command and Control (TA0011)
├── T1071.001  Application Layer Protocol: Web Protocols  ← c2_beaconing_periodic, tls_ja3_anomaly
├── T1071.004  Application Layer Protocol: DNS            ← high_entropy_dns
├── T1572      Protocol Tunneling                         ← high_entropy_dns
└── T1573      Encrypted Channel                          ← tls_ja3_anomaly

Lateral Movement (TA0008)
└── T1021.001  Remote Services: RDP                      ← lateral_movement_rdp

Initial Access / Persistence / Defence Evasion
└── T1078      Valid Accounts                             ← valid_accounts_credential_abuse

Defence Evasion / Privilege Escalation
├── T1055  Process Injection    ← process_injection
└── T1036  Masquerading         ← masquerading

Execution (TA0002)
└── T1059.001  Command and Scripting Interpreter: PowerShell  ← powershell_suspicious_execution

Defence Evasion (TA0005) — continued
├── T1562      Impair Defenses                                ← impair_defences
├── T1562.001  Disable or Modify Tools
├── T1562.002  Disable Windows Event Logging
└── T1490      Inhibit System Recovery (Shadow Copies)        ← impair_defences
```

## Convert to Your SIEM

Install sigmac:

```bash
pip install sigmatools
```

Splunk:
```bash
sigmac -t splunk rules/c2_beaconing_periodic.yml
```

QRadar:
```bash
sigmac -t qradar rules/tls_ja3_anomaly.yml
```

Elastic/KQL:
```bash
sigmac -t es-qs rules/lateral_movement_rdp.yml
```

## Stacking Rules for Higher Confidence

| Combination | Confidence | Action |
|---|---|---|
| c2_beaconing_periodic only | Medium | Investigate |
| tls_ja3_anomaly (known bad JA3) | High | Alert SOC |
| c2_beaconing + tls_ja3_anomaly same host | Very High | Page on-call |
| valid_accounts + lateral_movement_rdp same account | High | Isolate host |

## Author

**Charudatta Padhye**
NDR Solutions Engineer | Detection Engineering | ML for Security

[LinkedIn](https://www.linkedin.com/in/charudatta-padhye-02a281190/) · 
[GitHub](https://github.com/Git805) · 
[Blog](https://dev.to/charudatta29)
