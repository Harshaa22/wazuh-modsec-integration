# Wazuh + ModSecurity (ModSec) Integration — Step‑by‑Step Guide

> Complete README to publish in your GitHub repo. Includes **why** each command is run, full setup steps (Wazuh Manager + Agent), ModSecurity configuration, testing, and dashboard creation.

---

## Table of contents

1. [Project overview](#project-overview)
2. [Architecture](#architecture)
3. [Prerequisites](#prerequisites)
4. [Files in this repo](#files-in-this-repo)
5. [Wazuh Manager (Ubuntu) — installation & why](#wazuh-manager-ubuntu)
6. [Wazuh Agent (Kali) — installation & why](#wazuh-agent-kali)
7. [Apache + ModSecurity (Kali) — install & configure](#apache--modsecurity-kali)
8. [Forward ModSecurity logs to Wazuh Agent](#forward-modsecurity-logs)
9. [Testing the pipeline (attacks → logs → Wazuh)](#testing-the-pipeline)
10. [Build the WAF dashboard (visualizations)](#build-the-waf-dashboard)
11. [Troubleshooting & tips](#troubleshooting--tips)
12. [How to publish this repo to GitHub (quick)](#publish-to-github)
13. [Contributing / License / Contact](#contributing--license--contact)

---

## Project overview

This repository documents how to integrate **ModSecurity (WAF)** running on an Apache webserver with **Wazuh (SIEM)**. The final result:

* ModSecurity (with OWASP CRS) protects the webserver and writes audit logs in JSON.
* The Wazuh Agent (installed on the same host as the webserver) tails the ModSecurity audit log and forwards events to the Wazuh Manager.
* Wazuh Manager + Dashboard ingests events and lets you create visualizations (Top attack types, attacker IPs, timeline, raw logs).

This README is designed as a step-by-step user-facing document you can add to your GitHub repo so visitors can reproduce the project.

---

## Architecture

```
             [Internet]
                 |
             [Client]
                 |
            [Kali VM: Apache + ModSecurity]  <== WAF
                 |   (modsec_audit.log)
             Wazuh Agent (on Kali)  --(TCP 1514/1515)-->  Wazuh Manager (Ubuntu)
                                                         Wazuh Indexer + Dashboard
```

* **Kali VM**: webserver + ModSecurity (acts as Agent)
* **Ubuntu VM**: Wazuh Manager + Indexer + Dashboard

---

## Prerequisites

* Two VMs (or hosts):

  * **Manager**: Ubuntu 20.04/22.04 (Wazuh Manager + Dashboard)
  * **Agent**: Kali Linux (Apache + ModSecurity + Wazuh Agent) — this scenario uses Kali as the webserver but you can use any Linux.
* Internet access to download packages
* A GitHub account (for hosting this README & PDF)
* Basic shell/SSH access to both machines

---

## Files in this repo

* `README.md` — (this file) step-by-step guide
* `Wazuh-Modsec-Integration.pdf` — printable PDF of the guide (optional)
* (optional) `scripts/` — automation scripts (not included by default)

---

## Wazuh Manager (Ubuntu)

This installs the Manager, Indexer, and Dashboard using Wazuh's consolidated installer. Replace the IP addresses and hostnames below with your own.

### 1) Download & run the installer

```bash
curl -sO https://packages.wazuh.com/4.12/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

**Why:** the official script installs the Wazuh Manager, Wazuh Indexer (Elasticsearch-compatible), and the Wazuh Dashboard in "all-in-one" mode. This is the fastest way to get a working manager + UI.

### 2) Verify services

```bash
sudo systemctl status wazuh-manager wazuh-indexer wazuh-dashboard
```

**Why:** ensures the core services are up. If any service failed, `journalctl -xeu <service>` will show errors.

### 3) Firewall (if UFW enabled)

```bash
sudo ufw allow 1514/tcp    # agent-manager log transport
sudo ufw allow 1515/tcp    # registration/authd (optional)
sudo ufw reload
```

**Why:** allows agents to connect to the manager over the default Wazuh ports.

---

## Wazuh Agent (Kali)

Two installation methods are shown — *deb* package or repository. Use whichever you prefer.

### Option A — Install agent (deb download)

```bash
# download agent package (example path)
curl -sO https://packages.wazuh.com/4.12/wazuh-agent-4.12.0.deb
sudo dpkg -i wazuh-agent-4.12.0.deb
```

**Why:** installs the Wazuh agent binary and service on the agent host.

### Option B — Install via Wazuh apt repo (preferred for upgrades)

```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --dearmor -o /usr/share/keyrings/wazuh.gpg
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list
sudo apt update
sudo apt install -y wazuh-agent
```

**Why:** repo method keeps the agent upgradable via `apt`.

### Configure agent to point to the Manager

Edit `/var/ossec/etc/ossec.conf` and set `<server><address>` to your manager IP (example `192.168.0.8`). Example snippet:

```xml
<server>
  <address>192.168.0.8</address>
  <port>1514</port>
  <protocol>tcp</protocol>
</server>
<client>
  <name>Kali</name>
</client>
```

**Why:** this config tells the agent where to forward logs.

### Start the agent

```bash
sudo systemctl enable --now wazuh-agent
sudo systemctl status wazuh-agent
```

**Why:** ensures auto‑start and validates the agent is running.

### Register the agent with the Manager (secure key exchange)

On Manager:

```bash
sudo /var/ossec/bin/manage_agents
# A -> add agent (give name & IP)
# E -> extract key (copy the key to clipboard)
# Q -> quit
```

On Agent (Kali):

```bash
sudo /var/ossec/bin/manage_agents
# I -> import key (paste the key copied from the manager)
# W -> write changes
# Q -> quit
```

Then restart the agent:

```bash
sudo systemctl restart wazuh-agent
```

**Why:** this creates a shared key so manager trusts the agent; required for secure enrollment.

---

## Apache + ModSecurity (Kali)

We install Apache, ModSecurity module, and configure OWASP CRS and JSON audit logging.

### 1) Install packages

```bash
sudo apt update
sudo apt install -y apache2 libapache2-mod-security2 modsecurity-crs
```

**Why:** installs the web server, modsecurity module, and the CRS rule collection.

### 2) Enable ModSecurity

```bash
sudo cp /etc/modsecurity/modsecurity.conf-recommended /etc/modsecurity/modsecurity.conf
sudo sed -i 's/SecRuleEngine DetectionOnly/SecRuleEngine On/' /etc/modsecurity/modsecurity.conf
```

**Why:** copy the recommended configuration, then switch the `SecRuleEngine` to `On` so ModSecurity actively blocks malicious requests (instead of just logging).

### 3) Enable module and CRS

```bash
sudo a2enmod security2
# Ensure CRS setup file exists -- create or copy if missing
sudo mkdir -p /etc/modsecurity/crs
# Copy example (path may vary by distro); if missing, fetch from GitHub
sudo cp /usr/share/modsecurity-crs/crs-setup.conf.example /etc/modsecurity/crs/crs-setup.conf 2>/dev/null || true
# Link rules directory
sudo ln -s /usr/share/modsecurity-crs/rules /etc/modsecurity/crs/rules 2>/dev/null || true
sudo apache2ctl configtest
sudo systemctl restart apache2
```

**Why:** activates security2 module, assures CRS files are in the expected locations, validates Apache config, and restarts the server.

> **If Apache fails to restart**: `apache2ctl configtest` will show the exact error; common problems are missing `crs-setup.conf` or wrong `Include` paths. Fix by locating `crs-setup.conf.example` (often in `/usr/share/modsecurity-crs/`) and copying it to `/etc/modsecurity/crs/`.

### 4) Configure JSON audit logging (recommended)

Edit `/etc/modsecurity/modsecurity.conf` and set the audit section to:

```
SecAuditEngine On
SecAuditLogFormat JSON
SecAuditLogType Serial
SecAuditLog /var/log/apache2/modsec_audit.log
SecAuditLogParts ABDEFHIJZ
```

**Why:** JSON + Serial gives one JSON object per line in a single file — this is easiest for Wazuh to parse and index.

Create and secure the log file:

```bash
sudo touch /var/log/apache2/modsec_audit.log
sudo chown www-data:adm /var/log/apache2/modsec_audit.log
sudo chmod 640 /var/log/apache2/modsec_audit.log
```

**Why:** ensures Apache (www-data) can write the audit log, and that permissions are not overly permissive.

---

## Forward ModSecurity logs to Wazuh Agent

Tell the agent to monitor the ModSecurity audit log. On Kali edit `/var/ossec/etc/ossec.conf` and add a `<localfile>` block inside `<ossec_config>`:

```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/apache2/modsec_audit.log</location>
</localfile>
```

Then restart the agent:

```bash
sudo systemctl restart wazuh-agent
```

**Why:** the Wazuh Agent tails that file and sends log lines to the manager. `log_format`=`json` tells Wazuh to apply JSON decoders.

---

## Testing the pipeline

1. Generate test attacks on Kali (local requests):

```bash
# Simple XSS attempt
curl -s "http://localhost/?q=<script>alert(1)</script>" > /dev/null
# SQLi like payload (trick user input)
curl -s -A "sqlmap/1.7" "http://localhost/index.php?id=1'%20OR%20'1'='1" > /dev/null
```

**Why:** these payloads usually trigger CRS rules for XSS and SQL injection.

2. Check ModSecurity audit log:

```bash
sudo tail -n 20 /var/log/apache2/modsec_audit.log
```

**Why:** confirms ModSecurity logged the triggered events in JSON.

3. Check the Wazuh Agent log (Kali) to verify the file is being monitored:

```bash
sudo tail -f /var/ossec/logs/ossec.log | grep -i modsec
```

You should see a line like: `Monitoring file: /var/log/apache2/modsec_audit.log`.

4. Check the Wazuh Manager alerts (Ubuntu):

```bash
sudo tail -f /var/ossec/logs/alerts/alerts.json
```

**Why:** ensures manager received & processed the events. They will later be visible in the Dashboard.

---

## Build the WAF Dashboard (OpenSearch / Wazuh Dashboard)

Follow these steps inside the Wazuh Dashboard UI (OpenSearch Dashboards / Kibana lookalike):

### A) Discover — verify incoming fields

* Open **Discover** → choose index `wazuh-alerts-*`.
* Search for your agent: `agent.name:"Kali"`.
* Confirm fields like `rule.description`, `data.srcip`, `data.url`, `@timestamp`, `full_log` appear.

### B) Visualization 1 — Pie: Top Attack Types

* Create new visualization → Pie.
* Data source: `wazuh-alerts-*`.
* Buckets → Split slices → Aggregation: `Terms` → Field: `rule.description` → Size: 10.
* Save as `WAF - Top Attack Types`.

### C) Visualization 2 — Bar: Top Source IPs

* Create new visualization → Vertical Bar.
* X-axis → Terms → Field: `data.srcip` → Size: 10.
* Y-axis → Metric: Count.
* Save as `WAF - Top Source IPs`.

### D) Visualization 3 — Line: Attack Timeline

* Create new visualization → Line chart.
* X-axis → Date Histogram on `@timestamp`.
* Y-axis → Count.
* Save as `WAF - Attack Timeline`.

### E) Visualization 4 — Raw logs table

* In Discover, create a saved search filtered for `agent.name:"Kali" AND rule.groups:attack` and add columns: `@timestamp`, `rule.description`, `data.url`, `data.srcip`, `full_log`.
* Save search as `WAF - Raw Logs` and add it to the dashboard.

### F) Assemble dashboard

* Dashboard → Create new → Add the saved visualizations (Pie, Bar, Line, Saved search).
* Arrange layout and set an appropriate time range (Last 24 hours / Last 7 days).
* Save as `ModSecurity WAF Dashboard`.

---

## Troubleshooting & tips

**Apache failed to restart after enabling ModSecurity**

* Run: `sudo apache2ctl configtest` → the output gives the faulty config line.
* Common error: missing `crs-setup.conf` or wrong include paths. Fix by copying `crs-setup.conf.example` into `/etc/modsecurity/crs/` or adjusting `security2.conf` includes.

**Agent not forwarding logs**

* Check `/var/ossec/logs/ossec.log` on the agent — it should show `Monitoring file: /var/log/apache2/modsec_audit.log`.
* Ensure `log_format` matches the actual format: use `json` if `SecAuditLogFormat JSON` is set, otherwise use `syslog`.

**No fields in Dashboard (0 fields in Data View)**

* In OpenSearch Dashboards → Stack Management → Data Views (Index Patterns) → select `wazuh-alerts-*` → Refresh field list.
* If still empty, ensure indices exist and mapping is present on the manager (check `curl http://localhost:9200/_cat/indices?v`).

**Permissions problems writing audit log**

* Make sure `/var/log/apache2/modsec_audit.log` is owned by `www-data:adm` and has `640` permissions.

---

## Publish this repo to GitHub (quick)

1. Create a new repository on GitHub: name it `wazuh-modsec-integration`.
2. Upload this `README.md` and the PDF (`Wazuh-Modsec-Integration.pdf`) via the web UI or push with Git (see `publish-to-github` instructions in the repo root).

---

## Contributing

Contributions welcome — open an issue or a PR with improvements, scripts, or bug fixes.

## License

This guide is provided under the MIT License — adapt as you prefer.

---

## Contact

If you have questions, open an issue in this repository or contact the author.

---

*End of README — copy this file into your repo’s root as `README.md`. Visitors will see the full step‑by‑step guide when they open your repository.*
