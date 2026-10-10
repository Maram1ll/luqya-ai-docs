# لُقيا AI – University AI-Powered Lost & Found System

لُقيا AI is a specialized university lost and found platform developed with **Pure PHP**, **MySQL**, and **Vanilla HTML5/CSS3/ES6+ JavaScript**. Its user interface and the entire system are designed and developed **fully in Arabic (RTL)** to seamlessly support the Saudi university environment. It features simulated AI auto-tagging, OCR card scanning for student ID cards, automated matching with secure pickup codes, a dedicated Security Officer handover dashboard, and a custom glowing SVG brand mark: a tech magnifying glass with an embedded AI circuit board and neural nodes.

---

## 🚀 Deploying to free hosting?

See **[DEPLOY.md](DEPLOY.md)** — a step-by-step guide for 000webhost / InfinityFree /
Hostinger or any Apache + PHP + MySQL host, including the security checklist and how to
run without a Gemini API key.

---

##  Key Features

1. **System Actors & Workflows**:
   - **Student / Beneficiary**: Registers/logs in with university ID format (`S` + 9 digits), reports lost items with AI auto-tagging, reports found items through a unified smart upload that auto-detects ID cards and extracts the student ID via vision OCR, tracks reports on personal Profile page with step-by-step lifecycle progress trackers.
   - **Finder / Good Samaritan**: Reports found items (details, location, date), uploads a single image via the smart field (auto-detects whether it is an ID card to extract the student ID, or a general item photo), and hands physical items directly to the Security Officer.
   - **Security / Lost & Found Officer**: Dedicated Security Dashboard with fast Pickup Code / Student ID verification, instant handover confirmation, and automatic report resolution.
   - **AI System (Simulated Engine)**: Simulated image auto-tagging, OCR for ID/ATM cards (extracting Student ID `S000000001`, owner name, college), automated matching engine with high confidence score (>85%), and unique 6-digit verification **Pickup Codes** (`RET-XXXXXX`).

2. **Strict Identity Validation**:
   - Username / Student ID format: `S` followed by 9 digits (e.g. `S000000001`).
   - Auto-generated University Email: `s000000001@uoh.edu.sa`.
   - Frontend and Backend regex validation.

3. **Fully Arabic (RTL) — Built for the Saudi University Environment**:
   - The entire user interface and the whole system are designed and developed fully in Arabic and rendered right-to-left (RTL) to seamlessly support Saudi universities.
   - The document root is always served as `<html lang="ar" dir="rtl">`; every UI label, form hint, validation message, notification and AI reply is authored in Arabic.
   - RTL-native styling throughout the design system (`[dir="rtl"]` rules) keeps every component correctly mirrored and aligned.

4. **Design System & Branding**:
   - Custom Inline SVG Brand Mark: A photorealistic blue magnifying glass on a brushed metallic silver surface, with an embedded glowing AI circuit board and neural nodes, accented with vibrant cyan.
   - Color Palette: Navy Blue (`#1B365D`), Cyan Blue (`#00A8B5`), Mint Green (`#4CD9A1`), and Clean White (`#FFFFFF`).
   - Soft-rounded cards (`14px`), responsive grids, glassmorphism headers, interactive FAQ accordion, live activity feed ticker, and Mint Green toast alerts.

---

##  Quick Setup & Installation Guide

1. **Place Project in XAMPP Directory**:
   Ensure the codebase is located in `C:\xampp\htdocs\luqya-ai`.

2. **Initialize Database**:
   - Open XAMPP Control Panel and start **Apache** and **MySQL**.
   - Run the database setup script by visiting in browser or executing via PHP CLI:
     ```bash
     C:\xampp\php\php.exe c:\xampp\htdocs\luqya-ai\setup_db.php
     ```
     Or open in browser: `http://localhost/luqya-ai/setup_db.php`

3. **Access Application**:
   Open browser at: `http://localhost/luqya-ai/index.php`

---

##  Demo Account Credentials

Demo accounts are seeded for local development. These are dummy/synthetic credentials �?replace them with real values before any deployment.

