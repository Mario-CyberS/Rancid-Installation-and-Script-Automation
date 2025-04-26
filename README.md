# Wazuh Rancid Installation & Script Automation for ASA Backups  
This project explains how to create a **Rancid user** on the Wazuh server, set up **automated daily backups** of Cisco ASA running configurations using **Rancid Tool**, **Expect scripting**, **Bash scripting**, and **crontab scheduling**.

---

## 🎯 Objective  
To automate the secure backup of Cisco ASA device configurations to a Wazuh server, implement monthly and yearly backup retention policies, and execute easy export of backups to local machines when needed. 

---

## 🔍 Why Automate ASA Config Backups?  
Backing up firewall configurations daily is critical for:
- Disaster recovery readiness
- Compliance requirements (e.g., CJIS retention)
- Preventing manual error or loss of configs
- Centralized, version-controlled configuration management

Automation ensures that backups happen consistently without relying on manual intervention.

---

## 📚 Skills Learned  
- Setting up SSH public key authentication for automated access  
- Writing Expect scripts to interact with ASA CLI  
- Bash scripting for file management and retention enforcement  
- Using `crontab` to schedule daily backups
- Exporting configuration backups to a local machine using `scp` or `pscp`, depending on OS

---

## 🛠️ Tools Used  
<div>
  <a href="https://shrubbery.net/rancid/" target="_blank"><img src="https://img.shields.io/badge/-Rancid-800000?&style=for-the-badge&logoColor=white" />
  <img src="https://img.shields.io/badge/-Expect-333333?&style=for-the-badge&logo=Linux&logoColor=white" />
  <img src="https://img.shields.io/badge/-Bash_Scripting-4EAA25?&style=for-the-badge&logo=GNU-Bash&logoColor=white" />
</div>

---

## 📝 Deployment Steps

### 1. Set Up the Rancid User on Wazuh Server
```bash
sudo useradd -m -s /bin/bash rancid
sudo passwd rancid
sudo usermod -aG wheel rancid
```
Verify group membership:
```bash
groups rancid
```

### 2. Configure SSH Key Authentication





































