## Problem description

The project is a **ticketing and support system** for managing client issues across multiple portals (client and support). It aims to:

- **Centralize client requests** into tickets instead of ad‑hoc emails/calls.
- **Track technician availability** and **allocate techs** to client tickets efficiently.
- **Ensure no ticket is left unattended**, with automatic email notifications if no one responds.
- **Enable omni‑channel responses**, allowing ticket replies via email and phone.
- **Capture and maintain accurate contact information** for both clients and technicians (email and phone) to support these workflows.

Overall, it reduces response delays, makes ownership clear, and provides a consistent, auditable support experience.

---

## Architecture diagram (mermaid)

```mermaid
flowchart LR
    subgraph Clients
        CWeb[Client Portal (Angular)]
        CEmail[Client Email]
        CPhone[Client Phone Call]
    end

    subgraph Support
        SWeb[Support Portal (Angular)]
        Techs[Support Technicians]
    end

    subgraph Backend["Ticketing Web Server"]
        API[REST/GraphQL API]
        Auth[Auth & Roles]
        Tickets[Ticket Service]
        Users[User & Contact Service]
        Alloc[Tech Allocation Engine]
        SLA[SLA & Reminder Engine]
        Msg[Message Composer]
        DB[(Relational DB\nTickets, Users, Messages,\nAssignments, Availability)]
    end

    subgraph Integrations
        EmailSvc[Email Service\n(ticketing-email.service)]
        SMS[Phone/SMS Gateway]
    end

    CWeb -->|Create/View Tickets| API
    SWeb -->|Manage Tickets & Clients| API

    API --> Auth
    API --> Tickets
    API --> Users
    Tickets --> Alloc
    Tickets --> SLA
    Tickets --> Msg
    Alloc --> DB
    Tickets --> DB
    Users --> DB
    SLA --> DB
    Msg --> DB

    SLA -->|Unanswered ticket alerts| EmailSvc
    Msg -->|Ticket replies| EmailSvc
    Msg -->|Ticket replies| SMS

    EmailSvc --> CEmail
    SMS --> CPhone

    Techs <-->|Use Support Portal| SWeb
```

---

## UI screens (mermaid diagram)

```mermaid
flowchart TD
    subgraph ClientPortal["Client Portal"]
        CP_Dash[Client Dashboard\n- Open tickets summary\n- Status widgets]
        CP_TicketsList[Tickets List\n- Filter: All / Open / Closed\n- Filter: Unassigned\n- Search & Sort]
        CP_TicketDetails[Ticket Details\n- Conversation thread\n- Attachments\n- Assigned technician\n- SLA/last response]
        CP_NewTicket[New Ticket Form\n- Subject, Description\n- Category, Priority\n- Contact email/phone\n- Attachments]
        CP_Profile[Client Profile\n- Organization info\n- Contact preferences]
    end

    subgraph SupportPortal["Support Portal"]
        SP_Dash[Support Dashboard\n- Ticket queues\n- Unassigned tickets widget\n- SLA at-risk]
        SP_TicketsList[Ticket Queue\n- Filters: Unassigned / My tickets / Team\n- Status, Priority, Client]
        SP_TicketDetailsS[Ticket Details\n- Assign/Unassign technician\n- Set status & priority\n- Internal notes\n- Reply by email/phone]
        SP_Clients[Clients List\n- Search clients\n- Client details]
        SP_ClientForm[Client Form\n- Edit client info\n- Contact email/phone]
        SP_Settings[Tech Availability & Routing\n- Availability status\n- Skills / groups]
    end

    CP_Dash --> CP_TicketsList
    CP_TicketsList --> CP_TicketDetails
    CP_Dash --> CP_NewTicket
    CP_TicketDetails --> CP_NewTicket
    CP_Dash --> CP_Profile

    SP_Dash --> SP_TicketsList
    SP_TicketsList --> SP_TicketDetailsS
    SP_Dash --> SP_Clients
    SP_Clients --> SP_ClientForm
    SP_Dash --> SP_Settings
```

---

## Workflow (mermaid)

### End‑to‑end ticket lifecycle

```mermaid
sequenceDiagram
    actor Client
    participant CPortal as Client Portal
    participant SPortal as Support Portal
    participant API as Ticketing API
    participant Alloc as Allocation Engine
    participant SLA as SLA & Reminder
    participant Email as Email Service
    participant SMS as Phone/SMS

    Client->>CPortal: Create new ticket\n(subject, description, contact info)
    CPortal->>API: POST /tickets
    API->>Alloc: Evaluate available technicians\n(by skills, availability, client)
    Alloc-->>API: Selected technician (or none)
    API->>API: Create ticket + assignment in DB
    API->>Email: Send ticket created email\n(to client + assigned tech)
    API->>SPortal: Push to ticket queue view

    Note over SLA,API: Start SLA / no‑response timers

    SPortal->>Tech: Tech sees new/unassigned tickets\n(via dashboard & filters)
    Tech->>SPortal: Open ticket, update status,\nadd reply (email/phone)
    SPortal->>API: POST /tickets/{id}/messages
    API->>Email: Send reply email to client
    API->>SMS: Optionally send reply/notification SMS
    API->>SLA: Register activity (response time)

    SLA->>SLA: Check timer: has anyone responded?
    alt No response within threshold
        SLA->>Email: Send reminder to tech / team\n(or escalate)
    end

    Client->>CEmail: Reads email / replies via email
    CEmail->>Email: Reply received
    Email->>API: Webhook / inbound email\n-> new message on ticket
    API->>SPortal: Update ticket conversation
    API->>CPortal: Update ticket details

    Tech->>SPortal: Resolve ticket
    SPortal->>API: PATCH /tickets/{id}\nstatus = Resolved/Closed
    API->>Email: Send resolution email & feedback link
```

---

## Results / impact

- **Faster response and resolution times**: Automated routing based on technician availability and reminders for unanswered tickets reduce idle time and prevent tickets from “falling through the cracks.”
- **Higher visibility and accountability**: Dashboards for both clients and support teams, plus clear assignment and status, make ownership obvious and support performance measurable.
- **Improved client experience**: Clients can submit and track tickets via the portal and receive consistent updates by email/phone, increasing trust and satisfaction.
- **Operational efficiency**: Centralized ticket and contact data, structured workflows, and integrated email/phone communications reduce manual coordination, duplicated effort, and miscommunication.

