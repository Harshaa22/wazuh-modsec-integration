# Wazuh + ModSecurity (ModSec) Integration — Step‑by‑Step Guide

> Complete README to publish in your GitHub repo. Includes background concepts (Apache, WAF, ModSecurity, Wazuh), **why this project is useful**, detailed setup steps, architecture diagram, and dashboard creation.

---

## 📖 Project Overview

This project demonstrates how to integrate **ModSecurity (WAF)** running on an Apache webserver with **Wazuh (SIEM)**. The integration provides **real-time monitoring of web application attacks** (like SQL Injection, XSS) and shows them in the Wazuh Dashboard.

### 🔹 What is Apache?

Apache is one of the most popular open-source web servers. It serves content (HTML, PHP, etc.) to clients over HTTP/HTTPS.

### 🔹 What is a WAF?

A **Web Application Firewall (WAF)** inspects and filters HTTP/S requests to protect applications against common attacks such as SQL injection, XSS, RFI, LFI, and CSRF.

### 🔹 What is ModSecurity (ModSec)?

ModSecurity is a WAF module for Apache, Nginx, and IIS. It can operate in detection-only or blocking mode. With the **OWASP Core Rule Set (CRS)**, ModSecurity can detect/stop many known web application attacks.

### 🔹 What is Wazuh?

Wazuh is an open-source **SIEM** and security platform. It collects logs, monitors agents, applies rules, and visualizes security events. It helps in detection, compliance (PCI-DSS, HIPAA, GDPR, etc.), and incident response.

### 🔹 Why are we doing this project?

* To demonstrate **end-to-end attack detection**: from a malicious request → ModSecurity detection → Wazuh collection → Dashboard alert.
* To integrate **WAF + SIEM** for better visibility.
* To learn how to configure logging, log forwarding, and visualization.

### 🔹 Use Cases

* Monitor SQL Injection, XSS, and other web attacks in real-time.
* Centralize logs from multiple webservers into Wazuh.
* Provide visual dashboards for security analysts.
* Help organizations meet compliance requirements by logging and auditing attacks.

---

## 🏗️ Architecture

```
               [Attacker / Client]
                        |
                ┌───────────────────┐
                │  Kali VM (Agent) │
                │ ──────────────── │
                │ Apache + ModSec  │
                │  (WAF + CRS)     │
                │  modsec_audit.log│
                │ Wazuh Agent      │
                └───────────────────┘
                        |
                  Forward logs
                 (TCP 1514/1515)
                        |
                ┌───────────────────┐
                │ Ubuntu VM (Mgr)  │
                │ ──────────────── │
                │ Wazuh Manager     │
                │ Wazuh Indexer     │
                │ Wazuh Dashboard   │
                └───────────────────┘
```

* **Kali VM (Agent)** runs the webserver (Apache + ModSecurity). Its Wazuh Agent tails the ModSecurity audit log and forwards it to the Manager.
* **Ubuntu VM (Manager)** receives logs, applies detection rules, stores them in the indexer, and visualizes them in the Wazuh Dashboard.

---

## 📦 Prerequisites

* **Ubuntu VM** (Wazuh Manager + Dashboard)
* **Kali VM** (Apache + ModSecurity + Wazuh Agent)
* Internet access for package installation
* SSH or console access to both VMs
* GitHub account (to host this documentation)

---

## 📂 Files in this repo

* `README.md` — (this file) full step-by-step documentation
* `Wazuh-Modsec-Integration.pdf` — printable PDF of the guide

---

## ⚙️ Wazuh Manager (Ubuntu)

*(Installation steps… unchanged from previous version)*

---

## ⚙️ Wazuh Agent (Kali)

*(Installation steps… unchanged from previous version)*

---

## ⚙️ Apache + ModSecurity (Kali)

*(Installation steps… unchanged from previous version)*

---

## 📤 Forward ModSecurity logs to Wazuh Agent

*(Steps unchanged… includes localfile config in ossec.conf)*

---

## 🧪 Testing the pipeline

*(Attack generation, checking logs, verifying alerts… unchanged)*

---

## 📊 Build the WAF Dashboard

*(Steps unchanged… Pie, Bar, Timeline, Raw logs)*

---

## 🛠 Troubleshooting & Tips

*(Unchanged)*

---

## 🚀 Publish this repo to GitHub

*(Unchanged)*

---

## 🤝 Contributing

Contributions welcome — open an issue or PR with improvements.

## 📜 License

MIT License.

## 📬 Contact

Open an issue in this repository for questions.

---

*End of README.*
