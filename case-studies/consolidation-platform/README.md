## Problem description

- Multi-entity finance teams need to consolidate subsidiaries into a parent, align charts of accounts, manage FX, and produce trial balance/balance sheet views with proper posting/reversal controls.
- They need secure self-service onboarding (parent + child companies), segment-driven charts of accounts, mapping between local and consolidation accounts, and auditable transaction workflows.

---

## Architecture

<p align="center">
  <img src="./images/architecture-diagram.svg" alt="Consolidation system architecture diagram" />
</p>

---

## UI Screens

<p align="center">
  <img src="./images/ui-screens.svg" alt="Consolidation UI screens" />
</p>

---

## Workflow

<p align="center">
  <img src="./images/workflow.svg" alt="Consolidation workflow sequence diagram" />
</p>


---

## Results / Impact

- Faster onboarding: parent + child companies scaffolded with default segments and users.
- Reduced reconciliation time via enforced mappings between local and consolidation segments.
- FX-ready numbers through managed conversion rates per company/currency/date.
- Auditability: posting/reversal flags, creator/updater tracking, and secure cookie/RSA-based auth.
- Decision-ready reports: balance sheet and trial balance views aligned to the consolidated chart.
