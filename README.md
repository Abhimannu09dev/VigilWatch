# 🛡️ VigilWatch — Real-Time Intrusion Detection & Security Monitoring System

[![Python](https://img.shields.io/badge/Python-3.x-3776AB?logo=python&logoColor=white)](https://www.python.org/)
[![Flask](https://img.shields.io/badge/Flask-2.2.2-000000?logo=flask)](https://flask.palletsprojects.com/)
[![VirusTotal](https://img.shields.io/badge/VirusTotal-API-394EFF?logo=virustotal)](https://www.virustotal.com/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)

---

## 📖 Project Description

**VigilWatch** is a real-time **Intrusion Detection System (IDS)** and security event monitoring platform built with Python and Flask. It ingests security events from external systems via a REST API, analyses them against a set of configurable threat detection rules, and responds automatically to threats — blocking malicious IPs via `iptables`, locking compromised accounts via LDAP/Active Directory, quarantining malicious file uploads, and generating severity-tagged alerts across a live admin dashboard.

### Problem it solves

Enterprise environments generate thousands of security events per minute — failed logins, unusual file access, suspicious uploads, off-hours activity — and manual monitoring is too slow to prevent damage. VigilWatch automates the detection-to-response pipeline: the moment a brute-force threshold is crossed, the attacker's IP is blocked. The moment a malicious file is uploaded, it is quarantined before any user can open it. No manual intervention required.

---

## ✨ Features

- **Brute-Force Detection** — Tracks failed login attempts per IP using a rolling 1-minute window. If attempts exceed the configurable threshold (default: 10), a `Critical` alert fires and the IP is immediately blocked via `iptables`
- **Token Bucket Rate Limiting** — Each client IP is assigned a token bucket (capacity: 20 requests/minute). Requests beyond the limit return `429 Too Many Requests` and trigger a `Medium` alert
- **Unknown Location Login Detection** — Each user has a whitelist of known IP addresses. A successful login from an unlisted IP triggers a `Medium` alert
- **Multi-User IP Detection** — If the same IP authenticates as more than one user within a 1-minute window, a `Medium` alert fires (credential sharing / pivot detection)
- **Off-Hours Login Detection** — Successful logins outside configurable working hours (default: 08:00–18:00) trigger a `Low` alert
- **Privilege Escalation Detection** — Any `permission_change` event that assigns the `admin` role triggers an immediate `Critical` alert
- **Download Abuse Detection** — Users exceeding 5 downloads per minute are temporarily blocked for 60 seconds and a `Critical` alert is raised
- **Spam Email Detection** — Incoming email events are scanned for keywords (`free money`, `win prize`, `urgent`, `click here`, `lottery`). Matches trigger a `Low` alert
- **File Upload Virus Scanning** — Uploaded files are scanned in a background thread via `ThreadPoolExecutor`. Detection uses two layers: local EICAR test signature check first, then full VirusTotal API scan (15 retries × 5s polling)
- **File Quarantine** — Malicious files are written to the local `quarantine/` directory and never made available to users
- **Automatic IP Blocking** — On any `Critical` alert containing an IP address, `iptables -I INPUT -s <ip> -j DROP` is executed immediately via `subprocess`
- **LDAP/AD Account Locking** — On any `Critical` alert containing a username, the user's Active Directory account is disabled by setting `userAccountControl=514` via `ldap3`
- **Fail2Ban Integration** — Authentication failures are written to `logs/flask_auth.log` in Fail2Ban-compatible format for OS-level ban enforcement
- **System Activity Monitoring** — Accepts system-level events: financial transactions (>$1M triggers `Critical`), network anomaly detection, and bulk auth failure tracking
- **Alert Deduplication** — A 60-second suppression window prevents alert storms from the same event type
- **Dynamic Configuration** — All detection thresholds (rate limit, failed login threshold, download threshold, working hours) are live-adjustable at runtime via `POST /config` without restarting the server
- **Prepopulated Demo Logs** — On startup, `prepopulated_logs.json` is loaded to seed the dashboard with realistic demo data
- **Admin Dashboard** — A Flask-rendered HTML dashboard at `/admin` displaying all alerts (with severity), IP request logs, failed login counts, multi-user IP activity, and user download activity

---

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| **Language** | Python 3 |
| **Web Framework** | Flask 2.2.2 |
| **HTTP** | Werkzeug 2.2.2 |
| **LDAP / Active Directory** | ldap3 2.9.1 |
| **Virus Scanning** | VirusTotal REST API v3 (via `requests` 2.28.1) |
| **Cloud SDK** | boto3 1.26.0 (AWS) |
| **Concurrency** | `concurrent.futures.ThreadPoolExecutor` |
| **Firewall Integration** | `iptables` (via `subprocess`) |
| **Templating** | Jinja2 (Flask built-in) |
| **Logging** | Python `logging` module + file handler |

---

## 🏗️ Architecture Overview

```
External Systems (auth servers, file systems, email gateways)
        │
        │  POST /log          POST /system_activity    POST /upload
        ▼
┌─────────────────────────────────────────────────┐
│               Flask REST API (app.py)            │
│                                                  │
│  Token Bucket Rate Limiter (per IP)              │
│        │                                         │
│        ▼                                         │
│  Event Router (process_event)                    │
│  ├── login_attempt   → process_login_attempt     │
│  ├── permission_change → process_permission_change│
│  ├── file_access     → process_file_access       │
│  ├── email           → process_email_event       │
│  └── file_upload     → process_file_upload       │
│                                                  │
│  System Activity Router                          │
│  ├── transaction     → amount threshold check    │
│  ├── network         → IP origin check           │
│  └── auth            → failure count check       │
│                                                  │
│  send_alert(msg, severity, meta)                 │
│  └── if Critical + IP   → block_ip() (iptables)  │
│  └── if Critical + user → lock_user_account()    │
│                           (LDAP/AD)              │
└─────────────────────────────────────────────────┘
        │
        ▼
┌────────────────────────┐    ┌──────────────────────────┐
│   In-Memory Alert Store│    │  logs/flask_auth.log     │
│   (+ JSON persistence) │    │  (Fail2Ban compatible)   │
└────────────────────────┘    └──────────────────────────┘
        │
        ▼
  /admin  →  Admin Dashboard (Jinja2 HTML)
```

---

## 📁 Project Structure

```
VigilWatch/
├── app.py                      # Main Flask application (377 lines — all logic lives here)
├── requirements.txt            # Python dependencies
├── prepopulated_logs.json      # Demo data loaded on startup (alerts, IP logs, failed logins)
├── file.txt                    # Miscellaneous notes
│
├── templates/
│   ├── index.html              # Public landing page (GET /)
│   └── admin.html              # Admin dashboard (GET /admin) — alerts, logs, config
│
├── static/                     # CSS / JS / assets for the dashboard UI
│
├── logs/
│   └── flask_auth.log          # Auth failure log (auto-created) — Fail2Ban reads this
│
├── quarantine/                 # Malicious uploaded files are stored here (auto-created)
│
└── vigilwatch/                 # Python virtual environment
```

---

## ⚙️ Installation Guide

### Prerequisites

- Python 3.8 or later
- `pip`
- Linux (required for `iptables` IP blocking — the app runs on other OS but firewall actions will fail gracefully)
- A [VirusTotal API key](https://www.virustotal.com/gui/my-apikey) (free tier works)
- *(Optional)* An LDAP/Active Directory server for account locking
- *(Optional)* [Fail2Ban](https://www.fail2ban.org/) installed for OS-level IP banning

---

### 1. Clone the repository

```bash
git clone https://github.com/Abhimannu09dev/VigilWatch.git
cd VigilWatch
```

---

### 2. Create and activate a virtual environment

```bash
python3 -m venv vigilwatch
source vigilwatch/bin/activate       # Linux / macOS
vigilwatch\Scripts\activate          # Windows
```

---

### 3. Install dependencies

```bash
pip install -r requirements.txt
```

---

### 4. Configure environment variables

Create a `.env` file or export the variables directly:

```bash
export FLASK_SECRET_KEY="your-strong-random-secret-key"
export VT_API_KEY="your-virustotal-api-key"
```

Or on Windows:

```cmd
set FLASK_SECRET_KEY=your-strong-random-secret-key
set VT_API_KEY=your-virustotal-api-key
```

> **💡 VirusTotal Free Tier:** The free API allows 4 requests/minute and 500 requests/day. For high-volume scanning, consider a premium key or implement a local ClamAV fallback.

> **⚠️ `iptables` requires `sudo`:** The `block_ip()` function calls `sudo iptables`. Either run the Flask app as root (not recommended for production) or configure a `sudoers` entry that allows the app user to run `iptables` without a password.

---

### 5. (Optional) Configure known user IPs

Edit the `known_user_ips` dictionary in `app.py` to map your users to their expected IP addresses:

```python
known_user_ips = {
    "alice": ["192.168.1.100", "10.0.0.50"],
    "bob":   ["192.168.1.105"]
}
```

Any login from an IP not in this list will trigger an Unknown Location alert.

---

### 6. (Optional) Configure LDAP

Update the LDAP server address and admin credentials in `lock_user_account()`:

```python
server = ldap3.Server("ldaps://your-ldap-server.domain.local")
conn = ldap3.Connection(server, user="cn=admin,dc=domain,dc=local", password="yourpassword")
```

> **💡 Test mode:** If the LDAP server is unreachable, `lock_user_account()` logs a warning and continues — it does not crash the app. This is safe for local development.

---

## ▶️ How to Run the Project Locally

```bash
python app.py
```

The server starts at:

```
http://127.0.0.1:5000
```

Visit **http://127.0.0.1:5000/admin** for the live security dashboard.

> **⚠️ Development mode only:** `app.run(debug=True)` is set in `app.py`. For production, use a WSGI server like `gunicorn`:
>
> ```bash
> pip install gunicorn
> gunicorn -w 4 -b 0.0.0.0:8000 app:app
> ```

---

## 🔐 Environment Variables

| Variable | Required | Description |
|---|---|---|
| `FLASK_SECRET_KEY` | Yes | Secret key for Flask session signing. Use a long random string in production. |
| `VT_API_KEY` | Yes | VirusTotal API v3 key for malware scanning. Get one at [virustotal.com](https://www.virustotal.com/gui/my-apikey). |

---

## 📡 API Endpoints

### Public

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/` | Landing page (Jinja2 HTML) |
| `GET` | `/admin` | Admin dashboard — alerts, IP logs, config |
| `GET` | `/alerts` | Returns all alerts as JSON |
| `GET` | `/config` | Returns current detection thresholds as JSON |
| `POST` | `/config` | Update one or more thresholds at runtime (no restart needed) |
| `GET` | `/log_analysis` | Returns a count of system activity events grouped by type |

### Event Ingestion

| Method | Endpoint | Body | Description |
|---|---|---|---|
| `POST` | `/log` | JSON event object | Submit a security event for analysis (rate-limited per IP) |
| `POST` | `/system_activity` | JSON activity object | Submit a system-level event (transaction, network, auth) |
| `POST` | `/upload` | `multipart/form-data` | Upload a file for virus scanning (field: `upload_file`, param: `user`) |

---

### Event Schema — `POST /log`

All events must include a `type` field. Supported types:

**`login_attempt`**
```json
{
  "type": "login_attempt",
  "user": "alice",
  "ip": "192.168.1.200",
  "timestamp": "2024-01-15T22:30:00",
  "success": false
}
```

**`permission_change`**
```json
{
  "type": "permission_change",
  "user": "bob",
  "new_role": "admin"
}
```

**`file_access`**
```json
{
  "type": "file_access",
  "user": "alice",
  "access_type": "download",
  "timestamp": "2024-01-15T14:00:00"
}
```

**`email`**
```json
{
  "type": "email",
  "user": "bob",
  "subject": "Urgent: Click here to win prize",
  "body": "You have won a lottery!"
}
```

**`file_upload`**
```json
{
  "type": "file_upload",
  "user": "alice",
  "file": "report.pdf",
  "content": "<base64 or raw text content>"
}
```

---

### System Activity Schema — `POST /system_activity`

```json
{ "type": "transaction", "amount": 1500000 }
{ "type": "network",     "ip": "203.0.113.45" }
{ "type": "auth",        "failures": 8 }
```

---

## 🧩 How the System Works

### Event Detection Pipeline

```
POST /log  { "type": "login_attempt", "ip": "1.2.3.4", "success": false, ... }
        │
        ▼
Token bucket check for 1.2.3.4
  └── Consumed? → continue
  └── Exhausted? → 429 + Medium alert
        │
        ▼
process_event(event)
  └── routes to process_login_attempt(event)
        │
        ├── Success from unknown IP?   → Medium alert
        ├── Failed login?
        │   ├── Log to flask_auth.log  (Fail2Ban)
        │   ├── Append to failed_logins[ip]
        │   ├── Prune entries > 1 min old
        │   └── Count > threshold?    → Critical alert → block_ip()
        ├── Same IP, multiple users?   → Medium alert
        └── Off-hours success?         → Low alert
```

### Automatic Response on Critical Alerts

```
send_alert("Brute-force attack", severity="Critical", meta={"ip": "1.2.3.4"})
        │
        ├── Deduplicate check (60s window) — is this a repeat? Skip if yes
        ├── Append to in-memory alerts[]
        ├── Persist to prepopulated_logs.json
        └── severity == "Critical"?
            ├── meta.ip present?   → block_ip("1.2.3.4")
            │   └── subprocess: sudo iptables -I INPUT -s 1.2.3.4 -j DROP
            └── meta.user present? → lock_user_account("alice")
                └── ldap3: set userAccountControl=514 (disabled)
```

### Virus Scan Pipeline

```
POST /upload  (multipart, field: upload_file)
        │
        ▼
executor.submit(real_virus_scan, file_content)   ← background thread
        │
        ├── EICAR signature in content?
        │   └── Yes → return True (detected)
        │
        └── POST https://www.virustotal.com/api/v3/files
            └── Poll /analyses/{id} every 5s (up to 15 attempts)
                └── status == "completed"?
                    └── stats.malicious > 0 → return True
        │
        ▼
Malicious detected?
  ├── Yes → quarantine_file(name, content)   → written to quarantine/
  │         send_alert("Virus in ...", "Critical", {"user": user})
  │         return 200 { "status": "alert", "message": "File quarantined" }
  └── No  → return 200 { "status": "success", "message": "File clean" }
```

### Dynamic Configuration

The detection thresholds are stored in a mutable `config` dict and can be updated at runtime without restarting the server:

```bash
# Example: lower the brute-force threshold to 5 attempts
curl -X POST http://localhost:5000/config \
  -H "Content-Type: application/json" \
  -d '{"failed_login_threshold": 5, "working_hours": [9, 17]}'
```

---

## 🚨 Alert Severity Levels

| Severity | Triggers | Automatic Response |
|---|---|---|
| **Critical** | Brute-force threshold exceeded, privilege escalation to admin, high download rate, malicious file upload, bulk auth failures, large transaction (>$1M) | Block IP via `iptables` + Lock user account via LDAP |
| **Medium** | Rate limit exceeded, unknown-location login, multi-user IP, network anomaly from external IP | Alert logged only |
| **Low** | Off-hours login, suspicious email keywords, normal network activity | Alert logged only |

---

## ⚙️ Configuration Reference

All values are adjustable at runtime via `POST /config`:

| Key | Default | Description |
|---|---|---|
| `rate_limit_requests` | `20` | Maximum requests per IP per minute (token bucket capacity) |
| `failed_login_threshold` | `10` | Failed logins per IP per minute before brute-force alert fires |
| `download_threshold` | `5` | Downloads per user per minute before download-abuse alert fires |
| `alert_dedup_interval` | `60` | Seconds before the same alert message can fire again |
| `working_hours` | `[8, 18]` | Start and end hour (24h) for normal working hours |

---

## 📂 Log Files

| File | Location | Purpose |
|---|---|---|
| `flask_auth.log` | `logs/flask_auth.log` | Auth failure log in Fail2Ban-readable format: `LOGIN_FAILED ip=... user=...` |
| `prepopulated_logs.json` | Project root | Persisted alerts loaded on startup for dashboard demo data |

### Setting up Fail2Ban (optional)

Add a custom jail to `/etc/fail2ban/jail.local`:

```ini
[vigilwatch-auth]
enabled  = true
port     = http,https
filter   = vigilwatch-auth
logpath  = /path/to/VigilWatch/logs/flask_auth.log
maxretry = 10
bantime  = 3600
```

Create `/etc/fail2ban/filter.d/vigilwatch-auth.conf`:

```ini
[Definition]
failregex = LOGIN_FAILED ip=<HOST>
ignoreregex =
```

---

## 📸 Screenshots / Demo

> Screenshots and a demo link will be added here.

| View | Description |
|---|---|
| `[Admin Dashboard]` | Live alert feed with severity badges, IP request log, failed login heatmap |
| `[Alert Feed]` | Real-time Critical / Medium / Low alert cards |
| `[Config Panel]` | Live threshold editor — no server restart needed |

---

## 🚀 Future Improvements

- [ ] Persistent database (SQLite / PostgreSQL) instead of in-memory alert storage
- [ ] WebSocket or SSE for real-time dashboard updates without page refresh
- [ ] Email / Slack / PagerDuty alert notifications for Critical events
- [ ] ClamAV local fallback for virus scanning (remove VirusTotal dependency)
- [ ] Geo-IP enrichment on unknown location alerts
- [ ] Role-based admin dashboard authentication
- [ ] Docker + Docker Compose setup for easy deployment
- [ ] Unit tests for each detection rule (pytest)

---

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/your-feature`
3. Make your changes
4. Ensure the app starts: `python app.py`
5. Commit: `git commit -m "feat: add your feature"`
6. Open a Pull Request

---

## 📄 License

This project is licensed under the **MIT License** — see the [LICENSE](./LICENSE) file for details.

---

## 👨‍💻 Authors

**Abhimannu** — [GitHub](https://github.com/Abhimannu09dev) · [LinkedIn](https://www.linkedin.com/in/abhimannu-singh-kunwar-5a9096268/)

**Krishna** — [GitHub](https://github.com/krishna09-dev) 

> *Built with Python and Flask as a cybersecurity project — an intrusion detection system with automated threat response.*
