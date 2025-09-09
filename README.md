# Wazuh + ModSecurity (WAF) Integration for Apache

## 📌 Project Overview

This project demonstrates the integration of **Apache + ModSecurity (WAF)** with **Wazuh (SIEM)**. The goal is to capture ModSecurity audit logs from an Apache web server and forward them to the Wazuh Manager via the Wazuh Agent. The logs are then analyzed, visualized, and monitored in the Wazuh Dashboard.

This setup provides visibility into real-time web application attacks (SQL Injection, XSS, etc.) and centralizes WAF logs into a SIEM for enhanced security monitoring.

---

## 🔑 Key Components

### 1. Apache

Apache is one of the most widely used open-source web servers. It handles client HTTP requests and serves web content such as HTML, PHP, and static files.

### 2. WAF (Web Application Firewall)

A WAF inspects incoming HTTP(S) traffic and blocks malicious requests before they reach the application. It protects web applications against common attacks such as SQL injection, Cross-Site Scripting (XSS), and Remote Code Execution (RCE).

### 3. ModSecurity (ModSec)

ModSecurity is an open-source WAF module for Apache. It can run in **DetectionOnly** (logs attacks) or **On** (blocks attacks) mode. It supports rulesets like **OWASP CRS (Core Rule Set)** to detect and prevent web attacks.

### 4. Wazuh

Wazuh is an open-source SIEM and security monitoring platform. It collects logs from agents, correlates security events, and provides dashboards, alerting, and visualization features.

### 5. ModSecurity Audit Logs

ModSecurity generates detailed logs containing information about malicious requests, matched rules, attacker IPs, and targeted URLs. These logs are forwarded to Wazuh for analysis and monitoring.

---

## 🎯 Why this Project?

* Centralize **WAF security events** in a SIEM for better visibility.
* Detect, analyze, and visualize web attacks.
* Build dashboards that show:

  * Top attacker IPs
  * Most attacked URLs
  * Attack types (SQLi, XSS, etc.)
  * Alerts per agent/server
* Improve real-world knowledge of WAF + SIEM integration.

---

## 🏗️ Architecture Diagram

### Mermaid Diagram

```mermaid
flowchart LR
  A[Internet / Attacker] --> B[Apache + ModSecurity (Kali / Agent)]
  B --> C[/var/log/modsec_audit.log - JSON/serial/]
  C --> D[Wazuh Agent]
  D --> E[Wazuh Manager / Indexer]
  E --> F[Wazuh Dashboard]
```

### ASCII Diagram

```
[Internet] --> [Apache + ModSecurity (Agent host)]
                      |
                      v
         /var/log/modsec_audit.log  (JSON)
                      |
                      v
               [Wazuh Agent]
                      |
                      v
               [Wazuh Manager] <---> [Wazuh Indexer]
                      |
                      v
               [Wazuh Dashboard]
```

---

## ⚙️ Step-by-Step Setup

### 1. Wazuh Manager (Ubuntu)

1. Download and install Wazuh (Manager, Indexer, Dashboard):

   ```bash
   sudo bash ./wazuh-install.sh -a
   ```
2. Verify services:

   ```bash
   sudo systemctl status wazuh-manager
   sudo systemctl status wazuh-indexer
   sudo systemctl status wazuh-dashboard
   ```
3. Add an agent using:

   ```bash
   sudo /var/ossec/bin/manage_agents
   ```

   Generate a key and use it for the agent.

---

### 2. Agent (Kali Linux) - Apache + ModSecurity

1. Install Apache + ModSecurity:

   ```bash
   sudo apt update
   sudo apt install -y apache2 libapache2-mod-security2 modsecurity-crs
   ```
2. Enable ModSecurity:

   ```bash
   sudo a2enmod security2
   sudo cp /etc/modsecurity/modsecurity.conf-recommended /etc/modsecurity/modsecurity.conf
   ```

   Edit `/etc/modsecurity/modsecurity.conf` and set:

   ```
   SecRuleEngine On
   ```
3. Configure Audit Logging (JSON format):

   ```
   SecAuditLog /var/log/apache2/modsec_audit.log
   SecAuditLogFormat JSON
   SecAuditLogType Serial
   ```

   Fix permissions:

   ```bash
   sudo touch /var/log/apache2/modsec_audit.log
   sudo chown www-data:www-data /var/log/apache2/modsec_audit.log
   sudo chmod 640 /var/log/apache2/modsec_audit.log
   sudo systemctl restart apache2
   ```

---

### 3. Wazuh Agent (Kali)

1. Install and register agent:

   ```bash
   sudo apt install -y wazuh-agent
   sudo /var/ossec/bin/agent-auth -m <MANAGER_IP>
   ```
2. Configure agent to monitor ModSecurity logs (`/var/ossec/etc/ossec.conf`):

   ```xml
   <localfile>
     <log_format>json</log_format>
     <location>/var/log/apache2/modsec_audit.log</location>
   </localfile>
   ```
3. Restart agent:

   ```bash
   sudo systemctl restart wazuh-agent
   ```

---

### 4. Dashboard & Visualization

* Open Wazuh Dashboard → Discover → Search logs from your agent.
* Create visualizations:

  * Attacks by rule description
  * Top attacker IPs
  * Top targeted URLs
  * Alerts per agent
* Save and combine into a dashboard.

---

## 🧪 Testing

1. Trigger XSS:

   ```bash
   curl "http://localhost/?q=<script>alert(1)</script>"
   ```
2. Trigger SQL Injection:

   ```bash
   curl "http://localhost/index.php?id=1' OR '1'='1"
   ```
3. Check logs:

   ```bash
   tail -f /var/log/apache2/modsec_audit.log
   tail -f /var/ossec/logs/ossec.log
   ```
4. Verify events appear in Wazuh Dashboard.

---

## 🛠️ Troubleshooting

* **No logs in Wazuh:** check `ossec.conf` paths, restart agent.
* **ModSecurity not blocking:** ensure `SecRuleEngine On` and OWASP CRS rules are enabled.
* **Permission denied on audit log:** fix file ownership to `www-data:www-data`.
* **Agent disconnected after IP change:** re-run `agent-auth -m <MANAGER_IP>`.

---

## 📜 Notes

* Use this project in a **lab/test environment only**.
* Logs may contain sensitive data — secure log files and dashboard access.
* License: MIT (you can choose your preferred license).

---

## 🙌 Acknowledgements

This project was developed as a practical integration of **Apache + ModSecurity WAF** with **Wazuh SIEM**, based on lab testing and documented results.
