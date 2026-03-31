### Problem description

**Atlas Logging API** is a Flask-based service that ingests and manages patient leads and SMS conversations for a medical practice.
It centralizes data in Azure Cosmos DB, uses OpenAI to generate safe SMS replies, Twilio for messaging, and an Atlas work queue plus triage rules engine to:

- **Ingest leads** from website/partners/Meta and unify them into a single lead record.
- **Automate SMS conversations** while safely escalating urgent cases and honoring declines.
- **Prioritize staff work** with a derived queue (priority, state, next action, “why Atlas chose this”).
- **Provide auditability** via communication logs, event stream, and daily summary emails.

---

### Architecture diagram (Mermaid)

```mermaid
flowchart LR
    subgraph Clients
        WebLead[Partner / Website\nLead Sources]
        Meta[Meta Lead Ads]
        PatientSMS[Patient via SMS]
        StaffUI[Staff Browser UI]
        AdminOps[Admin / Ops]
    end

    subgraph AtlasLoggingAPI[Atlas Logging API (Flask)]
        AppFactory[App Factory\ncreate_app()]
        AuthBP[auth_routes\n(login, 2FA)]
        IndexBP[index_routes\n(leads, archived)]
        AtlasBP[atlas_routes\n(work queue, timeline)]
        LeadBP[lead_routes\n(/lead, /action, /meta)]
        LogBP[log_routes\n(/summary, /get_logs)]
        TwilioBP[twilio_routes\n(/webhook/sms)]
        AdminBP[admin_routes\n(daily summary)]
        subgraph AtlasCore[Atlas Core Logic]
            QueueMod[atlas.queue\n(state & scoring)]
            RulesMod[atlas.rules\n(triage rules engine)]
            EventsMod[atlas.events\n(event system)]
        end
        subgraph Utils[Utilities]
            OpenAIHelper[openai_helper\n(gpt-4o, gpt-4o-mini)]
            Tools[tools, filter_pipeline,\nresponse, sms_helper]
        end
    end

    subgraph DataStores[Azure Cosmos DB (SQL API)]
        Users[users]
        Leads[leads]
        Logs[log_entries]
        CommLogs[communication_logs]
        Webhooks[webhook_hits]
        DailySummary[daily_summary_log]
        AtlasEvents[atlas_events]
        AtlasState[atlas_lead_state]
    end

    subgraph ExternalServices[External Services]
        TwilioSMS[Twilio SMS]
        OpenAI[gpt-4o / gpt-4o-mini]
        MetaGraph[Meta Graph API]
        ACSEmail[Azure Comm Svcs Email]
    end

    %% Flows
    WebLead -->|POST /lead (API key)| LeadBP
    Meta -->|Webhook /leads/meta|\nLeadBP
    PatientSMS -->|Incoming SMS\nTwilio Webhook| TwilioSMS --> TwilioBP
    StaffUI -->|HTTP + Sessions| AuthBP & IndexBP & AtlasBP & LogBP
    AdminOps -->|/admin/daily-summary\n(X-Admin-Token)| AdminBP

    %% API to core
    LeadBP --> Leads
    LeadBP --> CommLogs
    LeadBP --> QueueMod
    LeadBP --> Webhooks

    TwilioBP --> RulesMod
    TwilioBP --> OpenAIHelper
    TwilioBP --> CommLogs
    TwilioBP --> QueueMod
    TwilioBP --> EventsMod

    AtlasBP --> QueueMod
    AtlasBP --> AtlasState
    AtlasBP --> AtlasEvents
    AtlasBP --> Leads
    AtlasBP --> CommLogs
    AtlasBP --> Logs

    LogBP --> Logs
    IndexBP --> Leads
    IndexBP --> CommLogs

    AdminBP --> Leads
    AdminBP --> DailySummary
    AdminBP --> ACSEmail

    %% Utilities and events
    EventsMod --> AtlasEvents
    QueueMod --> AtlasState
    OpenAIHelper --> OpenAI
    LeadBP --> MetaGraph
    TwilioBP --> TwilioSMS
```

---

### UI screens (Mermaid)

```mermaid
flowchart LR
    Login[Login Screen\n(username + password)]
    Setup2FA[2FA Setup\n(QR code, secret)]
    Verify2FA[2FA Verify\n(OTP input)]

    LeadsUI[Leads List\n(filters, pagination)]
    ArchivedUI[Archived Leads\n(archive/unarchive)]
    AtlasQueueUI[Atlas Work Queue\n(priority, why, state, automation)]
    LeadDetailModal[Lead / Case Detail Modal\n(timeline, events, state)]

    DailySummaryOps[Daily Summary Trigger\n(admin-only action)]

    %% Auth flow
    Login -->|Valid credentials| Setup2FA
    Setup2FA -->|Enable 2FA| Verify2FA
    Verify2FA -->|Success| LeadsUI

    %% Navigation from header/menu
    LeadsUI -->|Nav: "Archived Leads"| ArchivedUI
    LeadsUI -->|Nav: "Atlas Queue"| AtlasQueueUI
    ArchivedUI -->|Nav: "Leads"| LeadsUI
    AtlasQueueUI -->|Nav: "Leads"| LeadsUI

    %% In-queue interactions
    AtlasQueueUI -->|Click "View"| LeadDetailModal
    LeadDetailModal -->|Clear automation| AtlasQueueUI

    %% Admin
    LeadsUI -->|Admin calls /admin/daily-summary| DailySummaryOps
```

