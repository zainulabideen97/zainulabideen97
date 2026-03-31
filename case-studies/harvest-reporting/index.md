### Problem description

Harvest Reporting MVP is a reporting and compensation tool for law firms that use Harvest for time tracking. The core problem it solves is that Harvest’s native reports don’t answer key questions for firm leadership and attorneys:

- **For attorneys**: How am I tracking against my quarterly and annual bonus targets (billable, invoiced, and collected)? How much potential bonus is “locked” in uninvoiced or uncollected work?
- **For management**: Where is revenue stuck in the pipeline (uninvoiced / uncollected, by client and project, with aging)? Which attorneys are on/off pace for targets? How do we generate consistent PDF/summary reports for attorneys and leadership?
- **For operations**: How do we keep an internal analytics database incrementally synced with Harvest (time entries, invoices, payments, users, projects) in a robust, observable way?

The MVP introduces a dedicated backend that syncs Harvest data into a MariaDB warehouse, exposes higher-level reporting and bonus/true‑up calculations, and a front‑end that surfaces dashboards for attorneys and management.

---

### Architecture diagram (mermaid)

```mermaid
flowchart LR
    subgraph harvestLayer ["Harvest SaaS"]
        HTE["Time entries"]
        HINV["Invoices"]
        HPAY["Invoice payments"]
        HUSR["Users / attorneys"]
        HCLI["Clients & projects"]
    end

    subgraph backendLayer ["Backend (FastAPI, Python 3.11+)"]
        HARVEST_CLIENT["Harvest API client (app/clients/harvest_client.py)"]
        SYNC_RUNNER["Harvest sync runner CLI (app/services/harvest_sync_runner.py)"]
        SVC_REPORTS["Reporting & dashboard services (reporting_service.py, dashboard.py)"]
        SVC_BONUS["Bonus & true-up services (bonus_service.py, bonus_year_utils.py, true_up_service.py)"]
        SVC_COMP["Compensation & analytics (compensation_service.py, attorney_analytics_service.py)"]
        SVC_COLLECTIONS["Collections & payments sync (invoice_payment_sync_service.py, collection_service.py)"]
        AUTH["Auth & users (auth_service.py, user_service.py)"]
        API["FastAPI router & routes (app/main.py, app/api/*)"]
    end

    subgraph dbLayer ["MariaDB (analytics DB)"]
        T_TIME["harvest_time_entries"]
        T_INV["harvest_invoices"]
        T_PAY["harvest_invoice_payments"]
        T_USERS["harvest_users"]
        T_PROJECTS["harvest_projects & assignments"]
        T_BONUS["bonus_calcs, payouts, true_ups"]
        T_SYNC["sync_runs & sync_checkpoints"]
    end

    subgraph frontendLayer ["Frontend (React/Vite)"]
        UI_LOGIN["Login & auth flow (LoginPage.tsx, ForgotPasswordPage.tsx)"]
        UI_DASH["Attorney dashboard (DashboardPage.tsx)"]
        UI_ATTYS["Attorneys list & filters (AttorneysPage.tsx)"]
        UI_ATTY_DETAIL["Attorney detail & bonus view (AttorneyDetailsPage.tsx + subcomponents)"]
        UI_MGMT["Management dashboard (ManagementDashboardPage.tsx)"]
        UI_PROJ["Projects view (ProjectsPage.tsx)"]
        SHELL["App shell & layout (App.tsx, Shell.tsx)"]
    end

    subgraph infraLayer ["Infra / Ops"]
        CRON["Cron / scheduler (runs sync runner)"]
        MAILER["Email / PDF delivery (mailer.py, management_pdf.py, pdf_digest.py)"]
    end

    HTE --> HARVEST_CLIENT
    HINV --> HARVEST_CLIENT
    HPAY --> HARVEST_CLIENT
    HUSR --> HARVEST_CLIENT
    HCLI --> HARVEST_CLIENT

    CRON --> SYNC_RUNNER
    SYNC_RUNNER --> HARVEST_CLIENT
    SYNC_RUNNER --> T_TIME
    SYNC_RUNNER --> T_INV
    SYNC_RUNNER --> T_PAY
    SYNC_RUNNER --> T_USERS
    SYNC_RUNNER --> T_PROJECTS
    SYNC_RUNNER --> T_SYNC

    API --> SVC_REPORTS
    API --> SVC_BONUS
    API --> SVC_COMP
    API --> SVC_COLLECTIONS
    API --> AUTH

    SVC_REPORTS --> dbLayer
    SVC_BONUS --> dbLayer
    SVC_COMP --> dbLayer
    SVC_COLLECTIONS --> dbLayer
    AUTH --> dbLayer

    frontendLayer -->|HTTPS JSON| API
    MAILER -->|PDF / email| AttorneysAndManagement["Attorneys & management"]
```

---

### UI screens (mermaid diagram)

This focuses on the main screens and key data they surface, aligned with your `h1` spec (Summary, Revenue Pipeline & Aging, Time & Productivity) and existing React pages.

