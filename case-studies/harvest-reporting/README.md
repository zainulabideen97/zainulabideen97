### Problem description

Harvest Reporting MVP is a reporting and compensation tool for law firms that use Harvest for time tracking. The core problem it solves is that Harvest’s native reports don’t answer key questions for firm leadership and attorneys:

- **For attorneys**: How am I tracking against my quarterly and annual bonus targets (billable, invoiced, and collected)? How much potential bonus is “locked” in uninvoiced or uncollected work?
- **For management**: Where is revenue stuck in the pipeline (uninvoiced / uncollected, by client and project, with aging)? Which attorneys are on/off pace for targets? How do we generate consistent PDF/summary reports for attorneys and leadership?
- **For operations**: How do we keep an internal analytics database incrementally synced with Harvest (time entries, invoices, payments, users, projects) in a robust, observable way?

The MVP introduces a dedicated backend that syncs Harvest data into a MariaDB warehouse, exposes higher-level reporting and bonus/true‑up calculations, and a front‑end that surfaces dashboards for attorneys and management.

---

### Architecture diagram

<p align="center">
  <img src="./images/architecture-diagram.svg" alt="Harvest Reporting system architecture diagram" />
</p>

---

### UI screens

<p align="center">
  <img src="./images/ui-screens.svg" alt="Harvest Reporting UI screens" />
</p>

---

### Workflow

<p align="center">
  <img src="./images/workflow.svg" alt="Harvest Reporting workflow sequence diagram" />
</p>

---

### Results / impact

- **Transparency for attorneys**: Each attorney gets a clear QTD and YTD view of billed, invoiced, uninvoiced, collected, and not collected work, plus how much bonus is locked in each stage of the pipeline.
- **Actionable revenue pipeline**: Management sees aging of uninvoiced and uncollected amounts by client/project, helping prioritize invoicing and collections to unlock revenue and bonuses.
- **Robust, incremental sync from Harvest**: A dedicated sync runner uses checkpoints and structured logging to keep a MariaDB analytics store current, enabling all dashboards and reports to read from a single, consistent source of truth.
- **Bonus & true-up automation**: Quarterly and annual bonus tiers, Q4 true-up calculations, and recommended payment amounts are computed from actual collections and prior payouts, reducing manual spreadsheet work and payment errors.