---

### Workflow (Mermaid)

```mermaid
sequenceDiagram
    participant Partner as Partner / Website
    participant Meta as Meta Lead Ads
    participant Twilio as Twilio SMS
    participant Patient as Patient
    participant API as Atlas Logging API
    participant Queue as Atlas Queue System
    participant Rules as Triage Rules Engine
    participant Events as Event System
    participant OpenAI as OpenAI GPT
    participant DB as Cosmos DB
    participant Staff as Staff (UI)

    rect rgb(240,240,255)
    Note over Partner,API: New lead creation
    Partner->>API: POST /lead (JSON + API key)
    API->>DB: Insert into leads & log_entries
    API->>Twilio: Send initial SMS via Twilio
    Twilio-->>Patient: Outbound SMS from practice
    API->>Queue: Compute atlas_lead_state (priority, state, next_action)
    Queue->>DB: Upsert atlas_lead_state
    end

    rect rgb(240,255,240)
    Note over Meta,API: Meta lead webhook
    Meta->>API: Webhook (leadgen_id)
    API->>Meta: GET lead details via Graph API
    API->>DB: Upsert lead + webhook_hits
    API->>Queue: Recompute atlas_lead_state
    end

    rect rgb(255,245,240)
    Note over Patient,API: Incoming SMS with triage
    Patient->>Twilio: SMS to office number
    Twilio->>API: POST /webhook/sms
    API->>DB: Find/create lead by phone
    API->>Rules: Load & apply triage rules\n(GPT triage + keyword fallback)
    alt Urgent rule
        Rules->>Events: Emit triage_rule_fired (urgent)
        Events->>DB: Write atlas_events
        API->>DB: Update lead.status = "Needs Review"
        API->>Queue: Recompute atlas_lead_state (urgent + locked)
        API->>Patient: Send deterministic urgent response via Twilio
    else Decline / Scheduling / Form ACK
        API->>DB: Update lead.status\n(Resolved or Waiting On Staff)
        API->>Queue: Recompute atlas_lead_state
        API->>Patient: Send template response via Twilio
    else No matching rule and not locked
        API->>OpenAI: generate_with_context\n(conversation history)
        OpenAI-->>API: JSON {draft_reply, intent, urgency}
        API->>DB: Log inbound/outbound SMS in communication_logs
        API->>Queue: Recompute atlas_lead_state
        API->>Patient: Send Atlas-generated SMS via Twilio
    else Automation locked
        API->>Patient: Short holding reply\n(or no reply)
    end
    end

    rect rgb(245,245,255)
    Note over Staff,DB: Staff queue management
    Staff->>API: GET /atlas/queue
    API->>DB: Read atlas_lead_state + leads
    API-->>Staff: Atlas Queue UI (priority, why, next action)
    Staff->>API: View case detail (modal)
    API->>DB: Read lead + events + logs + SMS thread
    API-->>Staff: Timeline + derived state
    opt Staff clears urgent case
        Staff->>API: POST /atlas/leads/<id>/clear
        API->>Events: Emit human_cleared_case
        Events->>DB: Write atlas_events
        API->>Queue: Recompute atlas_lead_state (unlocked)
    end
    end

    rect rgb(255,250,240)
    Note over Staff,ACSEmail: Daily summary
    Staff->>API: POST /admin/daily-summary\n(with X-Admin-Token)
    API->>DB: Query leads by after-hours window
    API->>ACSEmail: Send daily summary email
    API->>DB: Upsert daily_summary_log\n(idempotency)
    end
```

---

### Results / impact

- **Reduced missed urgent cases**: Urgent triage rules plus an event-driven automation lock ensure potentially dangerous messages are immediately surfaced and not auto-replied by AI.
- **Higher staff efficiency**: The Atlas work queue compresses multi-table data (leads, SMS, events, logs) into a single prioritized list with “why” explanations and concrete next actions.
- **Consistent patient communication**: OpenAI-backed SMS replies, templated responses for key scenarios, and audit logs in `communication_logs` standardize tone and reduce manual drafting.
- **Better visibility and reporting**: Per-lead timelines, summary endpoints, and daily summary emails give operators a clear picture of volume, status, and outcomes across channels.

