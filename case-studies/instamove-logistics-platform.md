### Problem description

**Instamove** is a property rental platform focused on London that helps renters discover rooms, flatshares, and full properties while giving landlords a self‑service way to create, promote, and manage listings. It solves the problem of fragmented, generic rental search by combining granular filters (price, room type, area, amenities, commute time), lifestyle “house vibes”, and in‑app messaging/booking into a single, data‑driven marketplace with SEO‑optimized, location‑aware discovery.

---

### Architecture diagram (mermaid)

```mermaid
flowchart LR
    subgraph Client
        B[Web Browser\n(React/HTML templates)]
    end

    subgraph Backend[Flask App (Instamove)]
        A[Flask App\napp.py]
        BP1[Blueprint: home\n/browse, /, search]
        BP2[Blueprint: properties\nlisting, details, favorites, dashboard]
        BP3[Blueprints: auth, clients,\nmessages, notifications, payment, webhooks, premium]
        Admin[Flask-Admin\nManagement Console]
        Socket[SocketIO\nReal-time messaging/notifications]
        Celery[Celery Worker + Beat\nBackground tasks]
    end

    subgraph Data
        DB[(PostgreSQL\nProperties, Clients,\nMessages, Metrics, Premium)]
        S3[(AWS S3\nImages & Videos)]
        Stripe[(Stripe\nPayments & Billing)]
        SES[(AWS SES\nTransactional Email)]
        GoogleSSO[(Google OAuth\nSocial Login)]
        Maps[(Google Maps & GetAddress\nGeocoding & Stations)]
    end

    B <--> A
    A --> BP1
    A --> BP2
    A --> BP3
    A --> Admin
    A <--> Socket

    A <--> DB
    Celery <--> DB

    BP2 --> S3
    BP3 --> Stripe
    BP3 --> SES
    BP3 --> GoogleSSO
    BP1 --> Maps
    BP2 --> Maps

    A <--> Celery
```

---

### UI screens (mermaid diagram)

```mermaid
flowchart TD
    Landing[Landing / Home\nProperty search & SEO content]
    Browse[Browse Area\n/browse map + filters]
    Results[Search Results\nlist + map + filters]
    Details[Property Details\nimages, rooms, flatmates, vibes]
    SignUp[Sign Up / Login\nemail + Google SSO]
    Dashboard[User Dashboard\nsaved filters, favorites]
    PostAd[Post Ad\nmulti-step landlord form]
    EditAd[Edit Listing\nupdate images, rooms, pricing]
    Favourites[Favourites\nsaved properties]
    AdminUI[Admin Panel\nFlask-Admin dashboards]

    Landing --> Browse
    Landing --> Results
    Results --> Details
    Details --> SignUp:::auth
    Landing --> SignUp
    SignUp --> Dashboard
    Dashboard --> Favourites
    Dashboard --> PostAd
    PostAd --> EditAd
    EditAd --> Dashboard
    Dashboard --> Results
    AdminUI:::admin

    classDef auth fill:#eef,stroke:#336,stroke-width:1px;
    classDef admin fill:#fee,stroke:#933,stroke-width:1px;
```

---

### Workflow (mermaid diagram)

```mermaid
sequenceDiagram
    participant R as Renter
    participant L as Landlord
    participant W as Web UI
    participant F as Flask Backend
    participant DB as PostgreSQL
    participant S3 as S3
    participant C as Celery/Tasks
    participant P as Stripe

    Note over R,L: Discovery & search
    R->>W: Open landing / search page
    W->>F: GET / (filters, area, station)
    F->>DB: Query Properties + Metrics
    F->>Maps: (internal) Geocode areas & stations
    F-->>W: Render results + map + SEO content

    Note over R: Evaluate and engage
    R->>W: Click property
    W->>F: GET /property-to-rent/...
    F->>DB: Load Property, Rooms, Flatmates, Metrics
    F-->>W: Render property details + suggested listings
    W->>F: POST /track-time, /record-impression
    F->>DB: Update PropertyMetrics (views, time_spent)

    Note over R: Save & communicate
    R->>W: Login / Sign Up (email or Google)
    W->>F: Auth request
    F->>DB: Create / fetch Client
    F-->>W: Auth session
    R->>W: Add to favourites, start chat, save filter
    W->>F: POST /add-favorite, /save-filter, messages
    F->>DB: Persist favorites, threads, filters

    Note over L: List property
    L->>W: Login and open Post Ad
    W->>F: GET /post-ad
    L->>W: Fill multi-step form + upload media
    W->>F: POST /api/properties/generate-presigned-url
    F->>S3: Generate presigned URLs
    F-->>W: Return upload URLs
    W->>S3: PUT images/videos
    W->>F: POST /property-add (form + keys)
    F->>DB: Insert Property, Rooms, Flatmates
    F->>C: Async task: compute NearbyStations
    C->>Maps: Get stations & travel times
    C->>DB: Update property.NearbyStations

    Note over L: Monetize & boost
    L->>W: Toggle premium ad / boost
    W->>F: POST property-add(PremiumAd) / boost
    F->>P: Create Stripe session
    P-->>F: Checkout session id
    F-->>W: Redirect to Stripe
    P-->>F: Webhook payment success
    F->>DB: Mark premium, set BoostExpiration

    Note over Admin: Governance & quality
    Admin->>F: Login to Flask-Admin
    F->>DB: Manage properties, users, ads, flags
    C->>R: Scheduled reminders\n(unread messages, expiring ads)
```

---

### Results / impact

- **For renters**: Faster discovery of relevant rentals in London via commute-aware search, lifestyle filters, and surfaced “most popular” listings, reducing time and friction to find a room or flatshare.
- **For landlords**: Self‑serve listing, premium promotion, and performance analytics (views, saves, messages, calls) help maximize occupancy and justify marketing spend without relying on agents.
- **For the business**: SEO‑optimized landing paths, deep area coverage, and paid premium/boosted listings generate recurring revenue while building defensible search traffic and a rich dataset of demand and pricing signals.

