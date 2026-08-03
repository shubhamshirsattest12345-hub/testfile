# AI Celigo Auditing Tool — Folder & File Structure

> Every folder and file in the project with a one-line explanation of what it does.
> Verified against the files on disk on **2026-08-03**.
> Legend: 🔒 = gitignored (never committed) · 💤 = on disk but not used by the current flow.

```
Celigo Audit Claude Tool/
│
├── CLAUDE.md                          → Instructions auto-loaded by the AI agent at the start of every session
├── README.md                          → Project overview + the ABSOLUTE GUARDRAILS that make the tool read-only
├── CONTRIBUTING.md                    → Internal developer guide: hard rules, setup, conventions
├── CHANGELOG.md                       → Dated history of every change ever made to the project
├── requirements.txt                   → Python packages needed, plus a note on the Celigo CLI dependency
├── package.json                       → Node project metadata (the CLI is installed globally, not here)
├── .gitignore                         → Keeps credentials, snapshots and working files out of version control
│
├── .agent/                            → The AI agent's knowledge base — read before doing any work
│   ├── 00_ONBOARDING.md               → Start-here guide for a brand-new agent joining the project
│   ├── 01_KNOWN_ISSUES.md             → Platform quirks already hit and solved, so they are not hit twice
│   ├── 02_API_REFERENCE.md            → Celigo REST/CLI reference and the approved read-only access path
│   ├── 03_AUDIT_PLAYBOOK.md           → The audit itself: scoring model, report section map, formatting rules
│   ├── 04_ARCHITECTURE.md             → How the read-only enforcement layers and data flow fit together
│   ├── 05_TROUBLESHOOTING.md          → Error reference (currently a stub)
│   └── 06_DATA_SCOPE.md               → FROZEN list of what data is readable, blocked, or client-supplied
│
├── .claude/
│   ├── settings.json                  → Project automation: the session-start reminder hook + allowed commands
│   └── settings.local.json            → Local machine permission overrides (not shared)
│
├── config/
│   ├── account.example.json           → Template for the credential file — copy this, then fill it in
│   ├── account.json              🔒   → The live Celigo API token and connection settings (single source of truth)
│   ├── thresholds.json                → Configurable score-band thresholds
│   ├── manual-inputs/                 → Where the client drops values the API cannot provide (screenshots/exports)
│   │   └── README.md                  → Explains what to place here; anything missing renders NA, never a guess
│   ├── precautions/                   → Standing rules the agent must follow
│   │   ├── 01_making_changes.md       → The checklist to follow before and while making any change
│   │   └── 02_full_project_update.md  → The checklist for saving work properly into the project record
│   ├── WorkStartWithThis/
│   │   └── EveryDayStartPromp.md      → The read-only "day start" ritual that re-verifies the project state
│   └── work/                     🔒   → All temporary working data (never the deliverable)
│       ├── snapshot_DD_MM_YYYY/       → One read-only snapshot of the Celigo account — the audit's raw input
│       └── celigo-proxy-access.log    → Log of every call made to Celigo (token always redacted)
│
├── docs/
│   └── credential-setup.md            → Step-by-step guide to obtaining and configuring the Celigo API token
│
├── scripts/                           → Everything that talks to Celigo lives here
│   ├── CeligoAuth.psm1                → THE GATEKEEPER: forces read-only mode and blocks every change command
│   ├── celigo-readonly-proxy.mjs      → Holds the token and forwards read requests only — rejects all changes
│   ├── proxy.ps1                      → Starts the read-only proxy
│   ├── bootstrap.ps1                  → First-time setup: checks prerequisites, enforces read-only, tests the connection
│   ├── extract-celigo.ps1             → Takes the one read-only snapshot of the account
│   ├── query.ps1                      → Runs a single ad-hoc read-only query against the account
│   ├── run-audit.ps1             💤   → Audit runner — the read-only preflight works, orchestration not built yet
│   ├── run-report.ps1            💤   → Placeholder for a logged report-run wrapper
│   ├── generate-report.ps1       💤   → Placeholder for a report-generation wrapper
│   └── check-session-archive.mjs      → Reminds the agent at day-start if recent work was not saved to the record
│
├── audit-docx-tool/                   → Turns audit data into the finished Word report
│   ├── README.md                 💤   → Tool documentation (still describes the older NetSuite version)
│   ├── SKILL.md                  💤   → Agent instructions for the tool (also NetSuite-era)
│   ├── build.ps1                 💤   → Launcher for the old NetSuite build chain
│   ├── input/                    💤   → Input folder used by the old NetSuite build chain
│   │
│   ├── templates/
│   │   ├── audit_report_template_v03.docx           → The master design source: cover, fonts, colours, header/footer, logo
│   │   ├── celigo_audit_report_v01_source.md        → Written content of the sample/illustrative report
│   │   └── celigo_audit_report_template_v01.docx    → The built sample report (illustrative data, no client data)
│   │
│   ├── scripts/
│   │   ├── build_celigo_template.py   → THE CONVERTER: turns the written report into a styled Word document
│   │   ├── office/                    → Bundled helpers that safely open, validate and re-zip Word files
│   │   ├── generate_fresh_values.py 💤 → Old NetSuite data generator
│   │   ├── build_audit_docx.py      💤 → Old NetSuite report builder
│   │   ├── check_static_data.py     💤 → Old anti-fabrication lint
│   │   ├── smoke_test.py            💤 → Old build regression test
│   │   ├── inject_placeholders.py   💤 → Old template placeholder injector
│   │   └── auto_build_hook.ps1      💤 → Old auto-build hook (not connected)
│   │
│   └── live-prototype/                → The live audit engine — reads the snapshot, never the account
│       ├── README.md                              → Explains each module and which are current vs superseded
│       ├── gen-live-report-full.mjs               → THE GENERATOR: analyses the snapshot, scores it, writes the report
│       ├── build-live-docx-full.py                → Renders that report into the final Word document
│       ├── derive-run-history.mjs                 → Works out the real run window, success rate and job retention limit
│       ├── derive-error-mttr.mjs                  → Works out the open error backlog and average time to resolve
│       ├── derive-connection-usage.mjs            → Works out which connections are actually used, and which are orphaned
│       ├── derive-export-classification.mjs       → Classifies how each data source pulls its records
│       ├── derive-user-security.mjs               → Works out the real SSO / MFA posture and who has admin access
│       ├── derive-resource-classification.mjs     → Separates genuine production resources from test/disabled/orphaned ones
│       ├── derive-subscription-usage.mjs          → Works out licence and processing-time usage
│       ├── derive-stale-flows.mjs                 → Finds flows that are switched on but have not run in a long time — or ever
│       ├── fetch-review-scripts.mjs               → Reviews each custom script and rates it (never stores the code)
│       ├── fetch-connection-usage.mjs             → Asks Celigo what depends on each connection
│       ├── fetch-flow-errors.mjs                  → Collects error timestamps for the MTTR calculation (never the error content)
│       ├── fetch-subscription-usage.mjs           → Collects processing-time usage figures
│       ├── gen-live-report.mjs               💤   → The original 2026-07-13 prototype generator, kept for history
│       └── build_live_docx.py                💤   → Its original renderer, kept for history
│
├── reports/                           → The client deliverable — nothing else belongs here
│   ├── Celigo_Audit_Report_LIVE_DD_MM_YYYY.docx   → The finished audit report for the connected account
│   └── logs/                     🔒   → Run transcripts
│
└── project-complete-life-cycle-claude-session/    → Full history of how the project was built
    ├── README.md                                  → Session index: what each session did, decided and fixed
    └── session_NN_DD_MM_YYYY_summary.jsonl        → One redacted transcript per session (15 to date)
```

### The five things worth knowing about this structure

1. **`scripts/` is the only place that talks to Celigo** — and everything in it goes
   through `CeligoAuth.psm1` and the read-only proxy, so no other folder can reach the
   account at all.
2. **`config/work/` holds the raw data, `reports/` holds only the deliverable.** The
   snapshot is working data and is never shipped; `reports/` stays clean.
3. **Nothing sensitive is ever committed** — the token, the snapshots and any client
   exports are all gitignored (🔒).
4. **`audit-docx-tool/` has two halves:** `live-prototype/` produces the real client
   report from live data, while `templates/` + `scripts/build_celigo_template.py`
   produce the styled document and the illustrative sample.
5. **The 💤 items are inherited from the earlier NetSuite version** of this tool. They are
   kept deliberately for reference but are not part of the current process.
