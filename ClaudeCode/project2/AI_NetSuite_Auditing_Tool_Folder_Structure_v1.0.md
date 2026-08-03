# AI NetSuite Auditing Tool — Folder & File Structure

> Every folder and file in the project with a one-line explanation of what it does.
> Verified against the files on disk on **2026-08-03**.
> Legend: 🔒 = gitignored (never committed) · 💤 = on disk but not used by the current flow.

```
NetSuite Audit AI/
│
├── CLAUDE.md                          ← Instructions auto-loaded by the AI agent at the start of every session
├── README.md                          ← Human quick-start guide
├── CONTRIBUTING.md                    ← Internal developer guide: hard rules, setup, conventions
├── CHANGELOG.md                       ← Dated history of every change ever made to the project
├── requirements.txt                   ← Python packages needed (lxml, defusedxml)
├── package.json                       ← Node tooling: the SDF CLI and the JS obfuscator
├── .gitignore                         ← Keeps credentials, snapshots, working files and reports out of version control
│
├── .agent/                            ← The AI agent's knowledge base — read before doing any work
│   ├── 00_ONBOARDING.md               ← Start-here guide for a brand-new agent joining the project
│   ├── 01_KNOWN_ISSUES.md             ← Platform quirks already hit and solved, so they are not hit twice
│   ├── 02_SUITEQL_REFERENCE.md        ← Which SuiteQL tables are reachable, and the query patterns that work
│   ├── 03_AUDIT_PLAYBOOK.md           ← The audit itself: every check, its query, the scoring model, the section map
│   ├── 04_ARCHITECTURE.md             ← How the scripts, result files and report pipeline fit together
│   └── 05_TROUBLESHOOTING.md          ← Error reference with a quick-lookup table of every error ever hit
│
├── .claude/
│   ├── settings.json                  ← Project automation: the two hooks (prompt-trigger + auto-build)
│   └── settings.local.json       🔒   ← Local machine permission overrides (not shared)
│
├── config/
│   ├── account.example.json           ← Template for the credential file — copy this, then fill it in
│   ├── account.json              🔒   ← The live account ID, TBA tokens, SDF passkey and Java path (single source of truth)
│   ├── thresholds.json                ← Configurable audit thresholds
│   ├── certs/                    🔒   ← The RSA-4096 key pair used for SDF machine-to-machine deploy auth
│   │   ├── auditai_admin_cert.pem     ← Public certificate uploaded to NetSuite
│   │   ├── auditai_admin_private.pem  ← Private key used by the SDF CLI
│   │   └── _old_.../             💤   ← The superseded pair from before the sandbox refresh, kept for history
│   ├── manual-inputs/                 ← Where the operator drops the values no API can provide
│   │   ├── README.md                  ← The canonical checklist of what to place here; anything missing renders NA
│   │   ├── InstalledBundles*.csv 🔒   ← Bundle inventory export — drives the bundle-currency check
│   │   ├── BundleAuditTrail*.csv 🔒   ← Bundle update history export — supplies each bundle's latest version
│   │   ├── Integrations*.csv     🔒   ← Manage Integrations export — the integration table is sealed to every API
│   │   └── UIScripts_*.txt       🔒   ← UI Scripts list export — cross-verifies the per-type script counts
│   ├── precautions/                   ← Standing rules the agent must follow
│   │   ├── 01_making_changes.md       ← The checklist to follow before and while making any change
│   │   └── 02_full_project_update.md  ← The checklist for saving work properly into the project record
│   ├── WorkStartWithThis/
│   │   └── EveryDayStartPromp.md      ← The read-only "day start" ritual that re-verifies the project state
│   └── work/                     🔒   ← All temporary working data (never the deliverable)
│       ├── README.md                  ← Explains what belongs here and why it stays out of reports/
│       ├── Workflow_Action_Audit_*.json/.md  ← Output of the workflow action deep-dive, consumed by the generator
│       ├── CustomForm_Defs_*.json     ← Output of the custom-form definition deep-dive, consumed by the generator
│       └── Suitelet_SuiteQL_active_list.csv  ← A validation export kept from an earlier verification pass
│
├── docs/
│   └── credential-setup.md            ← Step-by-step guide to the TBA tokens, the SDF certificate and the role permissions
│
├── SDF/                               ← The audit engine that gets installed into NetSuite
│   ├── AuditAI/
│   │   ├── suitecloud.config.js       ← SDF CLI project configuration
│   │   ├── project.json          🔒   ← Which auth ID to deploy with (account-specific)
│   │   ├── .gitignore                 ← SDF's own ignore rules
│   │   └── src/
│   │       ├── manifest.xml           ← Declares the SERVERSIDESCRIPTING feature the scripts depend on
│   │       ├── deploy.xml             ← Tells SDF to deploy everything under FileCabinet/ and Objects/
│   │       ├── Objects/               ← 28 XML definitions: 24 Script+Deployment records + 4 saved searches
│   │       │   ├── customscript_audit_financial.xml        ← Financial anomaly audit: script record + daily schedule
│   │       │   ├── customscript_audit_security.xml         ← Security audit: script record + daily schedule
│   │       │   ├── customscript_audit_script_perf.xml      ← Script performance audit
│   │       │   ├── customscript_audit_sysconfig.xml        ← System configuration audit (Map/Reduce)
│   │       │   ├── customscript_audit_ui_perf.xml          ← UI performance audit (Map/Reduce)
│   │       │   ├── customscript_audit_sod.xml              ← Segregation-of-duties audit (weekly)
│   │       │   ├── customscript_audit_integration.xml      ← Integration health audit
│   │       │   ├── customscript_audit_bundle.xml           ← Installed-bundle inventory (weekly)
│   │       │   ├── customscript_audit_phd.xml              ← Process-health diagnostics
│   │       │   ├── customscript_audit_company_config.xml   ← Password policy / IP restriction audit
│   │       │   ├── customscript_audit_script_log.xml       ← Execution-log scan (Map/Reduce)
│   │       │   ├── customscript_audit_saved_search.xml     ← Saved-search efficiency + join complexity (Map/Reduce)
│   │       │   ├── customscript_audit_login_monitor.xml    ← Login failure-rate monitor
│   │       │   ├── customscript_audit_sched_freq.xml       ← Scheduled-cadence audit (Map/Reduce)
│   │       │   ├── customscript_audit_workflow.xml         ← Per-workflow configuration audit (Map/Reduce)
│   │       │   ├── customscript_audit_analytics.xml        ← Dataset + workbook audit (Map/Reduce)
│   │       │   ├── customscript_audit_pageperf.xml         ← APM page-timing capture (Map/Reduce)
│   │       │   ├── customscript_audit_deployer.xml         ← The Deployer RESTlet: the tool's single entry point
│   │       │   ├── customscript_audit_dashboard.xml        ← In-account health dashboard Suitelet
│   │       │   ├── customscript_audit_realtime.xml         ← Real-time alert monitor (User Event)
│   │       │   ├── customscript_audit_trend_logger.xml     ← Writes score history to an AuditAI custom record
│   │       │   ├── customscript_audit_export_email.xml     ← Emails the daily result summary
│   │       │   ├── customscript_audit_log_reader.xml       ← Execution-log reader helper
│   │       │   ├── customscript_audit_deal_risk.xml        ← Deal-risk scoring RESTlet
│   │       │   ├── customsearch_ai_audit_serverscript_log.xml  ← Saved search: server script log (loaded by scriptid)
│   │       │   ├── customsearch_ai_audit_script_count.xml      ← Saved search: all scripts detail
│   │       │   ├── customsearch_ai_audit_script_count_2.xml    ← Saved search: script execution detail
│   │       │   └── customsearch_ai_audit_dep.xml                ← Saved search: deployment + execution
│   │       └── FileCabinet/SuiteScripts/AuditAI/   ← The 24 SuiteScript source files (8 of them Map/Reduce)
│   │           ├── ns_audit_deployer_rl.js             ← THE ENTRY POINT: triggers scripts and returns result files
│   │           ├── ns_audit_financial_anomaly_ss.js    ← Finds financial anomalies in transactions
│   │           ├── ns_audit_security_ss.js             ← Audits roles, admin accounts, 2FA and batch user creation
│   │           ├── ns_audit_script_performance_ss.js   ← Audits script and deployment performance
│   │           ├── ns_audit_system_config_ss.js        ← Audits system configuration (Map/Reduce)
│   │           ├── ns_audit_ui_performance_ss.js       ← Audits client-script density per record type (Map/Reduce)
│   │           ├── ns_audit_sod_ss.js                  ← Detects segregation-of-duties conflicts
│   │           ├── ns_audit_integration_perf_ss.js     ← Audits integration execution health and token hygiene
│   │           ├── ns_audit_bundle_inventory_ss.js     ← Inventories installed bundles by name match
│   │           ├── ns_audit_phd_ss.js                  ← Process-health diagnostics
│   │           ├── ns_audit_company_config_ss.js       ← Reads password policy and IP restrictions
│   │           ├── ns_audit_script_log_ss.js           ← Scans the Script Execution Log (Map/Reduce, so any role can run it)
│   │           ├── ns_audit_saved_search_ss.js         ← Audits every saved search for date filters and joins (Map/Reduce)
│   │           ├── ns_audit_login_monitor_ss.js        ← Monitors login failure rates and top failing users/IPs
│   │           ├── ns_audit_sched_freq_mr.js           ← Reads each deployment's real schedule (Map/Reduce)
│   │           ├── ns_audit_workflow_mr.js             ← Loads every workflow for triggers, logging, run-as-admin (Map/Reduce)
│   │           ├── ns_audit_analytics_mr.js            ← Enumerates and loads datasets and workbooks (Map/Reduce)
│   │           ├── ns_audit_pageperf_mr.js             ← Captures APM page-load and script timing (Map/Reduce)
│   │           ├── ns_audit_trend_logger_ss.js         ← Writes each run's scores to the score-history custom record
│   │           ├── ns_audit_export_email_ss.js         ← Compiles all results and emails the summary
│   │           ├── ns_audit_log_reader_ss.js           ← Reads execution-log entries for other scripts
│   │           ├── ns_audit_health_dashboard_sl.js     ← In-account dashboard page showing the current scores
│   │           ├── ns_audit_realtime_ue.js             ← Fires on journal/bill/employee saves for real-time alerts
│   │           ├── ns_audit_deal_risk_rl.js            ← Scores deal risk on request
│   │           └── *.js.bak_folder               💤   ← 14 stale backups from the runtime-folder change; nothing references them
│   └── _backups/                     💤   ← Dated backups of the template and generator taken before past changes
│
├── scripts/                           ← Everything that talks to NetSuite from the operator's machine
│   ├── NsAuth.psm1                    ← THE SIGNER: the single OAuth 1.0a implementation all callers share
│   ├── bootstrap.ps1                  ← First-time setup: Java 21, SDF CLI, credentials, SDF auth, first deploy
│   ├── deploy.ps1                     ← THE ONLY DEPLOY PATH: wraps suitecloud project:deploy
│   ├── secure_deploy.ps1              ← Obfuscate, deploy, then restrict the File Cabinet folder (client deliveries)
│   ├── run-audit.ps1                  ← Triggers the audit scripts on demand through the Deployer RESTlet
│   ├── run-report.ps1                 ← Runs the whole report build and captures a transcript into reports/logs/
│   ├── query.ps1                      ← Runs a single ad-hoc read-only SuiteQL query
│   ├── audit-workflow-actions.ps1     ← Read-only deep-dive: imports workflows to a throwaway project for action values
│   ├── workflow_action_parser.py      ← Parses that workflow XML into the action audit the generator reads
│   ├── audit-customform-defs.ps1      ← Read-only deep-dive: imports custom forms to a throwaway project
│   ├── parse_form_xml.py              ← Parses that form XML into the field-level definition file the generator reads
│   ├── cross_reference.py        💤   ← Standalone form usage-vs-definition fuser; the generator does this inline now
│   ├── precautions_hook.ps1           ← Loads the right precautions file when the operator uses a trigger phrase
│   └── generate-report.ps1       💤   ← Deprecated Word-COM report builder with a hardcoded score; kept for history
│
├── audit-docx-tool/                   ← Turns audit data into the finished Word report
│   ├── README.md                      ← Tool documentation
│   ├── SKILL.md                       ← The full reference for all 401 placeholder keys
│   ├── build.ps1                      ← THE LAUNCHER: runs both deep-dives, then generate, then build
│   ├── input/
│   │   ├── values_<DD_MM_YYYY>.json   ← The 401 generated report values for a run (newest is auto-selected)
│   │   └── _archive/                  ← Older value snapshots kept for comparison
│   ├── examples/
│   │   └── values_*.example.json      ← A full schema example for reference
│   ├── templates/
│   │   ├── audit_report_template_v03.docx              ← The master design source: cover, fonts, colours, header, logo
│   │   ├── audit_report_template_v03_PLACEHOLDERS.docx ← THE BUILD TEMPLATE: the same document with 401 placeholders
│   │   ├── *.bak_*.docx / *.pre_*  💤 ← 50 dated template backups, one per past patch — the reversibility trail
│   │   └── temp_docx_extract/      💤 ← A leftover unpacked template from a past inspection
│   └── scripts/
│       ├── generate_fresh_values.py   ← THE GENERATOR: reads the account, computes the scores, writes the 401 values
│       ├── build_audit_docx.py        ← THE BUILDER: fills the template, runs the gates, writes the DOCX
│       ├── check_static_data.py       ← THE LINT: fails if any template data cell holds a literal instead of a placeholder
│       ├── smoke_test.py              ← Fast regression check: builds to a temp file and asserts report correctness
│       ├── inject_placeholders.py     ← One-time tool that created the placeholder template from the design source
│       ├── auto_build_hook.ps1        ← Hook logic that rebuilds the report when a values file is written
│       ├── office/                    ← Bundled OOXML toolkit: unpack, pack and schema-validate Word files
│       │   ├── unpack.py              ← Explodes a .docx into its parts
│       │   ├── pack.py                ← Rezips it and validates it against the OOXML schemas
│       │   ├── validate.py            ← Standalone validator entry point
│       │   ├── validators/            ← The per-format validators (only docx.py is used here)
│       │   ├── helpers/          💤   ← Run-merging and redline helpers not used by this build
│       │   ├── soffice.py        💤   ← LibreOffice conversion helper not used by this build
│       │   └── schemas/               ← The ISO/ECMA/Microsoft XSD set the validator checks against
│       └── maintenance/               ← One-off idempotent template patches, one per past change (the audit trail)
│           ├── patch_*.py             ← 21 patch scripts: each added, removed or relabelled part of the template
│           ├── make_*_dynamic.py      ← The patches that converted static cells into live placeholders
│           ├── wire_*.py              ← The patches that wired new repeating tables into the template
│           ├── scan_*.py / verify_*.py / inspect_*.py  ← Diagnostic helpers used while patching
│           └── fetch_live_values.py 💤 ← The generator's predecessor, superseded and kept for history
│
├── reports/                           ← The client deliverable — nothing else belongs here
│   ├── NetSuite_Audit_Report_<DD_MM_YYYY>.docx  🔒 ← The finished audit report for the connected account
│   ├── logs/                     🔒   ← Timestamped transcripts of each report run
│   └── .gitkeep                       ← Keeps the folder in version control while its contents stay out
│
└── project-complete-life-cycle-claude-session/    ← Full history of how the project was built
    ├── README.md                                 ← Session index: what each session did, decided and fixed
    └── session_NN_DD_MM_YYYY_summary.jsonl       ← One redacted transcript per session (27 to date)
```

