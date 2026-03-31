# Engineering case studies

Short, structured write-ups of real systems I’ve designed and shipped. Each case study follows: **Problem → Solution → Architecture → Tech stack → Results**.

---

## Index

| Case study | Domain | Tech stack (short) | Highlights |
| --- | --- | --- | --- |
| [Realistic Traffic Generator](./realistic-traffic-generator/) | Marketing automation / growth | Python, Tornado, Selenium, MySQL | Multi-tenant, human-like traffic & signup automation with strict per-site limits and observability. |
| [AllFresh Farmhouse Ordering](./allfresh-farmhouse-ordering/) | Inventory & supply chain | Angular, Python backend, MSSQL | Internal ordering, purchasing, receiving, and stock control across business units and virtual warehouses. |
| [Atlas Logging API](./atlas-logging-api/) | Healthcare ops / AI triage | Flask, Azure Cosmos DB, Twilio, OpenAI | Lead intake + SMS automation with triage rules and an Atlas work queue that explains “why this next”. |
| [Harvest Reporting](./harvest-reporting/) | Legal analytics & compensation | FastAPI, React, MariaDB, Harvest API | Reporting + bonus/true-up engine on top of synced Harvest data for attorneys and firm leadership. |
| [Instamove](./instamove-logistics-platform/) | Rentals marketplace | Flask, React, PostgreSQL, Stripe, S3 | London-focused rental marketplace with commute-aware search, landlord tools, and premium listings. |
| [ESG Reporting Platform](./esg-reporting-platform/) | ESG / compliance | React, Supabase, Postgres, object storage | Two-sided ESG portal for owners and suppliers, with auth, onboarding, and secure document uploads. |
| [Consolidation Platform](./consolidation-platform/) | Finance / consolidation | Angular, Tornado, PostgreSQL | Multi-entity financial consolidation: segments, mappings, FX, and balance sheet/trial balance reporting. |
| [Ticketing & Support System](./ticketing-and-support-system/) | Support / operations | Angular, backend API, email/SMS | Multi-portal ticketing with allocation, SLAs, omni-channel replies, and automated reminders. |

---

## Reading order

If you only have time for a few:

- Start with **Realistic Traffic Generator** and **Atlas Logging API** for system design, automation, and observability.
- Then skim **AllFresh Farmhouse Ordering** or **Harvest Reporting** for data modelling and workflow-heavy backoffice work.

Each file stands alone; follow the internal Mermaid diagrams for architecture and request flows.

