# 🛡️ Home SOC Lab: Brute Force Detection with Wazuh SIEM

A home cybersecurity lab simulating a real-world SSH brute force attack using Hydra from Kali Linux, detected in real time by a Wazuh SIEM — built entirely on free, open-source tools.

---

## 📌 Overview

This project demonstrates hands-on blue team skills: deploying a SIEM, onboarding a monitored endpoint, simulating an attack, and analyzing the resulting alerts. All machines run locally inside VirtualBox.

| Goal | Details |
|------|---------|
| **Simulate** | SSH brute force attack using Hydra |
| **Detect** | Real-time alerting via Wazuh SIEM |
| **Enrich** | Deep log visibility with Sysmon-equivalent (auditd) |
| **Analyze** | Trace attacker IP, username, and timestamps in Wazuh dashboard |

---

## 🖥️ Lab Architecture

```
┌─────────────────────────────────────────────────────┐
│              VirtualBox NAT Network                  │
│                  192.168.40.0/24                     │
│                                                      │
│  ┌─────────────────┐       ┌──────────────────────┐  │
│  │   Kali Linux    │──────▶│   Ubuntu (Victim)    │  │
│  │  192.168.40.123 │ SSH   │   192.168.40.159     │  │
│  │  Hydra attack   │ :22   │   Wazuh Agent        │  │
│  └─────────────────┘       └──────────┬───────────┘  │
│                                       │ logs          │
│                            ┌──────────▼───────────┐  │
│                            │    Wazuh SIEM        │  │
│                            │   192.168.40.58      │  │
│                            │   Dashboard + Alerts │  │
│                            └──────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

![Network Diagram](screenshots/network-diagram.png)
*Figure: Lab network — Kali attacker, Ubuntu victim, and Wazuh SIEM on VirtualBox NAT network*

---

## 🧰 Tools & Technologies

| Tool | Role | Version |
|------|------|---------|
| [VirtualBox](https://www.virtualbox.org/) | Hypervisor | Latest |
| [Wazuh](https://wazuh.com/) | SIEM / Log Analysis | 4.x OVA |
| [Ubuntu Server](https://ubuntu.com/server) | Victim endpoint | 22.04 LTS |
| [Kali Linux](https://www.kali.org/) | Attacker machine | Latest |
| [Hydra](https://github.com/vanhauser-thc/thc-hydra) | Brute force tool | Built-in on Kali |
| [auditd](https://linux.die.net/man/8/auditd) | System call logging | Built-in on Ubuntu |

---

## ⚙️ Lab Setup

### Phase 1 — Requirements

- Computer with **16GB RAM minimum** (32GB recommended)
- **100GB free SSD space**
- VirtualBox installed

### Phase 2 — Deploy Wazuh SIEM

1. Download the [Wazuh OVA](https://wazuh.com/install/) (pre-configured appliance)
2. In VirtualBox: **File → Import Appliance** → select the OVA
3. Set network adapter to **NAT Network** (shared with other VMs)
4. Start the VM and log in: `admin / admin`
5. Note the IP address:
```bash
ip a
# e.g. 192.168.40.58
```

### Phase 3 — Set Up Ubuntu Victim

1. Create a new VM using Ubuntu Server ISO
2. Set the same **NAT Network** as Wazuh
3. Install and enable SSH:
```bash
sudo apt update && sudo apt install openssh-server -y
sudo systemctl enable ssh && sudo systemctl start ssh
```
4. Navigate to `https://192.168.40.58` from the Ubuntu VM
5. In the Wazuh dashboard: **Add Agent → Linux → copy the install command**
6. Run the command on Ubuntu and start the agent:
```bash
sudo systemctl enable wazuh-agent && sudo systemctl start wazuh-agent
```
7. Verify: the Wazuh dashboard should show **1 Active Agent**

### Phase 4 — Enrich Logs with auditd

Standard Linux logs miss key attacker actions. Enrich them:

```bash
sudo apt install auditd -y
sudo systemctl enable auditd && sudo systemctl start auditd

# Watch for failed login attempts
sudo auditctl -w /var/log/auth.log -p rwxa -k auth_log
```

Then update the Wazuh agent config to ingest auth logs:

```xml
<!-- /var/ossec/etc/ossec.conf on Ubuntu -->
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/auth.log</location>
</localfile>
```

```bash
sudo systemctl restart wazuh-agent
```

---

## ⚡ Attack Simulation

### Step 1 — Launch Hydra from Kali

```bash
hydra -l ubuntu -P /usr/share/wordlists/rockyou.txt ssh://192.168.40.159 -t 4 -V
```

| Flag | Meaning |
|------|---------|
| `-l ubuntu` | Target username |
| `-P rockyou.txt` | Password wordlist |
| `ssh://192.168.40.159` | Target protocol and IP |
| `-t 4` | 4 parallel threads |
| `-V` | Verbose (show each attempt) |

### Step 2 — Observe in Wazuh Dashboard

Navigate to: **Wazuh Dashboard → Security Events → Search: "authentication failure"**

You will see a flood of **Level 10 alerts** similar to:

```
Rule ID   : 5710
Level     : 10
Rule Desc : Multiple authentication failures
Source IP : 192.168.40.123
User      : ubuntu
Location  : /var/log/auth.log
```

---

## 📊 Results

### Wazuh Dashboard

![Wazuh Dashboard](screenshots/wazuh-dash.png)
*Figure: Wazuh SIEM dashboard showing agent status and security event overview*

### Security Alerts — Brute Force Detected

![Wazuh Alert](screenshots/wazuh-alert.png)
*Figure: Flood of Level 10 authentication failure alerts triggered by Hydra from 192.168.40.123*

### Threat Hunting View

![Wazuh Threat Hunting](screenshots/wazuh-threat-hunting.png)
*Figure: Threat hunting panel showing attacker IP, targeted username, and timestamps*

### Key Findings

- Hydra generated **hundreds of failed SSH login attempts** within seconds
- Wazuh detected and alerted on the brute force pattern in **real time**
- The attacker's IP (`192.168.40.123`) and targeted account were clearly visible in the log detail
- Alert level escalated to **Level 10** after the threshold of failures was crossed

---

## 🧠 Skills Demonstrated

- SIEM deployment and configuration (Wazuh)
- Linux endpoint hardening and agent onboarding
- Log ingestion and enrichment (auditd, auth.log)
- Offensive tooling simulation (Hydra)
- Alert triage and log analysis
- Network segmentation with VirtualBox NAT

---

## 🚀 Future Improvements

- [ ] Add **active response** rule to automatically block the attacker's IP via `iptables`
- [ ] Simulate **Windows RDP brute force** and add a Windows 10 VM
- [ ] Integrate **Shuffle SOAR** for automated alert-to-ticket workflows
- [ ] Add **Elastic Stack** for advanced log visualisation and dashboards
- [ ] Write custom Wazuh rules for detecting credential stuffing patterns

---

## 📁 Repository Structure

```
├── README.md
└── screenshots/
    ├── network-diagram.png
    ├── wazuh-alert.png
    ├── wazuh-dash.png
    └── wazuh-threat-hunting.png
```

---

## ⚠️ Disclaimer

This lab is built for **educational purposes only**. All attacks are performed in an isolated virtual environment. Never use these techniques against systems you do not own or have explicit permission to test.

---

## 📬 Connect

Built by Gholam Haydar Rezaie — open to SOC Analyst, Blue Team, and Security Operations roles.

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-blue?logo=linkedin)](https://www.linkedin.com/in/haydar1374/)
[![GitHub](https://img.shields.io/badge/GitHub-Follow-black?logo=github)](https://github.com/hayRez)