### The six things worth knowing about this structure

1. **`SDF/AuditAI/` is the only thing that ever enters NetSuite** — and it enters through exactly
   one command (`scripts/deploy.ps1`). Every script needs a `.js` file under `FileCabinet/` **and**
   a matching `.xml` under `Objects/`; a single deploy installs both.
2. **`scripts/` is the only place that talks to the live account** from the operator's machine, and
   every call in it is signed by the one shared signer, `NsAuth.psm1`.
3. **`config/work/` holds the working data, `reports/` holds only the deliverable.** The deep-dive
   output, validation exports and intermediates are all working data and are never shipped.
4. **Nothing sensitive is ever committed** — the credentials file, the certificates, the operator
   CSV exports, the working files and the generated reports are all gitignored (🔒).
5. **`audit-docx-tool/` has two halves:** `scripts/` is the working pipeline (generate → build →
   gates), while `templates/` holds the locked design source plus one dated backup per past patch —
   the reason every template change in this project is reversible.
6. **The 💤 items are deliberate history, not clutter.** The deprecated Word-COM builder, the
   generator's predecessor, the stale `.js.bak_folder` files and the ~45 template backups are all
   kept as the reversibility and provenance trail — but no arrow in the working pipeline touches
   them. The 14 `.js.bak_folder` files are the one group worth cleaning up: they sit inside the
   deployable `FileCabinet/` path even though no `Objects/*.xml` references them.
