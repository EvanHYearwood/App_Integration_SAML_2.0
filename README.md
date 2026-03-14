# Custom App Integration with Okta (SAML 2.0)
**Python Flask · SAML 2.0 · Okta · Active Directory · Windows Server**

A hands-on project demonstrating how to build and deploy custom SAML 2.0 Service Provider (SP) applications integrated with Okta as the Identity Provider (IdP). Hosted on a Windows Server domain controller running an on-premises Active Directory environment.

> 🤖 **AI Disclosure:** The three Flask applications were built with the assistance of Claude (Anthropic AI). All infrastructure decisions, DNS configuration, Okta setup, troubleshooting, and deployment were executed hands-on. AI was used as a coding accelerator, the same way engineers use it in real-world environments.

---

## 📺 Demo Walkthrough

- ### [Watch the Demo Video 🚀](#) *(Coming Soon)*

---

## Table of Contents
1. [Project Overview](#project-overview)
2. [Environment](#environment)
3. [How SAML Works](#how-saml-works)
4. [What Was Built](#what-was-built)
5. [Phase 1 — DNS Setup](#phase-1--dns-setup)
6. [Phase 2 — Okta SAML App Configuration](#phase-2--okta-saml-app-configuration)
7. [Phase 3 — App Configuration (settings.json)](#phase-3--app-configuration-settingsjson)
8. [Phase 4 — Deployment](#phase-4--deployment)
9. [Errors & Troubleshooting](#errors--troubleshooting)
10. [Key Concepts Demonstrated](#key-concepts-demonstrated)
11. [Tools & Technologies](#tools--technologies)
12. [Project Status](#project-status)

---

## Project Overview

The goal of this project was to simulate what it looks like when an organization integrates internal or third-party applications with a cloud Identity Provider using SAML 2.0 — without needing access to the real paid SaaS tools.

Three custom web applications were built to act as SAML Service Providers, each themed to represent a common enterprise application. Okta serves as the Identity Provider, and all users are sourced from an on-premises Active Directory domain (`yearwood.local`).

When a user visits one of the apps and clicks **Sign in with Okta SSO**, the full SAML authentication flow executes — the browser is redirected to Okta, the user authenticates, Okta sends a signed SAML assertion back to the app, the app validates it, and a session is created.

---

## Environment

| Component | Detail |
|-----------|--------|
| Domain | yearwood.local |
| Domain Controller | Windows Server 2025 (on-prem) |
| Identity Provider | Okta (trial org) |
| Directory | Active Directory synced to Okta via AD Agent |
| App Runtime | Python 3.11 / Flask |
| Protocol | SAML 2.0 |
| Transport | HTTP (homelab — no TLS) |

---

## How SAML Works

```
1. User visits app (Service Provider)
         ↓
2. App redirects browser to Okta (Identity Provider) with SAML Request
         ↓
3. User authenticates at Okta
         ↓
4. Okta sends signed SAML Assertion back to app's ACS URL via browser POST
         ↓
5. App validates the assertion using Okta's X.509 certificate
         ↓
6. Session created — user is logged in
```


**Key terms:**
| Term | Meaning |
|------|---------|
| SP (Service Provider) | Your app — the resource being protected |
| IdP (Identity Provider) | Okta — the system that verifies identity |
| ACS URL | The endpoint on your SP that receives the SAML assertion from Okta |
| Entity ID | A unique URI that identifies your SP to Okta |
| X.509 Certificate | Okta's public cert — used by the SP to verify the assertion was signed by Okta |
| SAML Assertion | The signed XML document Okta sends back confirming the user's identity |

---

## What Was Built

Three Python/Flask apps, each acting as an independent SAML SP:

| App | Simulates | Hostname | Port |
|-----|-----------|----------|------|
| Slack SP | Slack workspace | slack.yearwood.local | 5001 |
| GWorkspace SP | Google Workspace | gworkspace.yearwood.local | 5002 |
| SharePoint SP | Microsoft SharePoint | sharepoint.yearwood.local | 5003 |

Each app includes:
- A themed **login page** with an "Sign in with Okta SSO" button
- A **`/login`** route that initiates the SP-side SAML redirect to Okta
- A **`/saml/acs`** endpoint (ACS URL) that receives and validates Okta's SAML assertion
- A **`/saml/metadata`** endpoint that exposes the SP's metadata XML for Okta registration
- A **`/logout`** route that clears the local session
- A themed **dashboard** showing the authenticated user's identity pulled from the SAML assertion


### Folder Structure

```
saml-apps/
├── INSTALL.bat               ← installs dependencies, creates saml/ folders
├── START_ALL.bat             ← launches all 3 apps
├── requirements.txt
├── settings.json.template    ← base config template
├── slack-app/
│   ├── app.py
│   ├── saml_utils.py
│   ├── saml/
│   │   └── settings.json     ← Okta IdP values go here
│   └── templates/
│       ├── login.html
│       └── dashboard.html
├── gworkspace-app/
│   └── ...
└── sharepoint-app/
    └── ...
```

---

## Phase 1 — DNS Setup

Before deploying the apps, three DNS A records were added in **Windows Server DNS Manager** under the `yearwood.local` forward lookup zone:

| Hostname | Type | Points To |
|----------|------|-----------|
| slack | A | 192.168.64.5 (DC IP) |
| gworkspace | A | 192.168.64.5 |
| sharepoint | A | 192.168.64.5 |

This allows any machine joined to `yearwood.local` to resolve `slack.yearwood.local` to the domain controller where the apps are running.

**Steps in DNS Manager:**
1. Server Manager → Tools → DNS
2. Expand Forward Lookup Zones → `yearwood.local`
3. Right-click → New Host (A record)
4. Add each hostname pointing to the DC's IP
5. Repeat for all three

---

## Phase 2 — Okta SAML App Configuration

Three SAML 2.0 applications were created in the Okta Admin Console — one per custom app.

**Steps for each app:**
1. Okta Admin Console → Applications → **Create App Integration**
2. Select **SAML 2.0** → Next
3. Fill in the general settings (app name) → Next
4. Configure SAML settings:

| Field | Slack Example |
|-------|--------------|
| Single Sign On URL (ACS) | `http://slack.yearwood.local:5001/saml/acs` |
| Audience URI (Entity ID) | `http://slack.yearwood.local:5001/saml/metadata` |
| Name ID Format | EmailAddress |

5. Finish → go to **Sign On tab** → click **View SAML Setup Instructions**

The setup instructions page provides the three values needed for `settings.json`:
- **Identity Provider Single Sign-On URL** → goes into `idp.singleSignOnService.url`
- **Identity Provider Issuer** → goes into `idp.entityId`
- **X.509 Certificate** → goes into `idp.x509cert` (remove BEGIN/END header lines, paste as one continuous string)

> ⚠️ After creating each app, go to the **Assignments tab** and assign the relevant Okta groups — otherwise Okta will block the login even if SAML is configured correctly.

---

## Phase 3 — App Configuration (settings.json)

Each app has its own `saml/settings.json` file. The structure has two sections:

- **`sp`** — YOUR app's local addresses (yearwood.local URLs)
- **`idp`** — Okta's addresses and certificate (from Okta's setup instructions)

```json
{
  "strict": false,
  "debug": true,
  "sp": {
    "entityId": "http://slack.yearwood.local:5001/saml/metadata",
    "assertionConsumerService": {
      "url": "http://slack.yearwood.local:5001/saml/acs",
      "binding": "urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST"
    },
    "singleLogoutService": {
      "url": "http://slack.yearwood.local:5001/saml/sls",
      "binding": "urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect"
    },
    "NameIDFormat": "urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress",
    "x509cert": "",
    "privateKey": ""
  },
  "idp": {
    "entityId": "http://www.okta.com/<okta-app-id>",
    "singleSignOnService": {
      "url": "https://<your-okta-tenant>.okta.com/app/<app-path>/sso/saml",
      "binding": "urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect"
    },
    "singleLogoutService": {
      "url": "https://<your-okta-tenant>.okta.com/app/<app-path>/sso/saml",
      "binding": "urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect"
    },
    "x509cert": "<full-certificate-as-one-line-no-headers>"
  }
}
```

> ⚠️ **Common Mistake:** The `sp.entityId` must be your **local** metadata URL — not Okta's issuer URL. The `idp.entityId` comes from Okta's "Identity Provider Issuer" field. Swapping these is the most frequent configuration error.

---

## Phase 4 — Deployment

### Prerequisites
- Python 3.11+ installed with **"Add Python to PATH"** checked during install
- Visual C++ Build Tools (required by `python3-saml` on Windows)

### Install
```batch
INSTALL.bat
```
Installs all pip dependencies and creates the `saml/` config directories inside each app folder.

### Launch
```batch
START_ALL.bat
```
Opens three terminal windows, one per app:

```
slack.yearwood.local:5001       → Slack SP
gworkspace.yearwood.local:5002  → GWorkspace SP
sharepoint.yearwood.local:5003  → SharePoint SP
```

Each terminal will show:
```
* Serving Flask app 'app'
* Running on http://0.0.0.0:500X
* Running on http://192.168.64.5:500X
```
This means the app is live and listening. The red "development server" warning from Flask is expected and can be ignored for a homelab.

---

## Errors & Troubleshooting

Real errors encountered during deployment — documented because troubleshooting is the actual skill.

---

**Error 1: Python not found**
```
Python was not found; run without arguments to install from the Microsoft Store
```
**Cause:** Python was not installed, or was installed without adding it to the system PATH.

**Fix:** Reinstall Python 3.11 from python.org. On the first screen of the installer, check **"Add Python to PATH"** before clicking Install. Close and reopen all terminal windows after installing.

---

**Error 2: settings.json not found — path resolves with `..`**
```
Settings file not found: sharepoint-app\..\saml\settings.json
```
**Cause:** Python resolved `__file__` relative to the working directory rather than the script's actual location, causing it to look one folder up.

**Fix:** Updated `saml_utils.py`:
```python
# Before
settings_path = os.path.join(os.path.dirname(__file__), "saml")

# After
settings_path = os.path.join(os.path.dirname(os.path.abspath(__file__)), "saml")
```

---

**Error 3: saml_utils.py loading from parent directory**
```
sharepoint-app\..\saml_utils.py
```
**Cause:** `sys.path.insert` in `app.py` was pointing one directory level up instead of the app's own folder.

**Fix:** Updated `app.py`:
```python
# Before
sys.path.insert(0, os.path.join(os.path.dirname(__file__), '..'))

# After
sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))
```

---

## Key Concepts Demonstrated

- **SAML 2.0 SP-initiated SSO flow** — full redirect, assertion, and session lifecycle
- **Service Provider vs Identity Provider** configuration — understanding which URLs belong where
- **Okta SAML app setup** — ACS URL, Entity ID, X.509 certificate, and metadata
- **DNS A record configuration** in Windows Server for custom subdomains
- **Active Directory + Okta integration** as the user source for SSO
- **Python path resolution on Windows** — a real deployment issue and its fix
- **AI-assisted development** — using Claude to generate boilerplate, then deploying, debugging, and owning the result

---

## Tools & Technologies

| Tool | Purpose |
|------|---------|
| Python 3.11 / Flask | SAML SP application runtime |
| python3-saml | SAML 2.0 Service Provider library (OneLogin) |
| Okta | Identity Provider (IdP) |
| Okta AD Agent | Syncs Active Directory users to Okta |
| Windows Server DNS Manager | Subdomain A records for app hostnames |
| Active Directory | User source — synced to Okta |
| Notepad++ | Editing JSON config files |
| Claude (Anthropic) | AI-assisted Flask app code generation |

---

## Project Status

✅ DNS A records configured for all 3 subdomains  
✅ Okta SAML apps created and configured  
✅ Custom SP apps built and deployed on Windows Server DC  
✅ settings.json configured per app with Okta IdP values  
✅ All 3 apps running and accessible via yearwood.local subdomains  
✅ SAML SSO flow functional — Okta redirects and assertions working  

⬜ Add HTTPS with self-signed certificates  
⬜ Test SSO login end-to-end with domain users  
⬜ Add negative access testing (user not assigned to app)  

---

*Custom SAML App Integration — Built as a portfolio project demonstrating hands-on SAML 2.0 SSO configuration with Okta and Active Directory.*
