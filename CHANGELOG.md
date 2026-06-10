# Changelog

All notable changes to this project are documented here.

---

## [1.0.0] — 2026-06-09

### Added
- Automated cloud flow triggered on every new Outlook email (Office 365 Outlook — V3 connector)
- Azure OpenAI GPT-4.1-mini classification via raw HTTP action (`api-version=2025-01-01-preview`)
- Four-category email classification: Urgent, Action Required, FYI, No Action
- Structured JSON response schema: `classification`, `reason`, `suggested_action`
- Parse JSON step to schema and type the Azure OpenAI API response
- Compose step (`Extract Classification`) to parse GPT content string into a typed object
- Condition routing with OR logic: Urgent / Action Required → Teams alert; FYI / No Action → Terminate
- Teams Adaptive Card delivery via Power Automate HTTP trigger webhook (FactSet layout, AdaptiveCard v1.2)
- Terminate action on False branch for clean run history
- Step-by-step build guide in `docs/build-guide.md` including troubleshooting table
- Architecture documentation with Mermaid flow diagram and infrastructure table