The credentials below were verified against the live `users` table with `password_verify()`.
`login.php` also renders a **quick demo login** panel that pre-fills these same values
(see [Quick demo login](#quick-demo-login) below).

### Students

| University ID | Email | Phone | Password |
|---|---|---|---|
| `S202403400` | `s202403400@uoh.edu.sa` | `+966551234567` | `password123` |
| `S202409876` | `s202409876@uoh.edu.sa` | `+966559876543` | `password123` |
| `S202401234` | `s202401234@uoh.edu.sa` | `+966554321987` | `password123` |
| `S202447940` | `s202447940@uoh.edu.sa` | `0533928899` | *(not a shared demo password)* |

The first three students share the seeded `password123` digest. `S202447940` has a
different password, so use `register.php` to create an account if you need a fourth.

### Security Officers

| Login Security Code | Email | Password |
|---|---|---|
| `SEC-USR-1` | `sec-b1@uoh.edu.sa` | `security123` |
| `SEC-USR-7` | `sec-b2@uoh.edu.sa` | `security123` |
| `SEC-USR-8` | `sec-g1@uoh.edu.sa` | `security123` |
| `SEC-USR-9` | `sec-g2@uoh.edu.sa` | `security123` |

The officer login code **is** the identifier you type into the login form (the
`الرقم الجامعي أو كود الأمن` field). It is stored as a bcrypt digest in
`users.security_code_hash`; there is no plaintext password in the source.

All seed names, IDs, reports and activity are synthetic/anonymized (see the note at the top of `schema.sql`).

### Quick demo login

Set `LUQYA_DEMO_LOGIN=1` in `.env` (it is on by default) and `login.php` shows a
"دخول تجريبي سريع" panel with one button per role. Clicking a button **only fills the
two form fields**; the form is then submitted normally to `api/auth.php?action=login`,
so CSRF tokens, rate limiting and `password_verify()` all still apply. No
authentication path is bypassed, and the button is the only thing the flag controls.
Set `LUQYA_DEMO_LOGIN=0` in production to hide the panel.

### Password recovery without a mail server

This build ships **no mailer**. With `LUQYA_EXPOSE_RESET_TOKEN=1` the reset link is
written to the PHP error log instead of being emailed:

1. Open `forgot_password.php` and submit your **university ID** and **registered
   university email**. If your account has a security code, enter it as well.
2. Open `C:\xampp\apache\logs\error.log` and find the line starting with
   `Luqya AI: [DEV ONLY] password reset link for user #...`.
3. Copy the `token=` value and open `forgot_password.php?step=reset&token=<value>`,
   or paste it into the second step.

**Strict identity verification.** Before any token is issued the account is resolved by
university ID and the email must match the registered address exactly. The security code
is checked with `password_verify()` against `security_code_hash` and is **required only
when the account has one** (an officer's code is their `SEC-USR-…` login). Accounts
created before the security-code feature have no code and reset with university ID +
email alone, so they are never locked out. The same generic message is returned for every
outcome, so the endpoint cannot be used to enumerate accounts.

The security code can be set or changed from the **"الإعدادات الأمنية"** card on your
profile — its form is hidden behind a "تغيير الرمز الأمني" toggle button and only opens
when you click to edit.

Tokens are SHA-256 hashed in `password_resets` (the raw token is never stored), expire
after 30 minutes, and are single-use. **Never enable `LUQYA_EXPOSE_RESET_TOKEN` in
production** — the raw token grants a full password change for its lifetime.

---

## Database Schema

MySQL/MariaDB (InnoDB, `utf8mb4_unicode_ci`). Six tables; all foreign keys use
`ON DELETE CASCADE` (except the intentionally standalone audit log):

| Table | Columns | Purpose |
|---|---|---|
| `users` | 13 | Accounts; `password_hash` and `security_code_hash` are bcrypt-only, never plaintext |
| `reports` | 17 | Lost/found reports — the heart of the system |
| `matches` | 10 | Lost ↔ found match pairs sharing one pickup code |
| `notifications` | 7 | In-app user notifications |
| `activity_logs` | 6 | Standalone audit trail (no FK by design) |
| `password_resets` | 6 | Password recovery tokens — only `SHA-256(token)` is stored (`token_hash CHAR(64)`), plus `expires_at` and a single-use `used_at` marker |

See **[luqya_database_schema.txt](luqya_database_schema.txt)** for the complete
column-by-column definition.

---

##  Directory Architecture

```
luqya-ai/
├── config/
│   └── db.php                  # Database PDO Connection & Helper Functions
├── api/
│   ├── auth.php                # API for Login, Register, Forgot/Reset Password, Security Code, Logout
│   ├── reports.php             # API for creating, listing, closing lost/found reports
│   ├── security.php            # API for Security Officer pickup code verification & handover
│   ├── notifications.php       # API for fetching & marking notifications
│   └── ocr_sim.php             # API for simulated AI image tagging & OCR card scanning
├── assets/
│   ├── css/
│   │   └── style.css           # Complete Vanilla CSS Design System & Responsive Layouts
│   ├── js/
│   │   ├── app.js              # Core UI interactivity, search/filters, activity ticker
│   │   ├── ocr_ai.js           # AI Tagging & Interactive OCR Card Scanner Simulation
│   │   └── toast.js            # Mint Green Pop-up Toast Notifications System
│   └── images/                 # SVG graphics & sample uploads
├── database/
│   └── schema.sql              # MySQL Schema & Seed Initial Data
├── views/
│   ├── header.php              # Navbar (Arabic RTL), SVG Brand Mark (tech magnifier + AI circuit), Notifications Dropdown
│   ├── footer.php              # University Footer & Quick Links
│   ├── modal_report_lost.php   # Modal for reporting lost items + AI auto-tag simulation
│   └── modal_report_found.php  # Modal for reporting found items + unified smart upload (general photo or ID-card OCR)
├── index.php                   # Main Dashboard (Metrics, Search, Feed, Grid, FAQ, About)
├── login.php                   # Authentication Login Page
├── register.php                # Authentication Registration Page
├── forgot_password.php         # Password Reset Request Page (ID + email + optional security code)
├── profile.php                 # User Profile Page, Lifecycle Tracker & Security Settings (toggleable)
├── security_dashboard.php      # Security Officer Pickup Verification & Handover Dashboard
├── setup_db.php                # Automatic MySQL Database Setup & Seeding Script
└── README.md                   # System Documentation
```
