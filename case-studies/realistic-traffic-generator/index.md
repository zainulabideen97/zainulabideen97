### Problem description

**Business problem**

- **Need**: Automatically generate realistic web traffic and sign‑ups for multiple client sites, while tracking detailed activity and enforcing per‑site limits.
- **Context**:
  - Each client has one or more landing pages.
  - For each page, you want controlled numbers of **visits** and **form signups** per day.
  - Traffic must look human (scrolls, key presses, delays, varied user agents, time windows, proxies, etc.).
  - Operators need a **web UI + API** to configure sites, landing pages, selectors, and monitor actions.
- **Solution**:
  - A **core traffic service** (`TrafficService`) that uses Selenium (with Bright Data proxies, stealth, and anti‑captcha) to visit pages and submit forms based on database configuration.
  - A **Tornado web/API server** that exposes CRUD endpoints and a portal UI for managing sites, pages, selectors, users, and dashboards.
  - A **MySQL database** storing configuration and logs (`Site`, `LandingPage`, `PageSelector`, `SiteRecord`, `Action`, `User`, etc.).

---

### Architecture diagram (Mermaid)

```mermaid
flowchart LR
    subgraph userOperator [User / Operator]
        U1["Browser - Admin Portal"]
    end

    subgraph webLayer [Web Layer]
        WS["MainWebServer (Tornado)"]
        WH["Web Handlers - MainWebHandler, Auth, Portal"]
        API["API Handlers - core modules"]
    end

    subgraph coreServiceLayer [Core Service Layer]
        TS["TrafficService - Selenium + Stealth"]
    end

    subgraph infra [Infrastructure]
        DB["MySQL Database"]
        PROXY["Bright Data Proxy"]
        CAPTCHA["Anti-Captcha (GHL reCAPTCHA v2/v3)"]
        CHROME["Headless Chrome (undetected-chromedriver)"]
    end

    U1 --> WS
    WS --> WH
    WS --> API

    WS --> DB

    TS --> DB
    TS --> PROXY
    TS --> CHROME
    CHROME --> CAPTCHA

    API --> DB
    WH --> DB

    subgraph systemdServices [Systemd Services]
        S1["kore-outreach.service"]
        S2["scrapper-core-service.service (runs TrafficService)"]
        S3["kor-acast-service.service"]
    end

    S2 --> TS
    S1 --> WS
    S3 --> DB
```

---

### UI screens (Mermaid diagram)

```mermaid
flowchart TD
    LOGIN[Login Screen]
    DASH[Dashboard]
    SITES[Sites List]
    SITE_DETAIL[Site Detail<br/>+ Landing Pages]
    LP_SELECTORS[Landing Page Selectors<br/>(Form Mapping)]
    RECORDS[Site Records<br/>(Leads List)]
    ACTIONS[Actions Log<br/>(Visits & Signups)]
    USERS[User Management]
    SETTINGS[System Settings]

    LOGIN --> DASH

    DASH --> SITES
    DASH --> ACTIONS
    DASH --> RECORDS
    DASH --> USERS
    DASH --> SETTINGS

    SITES --> SITE_DETAIL
    SITE_DETAIL --> LP_SELECTORS
    SITE_DETAIL --> RECORDS

    LP_SELECTORS --> ACTIONS
    RECORDS --> ACTIONS
```

**Screen purposes (brief)**
- **Login**: Authenticate operators into the portal.
- **Dashboard**: High‑level counts (visits, signups, success/failure, per‑site summary).
- **Sites List / Detail**: Configure sites, concurrency, visit/signup limits, delay/time windows.
- **Landing Page Selectors**: Configure CSS/XPath selectors for inputs, checkboxes, buttons, headings.
- **Site Records**: Inspect/import the raw lead data (`SiteRecord`) used for form fills.
- **Actions Log**: Detailed history of automated actions (time spent, activity count, status, metadata).
- **Users / Settings**: Access control, keys, base URL, logging level, etc.

---

### Workflow (Mermaid diagram)

```mermaid
sequenceDiagram
    participant Admin as Admin User
    participant UI as Web UI (Portal)
    participant API as API Handlers
    participant DB as MySQL DB
    participant SVC as TrafficService
    participant SEL as Selenium + Proxy

    Admin->>UI: Login & configure sites/pages/selectors
    UI->>API: Save site, landing pages, selectors, limits
    API->>DB: Insert/Update Site, LandingPage, PageSelector, SiteRecord

    Note over SVC: scrapper-core-service.service starts

    SVC->>DB: Select active Sites
    SVC->>DB: For each Site, check Action counts & LastVisit
    SVC->>SVC: Decide VISIT vs SIGNUP via CheckForLimit

    SVC->>DB: Get random eligible LandingPage
    SVC->>DB: Get random SiteRecord (for signup)
    SVC->>SEL: Launch Chrome via Bright Data proxy + stealth
    SEL->>TargetSite: Open Url / OptInUrl

    alt VISIT
        SVC->>SEL: Random scrolls, key presses, waits
    else SIGNUP
        SVC->>SEL: Fill inputs (type_like_human)
        SVC->>SEL: Click checkboxes/buttons
        SEL->>TargetSite: Submit form
    end

    SVC->>DB: Insert Action row (TimeSpent, ActivityCount, metadata, result)
    SVC->>DB: Update SiteRecord.Status (for successful signup)
    SVC->>DB: Update Site.LastVisit

    Admin->>UI: Open dashboard / logs
    UI->>API: Fetch Actions, Sites, stats
    API->>DB: Query summaries
    DB-->>API: Aggregated metrics
    API-->>UI: Charts, tables, per-site performance
```