```mermaid
flowchart TD
    START[Login Screen\nEmail + Password + TOTP]
    FORGOT[Forgot Password\nReset Request]
    DASH[Attorney Dashboard\nTab 1 - Summary]
    PIPELINE[Revenue Pipeline & Aging\nTab 2]
    PRODUCTIVITY[Time & Productivity\nTab 3]
    ATTYS[Attorneys List\n(firm-wide view)]
    ATTY_DETAIL[Attorney Detail Screen\nQTD/YTD, Bonus, Uninvoiced, Collections]
    BONUS_SEC[Bonus Dashboard & Q4 True-up Card]
    MGMT[Management Dashboard\nFirm-wide KPIs]
    PROJ[Projects / Clients View]

    START -->|Forgot password?| FORGOT
    START -->|Successful login| DASH

    DASH -->|Tab: Revenue pipeline & aging| PIPELINE
    DASH -->|Tab: Time & productivity| PRODUCTIVITY

    DASH -->|View all attorneys| ATTYS
    ATTYS -->|Select attorney| ATTY_DETAIL

    ATTY_DETAIL --> BONUS_SEC
    ATTY_DETAIL --> PIPELINE
    ATTY_DETAIL --> PRODUCTIVITY

    DASH -->|Mgmt role| MGMT
    MGMT --> PROJ

    %% Dash content (Summary)
    subgraph Summary_Cards["Attorney Dashboard - Summary (Tab 1)"]
        QTD[QTD: Billed / Invoiced / Not Invoiced\n+ Collected / Not Collected]
        BONUS_QTD[QTD Bonus Card\n- Actual vs Target\n- Pace to threshold\n- Estimated quarterly bonus\n- Bonus locked in Not Invoiced]
        YTD[YTD: Billed / Invoiced / Not Invoiced\n+ Collected / Not Collected]
        BONUS_YTD[YTD Bonus Card\n- Actual vs Target (2x salary)\n- % to yearly threshold\n- Estimated yearly bonus\n- Bonus locked in Not Invoiced / Not Collected]
    end

    DASH --> Summary_Cards
```

---

### Workflow (mermaid diagram)

This shows the end‑to‑end flow from Harvest to dashboards/bonus calculations and PDFs.

```mermaid
sequenceDiagram
    participant Cron as Cron / Scheduler
    participant SyncRunner as Harvest Sync Runner<br/>`python -m app.services.harvest_sync_runner`
    participant Harvest as Harvest API
    participant DB as MariaDB (Analytics DB)
    participant Backend as FastAPI Backend<br/>(`/harvest/*`, `/bonus/*`, `/reports/*`)
    participant FE as Frontend (React)
    participant User as Attorney / Manager
    participant Email as Email/PDF Delivery

    Cron->>SyncRunner: Run with datasets (users, clients_projects, tasks,<br/>time_entries, invoices, payments)
    SyncRunner->>Harvest: Pull updated_since data per dataset
    Harvest-->>SyncRunner: Batched JSON (time entries, invoices, payments...)
    SyncRunner->>DB: Upsert into harvest_* tables<br/>Update sync_runs & sync_checkpoints

    User->>FE: Open app, login with password/TOTP
    FE->>Backend: POST /auth/login\n(receive JWT access token)
    Backend->>DB: Validate user, read `users` & TOTP state
    DB-->>Backend: User & auth info
    Backend-->>FE: Access token

    User->>FE: Navigate to Dashboard / Attorney Detail
    FE->>Backend: GET /dashboard, /harvest/summary,<br/>/harvest/attorney/{id}, /harvest/attorney/{id}/uninvoiced
    Backend->>DB: Query time_entries, invoices, payments,<br/>bonus_calcs, payouts, config
    DB-->>Backend: Aggregated metrics (QTD/YTD billed, invoiced,<br/>uninvoiced, collected, bonus tiers, true-up amounts)
    Backend-->>FE: JSON with metrics + bonus recommendations
    FE-->>User: Render Summary, Pipeline, Productivity tabs<br/>+ Q4 true-up recommended payment

    User->>Backend: POST /bonus/run (per quarter/year)
    Backend->>DB: Compute + persist bonus calcs & recommended payouts
    Backend-->>User: Updated bonus numbers

    User->>Backend: POST /bonus/payouts/{id}/mark-paid
    Backend->>DB: Mark payouts as PAID (affects recommended Q4 true-up)

    Manager->>Backend: Trigger management or attorney report\n(e.g., send PDF)
    Backend->>DB: Fetch metrics
    Backend->>Email: Generate PDF (management/attorney digest)\nSend via SMTP
    Email-->>User: PDF report in inbox
```

---

### Results / impact

- **Transparency for attorneys**: Each attorney gets a clear QTD and YTD view of billed, invoiced, uninvoiced, collected, and not collected work, plus how much bonus is locked in each stage of the pipeline.
- **Actionable revenue pipeline**: Management sees aging of uninvoiced and uncollected amounts by client/project, helping prioritize invoicing and collections to unlock revenue and bonuses.
- **Robust, incremental sync from Harvest**: A dedicated sync runner uses checkpoints and structured logging to keep a MariaDB analytics store current, enabling all dashboards and reports to read from a single, consistent source of truth.
- **Bonus & true-up automation**: Quarterly and annual bonus tiers, Q4 true-up calculations, and recommended payment amounts are computed from actual collections and prior payouts, reducing manual spreadsheet work and payment errors.

