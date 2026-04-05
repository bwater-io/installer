# Breakwater Installation UX Expectations

## User Journey Map

This document defines what the customer sees and experiences at every stage.
Each step has timing expectations, visual descriptions, and test coverage.

---

## Phase 1: Pre-Installation (Customer receives credentials)

**What happens:** Sales provides the customer with:
1. Install password (controls image access)
2. 30-day activation code (RSA-signed, machine-bound on first use)

**Customer expectation:** Two strings, received via secure channel (email/portal).

---

## Phase 2: Installation (3-5 minutes)

### Step 2.1: Run the installer

**Command:**
```bash
curl -fsSL https://install.bwater.io | sudo bash
```

**What user sees:**
```
  ____                 _               _
 | __ ) _ __ ___  __ _| | ___      __ | |_ ___ _ __
 |  _ \| '__/ _ \/ _` | |/ \ \ /\ / / | __/ _ \ '__|
 | |_) | | |  __/ (_| |   < \ V  V /  | ||  __/ |
 |____/|_|  \___|\\__,_|_|\_\ \_/\_/    \__\___|_|

 Operational Security Platform — Installer
```

**Timing:** Instant (<1s for banner display)
**Test:** `bootstrap.sh` banner output check

### Step 2.2: Four prompts

| Prompt | Input Type | Validation | Failure Message |
|--------|-----------|------------|-----------------|
| Install password | Hidden (read -s) | Server-side validation | "Validation failed. Check your install password." |
| Admin email | Visible | Basic format check | (none — accepted as-is) |
| Admin password | Hidden (read -s + confirm) | Min 12 chars, confirmation match | "Must be at least 12 characters." / "Passwords don't match." |
| Activation code | Hidden (read -s) | Server-side validation | "Validation failed. Check your activation code." |

**Timing expectation:** User interaction only (30-60s typical)
**Test:** `install-cli.test.js` validates prompt flow

If the customer pre-seeds `BREAKWATER_BOOTSTRAP_*` environment variables, the
bootstrap skips the interactive prompts and proceeds directly to validation.

### Step 2.3: Server validation

**What user sees:**
```
[breakwater] Validating credentials...
[breakwater] Credentials validated.
```

**What happens behind the scenes:**
1. POST to `license.bwater.io/api/v1/install/validate`
2. Server verifies install password
3. Server verifies activation code signature
4. Server returns: installer URL, compose URL, short-lived registry token
5. Server returns the RSA public key used for offline runtime validation

**Timing:** 1-3s (network round-trip)
**Failure:** Clear error with contact information

### Step 2.4: Automated setup

**What user sees (sequentially):**
```
[breakwater] Running pre-flight checks...
[breakwater] Pre-flight checks passed.
[breakwater] Setting up /opt/breakwater...
[breakwater] Writing offline license and public key...
[breakwater] Choosing TLS mode...
[breakwater] Generating cryptographic secrets...
[breakwater] Writing configuration...
[breakwater] Initializing 30-day trial...
[breakwater] Trial expires: 2026-05-04T00:00:00Z
[breakwater] Pulling Breakwater images (this may take a few minutes)...
  ✓ api:latest pulled
  ✓ dashboard:latest pulled
  ✓ postgres:15-alpine pulled
  ✓ redis:7-alpine pulled
[breakwater] Starting Breakwater...
[breakwater] Waiting for services to become healthy...
  API: waiting... (12s)
[breakwater] API is healthy.
[breakwater] Dashboard is ready.
[breakwater] Creating admin account...
[breakwater] Admin account created.
[breakwater] Installing breakwater CLI...
```

**Timing breakdown:**

| Step | Expected Duration | Notes |
|------|------------------|-------|
| Pre-flight checks | <2s | Docker, disk, memory checks |
| TLS planning | <2s | Hostname detection, ACME/local TLS decision |
| Secret generation | <1s | openssl rand |
| Config write | <1s | .env + compose file |
| Trial init | <1s | Write trial.json |
| Image pull | 60-180s | Depends on bandwidth (~2GB total) |
| Stack start | 5-15s | Docker compose up |
| Health wait | 10-30s | API + Dashboard startup |
| Admin creation | 1-2s | API call |

**Total automated phase: 90-240s (1.5-4 minutes)**

### Step 2.5: Success screen

```
============================================================
 Breakwater installed successfully!
============================================================

  Dashboard:  https://scanner.yourcompany.com/
  API:        http://scanner.yourcompany.com:8000
  Admin:      admin@yourcompany.com

  Trial expires: 2026-05-04T00:00:00Z (30 days)
  TLS:        automatic public certificate via Caddy/ACME
              or local appliance certificate with fingerprint

  Useful commands:
    breakwater status           Show service status and trial info
    breakwater logs             Tail service logs
    breakwater stop / start     Stop or start services
    breakwater update           Pull latest images and restart
    sudo breakwater license renew   Enter a new extension code
    sudo breakwater uninstall       Remove Breakwater
```

**Test:** `install-ux.test.js` test 1-2 verify API + trial are healthy

---

## Phase 3: First Login (30 seconds)

### Step 3.1: Open dashboard URL

**What user sees:** Login page with Breakwater branding over HTTPS.

```
┌─────────────────────────────────────────────────┐
│                                                 │
│            [Breakwater Logo]                    │
│            Sign in to Breakwater                │
│                                                 │
│   ┌─────────────────────────────────────────┐   │
│   │ Email address                           │   │
│   └─────────────────────────────────────────┘   │
│   ┌─────────────────────────────────────────┐   │
│   │ Password                                │   │
│   └─────────────────────────────────────────┘   │
│                                                 │
│   ┌─────────────────────────────────────────┐   │
│   │              Sign In                    │   │
│   └─────────────────────────────────────────┘   │
│                                                 │
└─────────────────────────────────────────────────┘
```

**Timing:** Page load <3s
**Screenshot:** `screenshots/installation/01-login-page.png`
**Test:** `install-ux.test.js` test 3

### Step 3.2: Enter credentials and sign in

**What user does:** Types admin email + password, clicks Sign In.
**What user sees:** Redirect to main dashboard.

**Timing:** Login → Dashboard <5s
**Screenshot:** `screenshots/installation/04-dashboard-after-login.png`
**Test:** `install-ux.test.js` test 4

### Step 3.3: Main dashboard

**What user sees:** Navigation sidebar, scan overview, device summary.
The dashboard shows the trial status in the sidebar/header:

```
Trial: 30 days remaining
```

**Screenshot:** `screenshots/installation/05-dashboard-main.png`
**Test:** `install-ux.test.js` test 4

---

## Phase 4: Daily Usage (30 days)

### Trial warning (7 days before expiry)

When `days_remaining <= 7`, the API adds a header to every response:

```
X-Trial-Days-Remaining: 5
```

The dashboard shows a renewal chip in the top bar and the license page surfaces
the same warning:

```
┌──────────────────────────────────────────────────────────┐
│ ⚠ Your trial expires in 5 days. Contact support to renew │
└──────────────────────────────────────────────────────────┘
```

**Test:** `install-ux.test.js` test 5

---

## Phase 5: Trial Expiry

### Step 5.1: API blocks protected endpoints

After 30 days, all API endpoints except health/status/extend return:

```http
HTTP/1.1 403 Forbidden
Content-Type: application/json

{
  "detail": "Trial expired",
  "message": "Your 30-day trial has expired. Contact support for an extension code.",
  "extend_endpoint": "/v1/licensing/trial/extend"
}
```

**Endpoints that remain open:**
- `GET /health` — Docker healthcheck (always 200)
- `GET /v1/licensing/trial/status` — shows expiry info
- `POST /v1/licensing/trial/extend` — accepts renewal codes

### Step 5.2: Dashboard and public `/renew` page show renewal screen

```
┌─────────────────────────────────────────────────┐
│                                                 │
│        ⚠  Your trial has expired                │
│                                                 │
│   Contact support@bwater.io to renew.           │
│   Include your Machine ID and License ID.       │
│                                                 │
│   ┌─────────────────────────────────────────┐   │
│   │ Paste extension code here               │   │
│   └─────────────────────────────────────────┘   │
│                                                 │
│   ┌─────────────────────────────────────────┐   │
│   │             Activate                    │   │
│   └─────────────────────────────────────────┘   │
│                                                 │
│   Machine ID: a1b2c3d4e5f6...                   │
│   License ID: BW-2026-0042                      │
│   Expired:    2026-05-04                        │
│                                                 │
└─────────────────────────────────────────────────┘
```

**Test:** `install-ux.test.js` test 7 (invalid code rejection)

---

## Phase 6: License Renewal (1 minute)

### Option A: CLI

```bash
$ sudo breakwater license renew
Enter your 30-day extension code
(Provided by Breakwater support)
Extension code: ••••••••••••

License extended successfully!
{
  "new_expires_at": "2026-06-03T00:00:00Z",
  "days_remaining": 30
}
```

### Option B: Dashboard

Paste code into the renewal screen, click Activate.

### Option C: API

```bash
curl -X POST http://localhost:8000/v1/licensing/trial/extend \
  -H "Content-Type: application/json" \
  -d '{"code": "<extension-code>"}'
```

**After renewal:** Dashboard immediately returns to normal. No restart needed.

---

## Performance Benchmarks

| Metric | Expectation | Measured By |
|--------|-------------|-------------|
| Full installation time | <5 minutes | Clean Ubuntu x86_64 install run |
| API health response | <500ms | `install-ux.test.js` test 8 |
| Login page load | <3000ms | `install-ux.test.js` test 8 |
| Login → dashboard redirect | <5000ms | `install-ux.test.js` test 8 |
| Trial status API | <500ms | `install-ux.test.js` test 8 |
| Extension code apply | <2000ms | `install-ux.test.js` test 7 |
| `breakwater status` command | <5000ms | `install-cli.test.js` |

---

## Test Coverage Matrix

| User Step | Test File | Test # | Screenshot |
|-----------|-----------|--------|------------|
| API healthy after install | `install-ux.test.js` | 1 | — |
| Trial active after install | `install-ux.test.js` | 2 | — |
| Login page renders | `install-ux.test.js` | 3 | `01-login-page.png` |
| Admin can log in | `install-ux.test.js` | 4 | `04-dashboard-after-login.png` |
| Trial headers present | `install-ux.test.js` | 5 | — |
| License settings page | `install-ux.test.js` | 6 | `06-license-settings.png` |
| Bad extension code rejected | `install-ux.test.js` | 7 | — |
| Performance benchmarks | `install-ux.test.js` | 8 | — |
| CLI help text | `install-cli.test.js` | 1 | — |
| CLI status command | `install-cli.test.js` | 2 | — |
| CLI license status | `install-cli.test.js` | 3 | — |
| Install directory structure | `install-cli.test.js` | 6 | — |
| .env secrets not placeholder | `install-cli.test.js` | 7 | — |
| Terminal screencast | Public install capture | — | `customer-install-demo.cast` |

---

## Reference Artifacts

### Public installation references

```text
Guide:      https://github.com/bwater-io/installer/blob/main/docs/INSTALL_GUIDE.md
Screencast: https://raw.githubusercontent.com/bwater-io/installer/main/docs/media/customer-install-demo.mp4
GIF:        https://raw.githubusercontent.com/bwater-io/installer/main/docs/media/customer-install-demo.gif
```

### Run UX tests with screenshots

```bash
cd e2e
npm test -- --testPathPatterns installation/install-ux
# Screenshots saved to: e2e/screenshots/installation/
```

### Run CLI tests

```bash
cd e2e
sudo npm test -- --testPathPatterns installation/install-cli
```

### Record screencast

```bash
curl -fsSL https://install.bwater.io | sudo bash
```

Public screencast assets are published at the URLs above.

### Generate screenshots for documentation

The Puppeteer test suite saves PNG screenshots at each step:

```
e2e/screenshots/installation/
├── 01-login-page.png
├── 02-login-page-annotated.png
├── 03-login-filled.png
├── 04-dashboard-after-login.png
├── 05-dashboard-main.png
├── 06-license-settings.png
├── 07-settings-page.png
└── timing-report.json
```

These screenshots can be embedded in the installation guide PDF or
documentation site.