If you want a **video-style workflow**, this sequence is the storyboard: each step (configure → schedule → browse → submit → log → dashboard) can be turned into a short clip or slide, with arrows matching this diagram.

---

### Results / impact

- **Controlled, human‑like traffic generation**
  - Configurable **daily caps** and **time windows** per site via `HasDailyLimit`, `CheckForLimit`, `LastVisit` updates.
  - Random delays, scrolling, key presses, mouse movement, window sizes, user agents, and proxies make sessions appear organic.

- **Automated signup completion and lead tracking**
  - **Form filling** driven by `PageSelector` definitions and `SiteRecord` data, with retries and error logging.
  - Each action is captured as an `Action` with **time spent**, **activity count**, and rich metadata, enabling deep analytics.

- **Operational visibility and control**
  - Systemd services (`scrapper-core-service.service`, `kore-outreach-service.service`, `kor-acast-service.service`) give ops a robust way to start/stop/restart and tail logs.
  - The web portal and API provide a central place to manage sites, view activity, and adjust configuration without touching code.

- **Business impact**
  - Reduces manual effort to test, warm up, or drive traffic to marketing funnels.
  - Provides consistent data (visits, signups, time on page, success/failure reasons) to optimize pages and campaigns over time.

If you’d like, I can now refine these into a slide‑ready outline (e.g., one slide per section) or expand the UI and workflow diagrams for a more detailed presentation deck.

### Problem description

**Business problem**

- **Need**: Automatically generate realistic web traffic and sign‑ups for multiple client sites, while tracking detailed activity and enforcing per‑site limits.
- **Context**:
  - Each client has one or more landing pages.
  - For each page, you want controlled numbers of **visits** and **form signups** per day.
  - Traffic must look human (scrolls, key presses, delays, varied user agents, time windows, proxies, etc.).
  - Operators need a **web UI + API** to configure sites, landing pages, selectors, and monitor actions.
- **Solution**:
  - A **core traffic service** (`TrafficService`) that uses Selenium (with Bright Data proxies, stealth, and anti‑captcha) to visit pages and submit forms based on database configuration.
  - A **Tornado web/API server** that exposes CRUD endpoints and a portal UI for managing sites, pages, selectors, users, and dashboards.
  - A **MySQL database** storing configuration and logs (`Site`, `LandingPage`, `PageSelector`, `SiteRecord`, `Action`, `User`, etc.).

---

### Architecture diagram

![Architecture diagram](./images/architecture.svg)

---

### UI screens

![UI screens](./images/ui-flow.svg)

**Screen purposes (brief)**
- **Login**: Authenticate operators into the portal.
- **Dashboard**: High‑level counts (visits, signups, success/failure, per‑site summary).
- **Sites List / Detail**: Configure sites, concurrency, visit/signup limits, delay/time windows.
- **Landing Page Selectors**: Configure CSS/XPath selectors for inputs, checkboxes, buttons, headings.
- **Site Records**: Inspect/import the raw lead data (`SiteRecord`) used for form fills.
- **Actions Log**: Detailed history of automated actions (time spent, activity count, status, metadata).
- **Users / Settings**: Access control, keys, base URL, logging level, etc.

---

### Workflow

![Workflow](./images/workflow.svg)

If you want a **video-style workflow**, this sequence is the storyboard: each step (configure → schedule → browse → submit → log → dashboard) can be turned into a short clip or slide, with arrows matching this diagram.

---

### Results / impact

- **Controlled, human‑like traffic generation**
  - Configurable **daily caps** and **time windows** per site via `HasDailyLimit`, `CheckForLimit`, `LastVisit` updates.
  - Random delays, scrolling, key presses, mouse movement, window sizes, user agents, and proxies make sessions appear organic.

- **Automated signup completion and lead tracking**
  - **Form filling** driven by `PageSelector` definitions and `SiteRecord` data, with retries and error logging.
  - Each action is captured as an `Action` with **time spent**, **activity count**, and rich metadata, enabling deep analytics.

- **Operational visibility and control**
  - Systemd services (`scrapper-core-service.service`, `kore-outreach.service`, `kor-acast-service.service`) give ops a robust way to start/stop/restart and tail logs.
  - The web portal and API provide a central place to manage sites, view activity, and adjust configuration without touching code.

- **Business impact**
  - Reduces manual effort to test, warm up, or drive traffic to marketing funnels.
  - Provides consistent data (visits, signups, time on page, success/failure reasons) to optimize pages and campaigns over time.

If you’d like, I can now refine these into a slide‑ready outline (e.g., one slide per section) or expand the UI and workflow diagrams for a more detailed presentation deck.

