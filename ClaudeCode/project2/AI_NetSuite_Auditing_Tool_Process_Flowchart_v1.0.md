# AI NetSuite Auditing Tool — Complete Process Flowchart (v1.0)

> **What this is.** The end-to-end process of this project, from an empty machine to a delivered
> DOCX audit report, drawn as flowcharts. It covers **every** stage: setup, the read-only discipline,
> the SDF deploy, the in-account capture, the two read-only deep-dives that SuiteScript cannot
> replace, the value generation, the scoring model, the DOCX render, the verification gates, and the
> governance (save / archive) loop.
>
> **Accuracy basis.** Every node below was traced from the **source files on disk** (not from the
> README or the CHANGELOG) on **2026-08-03**. Each node maps to a real file and line range — see
> [§12 Traceability](#12-traceability--node--source). Where a component exists on disk but is **not
> part of the working flow** (deprecated, superseded or dormant), it is drawn dashed and listed in
> [§13](#13-on-disk-but-not-in-the-flow).
>
> **How to read.** Diagrams are Mermaid — they render in VS Code (Markdown Preview, Mermaid
> extension), GitHub, Obsidian, and most Markdown viewers. Shape legend:
>
> | Shape | Meaning |
> |---|---|
> | Rectangle | A script / process step that runs |
> | Rounded | A file or data artifact produced or consumed |
> | Diamond | A decision or enforcement gate |
> | Dashed | Exists on disk but **not** wired into the working flow |

---

## Contents

1. [Master flow — the whole project in one view](#1-master-flow--the-whole-project-in-one-view)
2. [Phase 0 — One-time setup & preflight](#2-phase-0--one-time-setup--preflight)
3. [Phase 1 — Read-only discipline (the four layers)](#3-phase-1--read-only-discipline-the-four-layers)
4. [Phase 2 — Deploy (the only way code enters the account)](#4-phase-2--deploy-the-only-way-code-enters-the-account)
5. [Phase 3 — Capture (the audit engine runs inside NetSuite)](#5-phase-3--capture-the-audit-engine-runs-inside-netsuite)
6. [Phase 3b — The two read-only deep-dives + operator inputs](#6-phase-3b--the-two-read-only-deep-dives--operator-inputs)
7. [Phase 4 — Generate the 401 values](#7-phase-4--generate-the-401-values)
8. [Phase 5 — Score (34 checks, 3 pillars)](#8-phase-5--score-34-checks-3-pillars)
9. [Phase 6 — Render (values JSON → DOCX)](#9-phase-6--render-values-json--docx)
10. [Phase 7 — Verify (the gates)](#10-phase-7--verify-the-gates)
11. [Cross-cutting — the live-or-NA decision tree](#11-cross-cutting--the-live-or-na-decision-tree)
12. [Traceability — node → source](#12-traceability--node--source)
13. [On disk but NOT in the flow](#13-on-disk-but-not-in-the-flow)
14. [Phase 8 — Governance: change, save, archive](#14-phase-8--governance-change-save-archive)
15. [The exact command sequence](#15-the-exact-command-sequence)

---

## 1. Master flow — the whole project in one view

```mermaid
flowchart TD
    subgraph P0["Phase 0 - Setup (once per machine / account)"]
        A1["Operator fills config/account.json<br/>accountId + TBA tokens + sdf.passkey + certId + javaHome"]
        A2["scripts/bootstrap.ps1<br/>Java 21, SuiteCloud CLI, SDF auth via account:setup:ci, first deploy"]
        A1 --> A2
    end

    subgraph P1["Phase 1 - Read-only discipline (always on)"]
        B1["Layer 4: the platform - SuiteQL is SELECT-only"]
        B2["Layer 3: least-privilege read token -<br/>admin token only for triggering + read-only COUNTs"]
        B3["Layer 2: deployed scripts read, and write only their own result files"]
        B4["Layer 1: the standing guardrails, auto-loaded by the prompt hook"]
    end

    subgraph P2["Phase 2 - Deploy (SDF is the ONLY method)"]
        C1["scripts/deploy.ps1 - suitecloud project:deploy"]
        C2(["In NetSuite: 24 .js + 28 Objects, 8 of them Map/Reduce"])
        C1 --> C2
    end

    subgraph P3["Phase 3 - Capture (inside the account)"]
        D1["scripts/run-audit.ps1 -Script all<br/>16 writers triggered via the Deployer RESTlet"]
        D2(["17 audit_result_*.json in SuiteScripts/AuditAI<br/>folder resolved at runtime"])
        D1 --> D2
    end

    subgraph P3B["Phase 3b - What no SuiteScript can reach"]
        E1["Two read-only SDF object:import deep-dives<br/>into a THROWAWAY temp project"]
        E2(["config/work/Workflow_Action_Audit_*.json<br/>config/work/CustomForm_Defs_*.json"])
        E3(["Operator inputs: account.accountTier<br/>+ 4 config/manual-inputs CSV exports"])
        E1 --> E2
    end

    subgraph P4["Phase 4-5 - Generate + Score (last live contact)"]
        F1["generate_fresh_values.py - 12 phases<br/>17 result files + live SuiteQL + deep-dives + operator inputs"]
        F2["34 checks to 3 pillars to 1 overall<br/>NA excluded and re-normalised at both levels"]
        F3(["input/values_DD_MM_YYYY.json - 401 keys<br/>THE SNAPSHOT BOUNDARY"])
        F1 --> F2 --> F3
    end

    subgraph P6["Phase 6-7 - Render + Verify (100% offline)"]
        G1["build_audit_docx.py<br/>expand 16 repeating tables, strip editorials, fill 401 tokens"]
        G2{"Gates: static-data lint, 0 unfilled,<br/>0 demo fingerprints, OOXML valid, smoke_test"}
        G3(["reports/NetSuite_Audit_Report_DD_MM_YYYY.docx<br/>THE DELIVERABLE"])
        G1 --> G2
        G2 -->|"pass"| G3
        G2 -->|"fail - exit 2, no file written"| F1
    end

    subgraph P8["Phase 8 - Governance"]
        H1["Operator reviews the DOCX in Word"]
        H2["Save: CHANGELOG + memory + session archive + README row"]
        H3["Next day-start ritual re-derives every claim from disk"]
        H1 --> H2 --> H3
    end

    A2 --> B1 --> B2 --> B3 --> B4 --> C1
    C2 --> D1
    D2 --> F1
    E2 --> F1
    E3 --> F1
    F3 --> G1
    G3 --> H1
    H3 -.->|"next audit cycle"| D1
```

**The single most important property of this flow:** the tool **installs its own read-only engine**
and then reads. Live NetSuite contact spans Phases 2–4 and **ends when `values_<date>.json` is
written**; Phases 6–7 read that file from disk and make **zero** network calls, so a report can be
re-rendered any number of times without touching the account. (Re-*generating* the values does
require the account — the generator queries SuiteQL directly, there is no offline snapshot of the
raw account.)

---

## 2. Phase 0 — One-time setup & preflight

```mermaid
flowchart TD
    S1["Copy config/account.example.json to config/account.json"] --> S2["Fill accountId, accountName, TBA consumerKey/Secret + tokenId/Secret,<br/>optional adminTokenId/Secret, sdf.passkey, sdf.certIdAdmin,<br/>java.javaHome, audit.adminEmailDomain, optional accountTier"]
    S2 --> S3["Run scripts/bootstrap.ps1"]
    S3 --> G1{"Step 1 - Java 21 on a known path<br/>or in JAVA_HOME?"}
    G1 -->|"no"| S4["Prompt for the JDK 21 path -<br/>warn that SDF deploy fails without it"]
    G1 -->|"yes"| G2
    S4 --> G2{"Step 2 - suitecloud on PATH?"}
    G2 -->|"no"| S5["npm install -g @oracle/suitecloud-cli@latest<br/>exit 1 if npm itself is missing"]
    G2 -->|"yes"| G3
    S5 --> G3{"Step 3 - config/account.json exists?"}
    G3 -->|"no"| S6["Copy the example, print the fields to fill,<br/>wait for the operator, then write back the detected javaHome"]
    G3 -->|"yes, and no -Force"| S7["Keep it - credentials are never overwritten"]
    S6 --> G4
    S7 --> G4{"Step 4 - is sdf.authIdDeploy already registered?"}
    G4 -->|"yes"| S9
    G4 -->|"no"| S8["suitecloud account:setup:ci<br/>--authid --account --tokenid certIdAdmin --privatekeypath<br/>the RSA-4096 private key is copied to a space-free temp path first"]
    S8 --> S9["Step 5 - run scripts/deploy.ps1<br/>unless -SkipDeploy"]
    S9 --> S10["Ready: run-audit.ps1 -Script all, then the report build"]
```

**Key guarantees established here.** `SUITECLOUD_CI=1` plus `SUITECLOUD_CI_PASSKEY` make every
later SDF call non-interactive, so a deploy never blocks on a prompt. The passkey and the private
key stay in `config/account.json` / `config/certs/` — the single source of truth — and are read
fresh on every invocation rather than being duplicated into any script. The certificate is mapped
to NetSuite's built-in **SuiteCloud Development Integration** application; a stale
`credentials_ci.p12` from a previous account must be removed before re-running against a new one.

---

## 3. Phase 1 — Read-only discipline (the four layers)

This project's read-only property is **not** enforced by an egress gate. It rests on four layers,
and it is worth being precise about which of them are structural and which are disciplinary.

```mermaid
flowchart LR
    subgraph L4["Layer 4 - STRUCTURAL: the platform"]
        P1["SuiteQL has no INSERT, UPDATE or DELETE statement.<br/>Every query the tool can express is a SELECT."]
    end
    subgraph L3["Layer 3 - STRUCTURAL: least-privilege credentials"]
        P2["tba.tokenId - a read-only audit role<br/>SuiteAnalytics Workbook + record View only"]
        P3["tba.adminTokenId - used ONLY for: RESTlet triggering,<br/>read-only SELECT COUNT fallbacks, and creating<br/>the AuditAI-owned drill-down saved searches"]
    end
    subgraph L2["Layer 2 - STRUCTURAL: what the deployed code does"]
        P4["N/query, N/search, N/record.load, N/config, N/dataset - READ<br/>the only write is file.create of audit_result_*.json<br/>into the framework's OWN folder"]
    end
    subgraph L1["Layer 1 - DISCIPLINARY: the standing guardrails"]
        P5["CLAUDE.md + .agent/03_AUDIT_PLAYBOOK.md + config/precautions/01<br/>auto-loaded into the agent by scripts/precautions_hook.ps1"]
    end
    P1 --> NS["NetSuite account"]
    P2 --> NS
    P3 --> NS
    P4 --> NS
    P5 --> NS
```

### 3.1 What a request actually does

```mermaid
sequenceDiagram
    participant Op as Operator machine
    participant Signer as scripts/NsAuth.psm1
    participant NS as NetSuite
    participant FC as File Cabinet

    Op->>Signer: New-NsAuthHeader POST <suiteql url> cfg
    Signer->>Signer: fold any query-string params into the signature base
    Signer->>Signer: HMAC-SHA256 over method + base url + sorted params
    Signer->>Signer: realm = accountId uppercased with - replaced by _
    Signer-->>Op: OAuth 1.0a Authorization header
    Op->>NS: POST /services/rest/query/v1/suiteql  (a SELECT)
    NS-->>Op: 200 items

    Op->>NS: POST restlet.nl with action=runScript
    NS->>NS: RESTlet runs AS THE CALLING TOKEN'S ROLE
    NS->>NS: flip deployment to NOTSCHEDULED, N/task.submit, restore status
    NS->>FC: the audit script writes its audit_result file
    NS-->>Op: task id

    Op->>NS: POST with action=readFileFull and a fileName
    NS->>FC: search by name, load the NEWEST match
    NS-->>Op: parsed JSON content
```

### 3.2 The Deployer RESTlet's action surface — and what the pipeline actually calls

| Action | Kind | Called by the pipeline? |
|---|---|---|
| `GET ?fileId=` | read | No — manual investigation only |
| `runScript` | trigger | **Yes** — `run-audit.ps1` |
| `readAllResults` | read | **Yes** — `run-audit.ps1 -ReadResult` |
| `readLogs` | read | **Yes** — `run-audit.ps1 -ReadResult` (single file) |
| `readFileFull` | read | **Yes** — `generate_fresh_values.py` (all 17 result files) |
| `runQuery` | read (SELECT-guarded) | **Yes** — the admin-token fallback for full-access-gated tables |
| `ensureDebugProdSearch` | creates an **AuditAI-owned** saved search, idempotent | **Yes** — the §3.3 drill-down hyperlink |
| `ensureDebugAllSearch` | creates an **AuditAI-owned** saved search, idempotent | **Yes** — the §3.3 total drill-down hyperlink |
| `lockFolder` | restricts the AuditAI folder | Only `secure_deploy.ps1` |
| `runSavedSearch`, `savedSearchList`, `loadSearch`, `setExecuteAs` | read / config | **No** |
| `createFile`, `createScript`, `createDeployment`, `deployFull` | **write — FORBIDDEN by Guardrail 2** | **No — nothing calls them** |

> **Stated plainly:** the four deployment actions exist in the RESTlet source and are explicitly
> forbidden by Guardrail 2. Verified by search, **no `.ps1` or `.py` in the project calls any of
> them**; the only invoked actions are the eight marked "Yes". A mutation is therefore prevented by
> platform limits, credential scope and enforced discipline — not by a component that could reject
> one at the wire. Removing those four actions from the RESTlet would make the guarantee
> structural; that is a deliberate open item, not an oversight in this document.

---

## 4. Phase 2 — Deploy (the only way code enters the account)

```mermaid
flowchart TD
    K0["scripts/deploy.ps1 with optional -DryRun or -AuthId"] --> K1{"config/account.json present?"}
    K1 -->|"no"| Z1["exit 1 - copy the example and fill it in"]
    K1 -->|"yes"| K2{"java.javaHome valid,<br/>or a known JDK 21 path exists?"}
    K2 -->|"no"| Z2["exit 1 - run bootstrap.ps1"]
    K2 -->|"yes"| K3["Set JAVA_HOME, prepend it to PATH,<br/>SUITECLOUD_CI=1, SUITECLOUD_CI_PASSKEY from config"]
    K3 --> K4{"SDF/AuditAI/suitecloud.config.js found?"}
    K4 -->|"no"| Z3["exit 1 - SDF project not found"]
    K4 -->|"yes"| K5["Inject audit.adminEmailDomain into the 3 param XMLs<br/>security / sod / dashboard - remembering the originals"]
    K5 --> K6{"-DryRun?"}
    K6 -->|"yes"| Z4["Print the command and exit 0 - nothing deployed"]
    K6 -->|"no"| K7["suitecloud project:deploy --accountspecificvalues WARNING"]
    K7 --> K8["finally: restore ONLY the XMLs that were modified,<br/>so the committed source stays generic"]
    K8 --> K9{"exit code 0?"}
    K9 -->|"no"| Z5["Write-Error and propagate the exit code"]
    K9 -->|"yes"| K10(["Deployed: 24 .js under FileCabinet/SuiteScripts/AuditAI<br/>+ 28 Objects - Script, Deployment and saved-search records"])
```

### 4.1 What SDF installs, and the three platform quirks baked into it

| Item | Detail |
|---|---|
| `src/manifest.xml` | `SERVERSIDESCRIPTING` must sit **inside** `<dependencies>`, not at top level |
| `src/deploy.xml` | Deploys `~/AccountConfiguration/*`, `~/FileCabinet/*`, `~/Objects/*` |
| Script `<name>` field | Hard limit of **40 characters** — every script name in `Objects/` respects it |
| `<starttime>` | Must land on a 30-minute increment (`03:30:00Z`, never `03:45`) |
| Changing `@NScriptType` | NetSuite binds a file to its original type: to convert Scheduled → Map/Reduce you must delete the **deployment, then the Script record, then the `.js` file** in the target account and redeploy. A brand-new account deploys cleanly. |

### 4.2 The secure-delivery variant

`scripts/secure_deploy.ps1` is the same single deploy command with an obfuscation wrapper: it backs
up all 24 sources, extracts each file's JSDoc header (SDF requires the `@NApiVersion` /
`@NScriptType` tags to stay readable), obfuscates only the body, deploys, then **restores the
readable sources locally** — so NetSuite holds minified code while the repo keeps the original.
On a failed deploy it restores the originals and exits 1. Step 3 then calls `lockFolder` to
restrict the AuditAI File Cabinet folder to Administrator.

---

## 5. Phase 3 — Capture (the audit engine runs inside NetSuite)

```mermaid
flowchart TD
    M0["scripts/run-audit.ps1 -Script all"] --> M1{"tba.adminTokenId present?"}
    M1 -->|"yes"| M2["Swap tokenId/tokenSecret for the admin pair<br/>- a RESTlet runs as the CALLER's role and N/task.submit needs admin"]
    M1 -->|"no"| M3["Use tba.tokenId - triggering may be refused -<br/>fall back to waiting for the timer schedules"]
    M2 --> M4
    M3 --> M4["For each of the 16 writers in allWriters:<br/>POST action=runScript with scriptId and deployId, then sleep 2s"]
    M4 --> M5{"Trigger succeeded?"}
    M5 -->|"no"| M6["Write-Warning and CONTINUE to the next writer<br/>- one blocked script never aborts the run"]
    M5 -->|"yes"| M7["The script executes in NetSuite"]
    M7 --> M8{"Is this a Map/Reduce writer?"}
    M8 -->|"yes"| M9["Each item gets its OWN governance budget<br/>- no early governance cut-off on a large account"]
    M8 -->|"no"| M10["Single execute() within one budget"]
    M9 --> M11
    M10 --> M11{"Did a check hit a role or platform block?"}
    M11 -->|"yes"| M12["Record the failure IN the result file, e.g. search_error<br/>so a real zero is distinguishable from missing data"]
    M11 -->|"no"| M13["Record the live counts, summary and topFindings"]
    M12 --> M14
    M13 --> M14["file.create into the folder resolved at RUNTIME<br/>- looks up the deployed deployer script's own folder,<br/>falls back to the SuiteScripts root. No hardcoded ID."]
    M14 --> RES(["17 audit_result_*.json in the File Cabinet"])
    M6 --> M4
```

### 5.1 The 24 deployed scripts and what each contributes

| # | Script | Type | Writes / does |
|---|---|---|---|
| 1 | `ns_audit_deployer_rl.js` | RESTlet | **The entry point** — triggers scripts, returns result files, runs read-only COUNTs, ensures the two drill-down saved searches |
| 2 | `ns_audit_financial_anomaly_ss.js` | Scheduled | `audit_result_financial.json` — transaction anomalies |
| 3 | `ns_audit_security_ss.js` | Scheduled | `audit_result_security.json` — admin/operator accounts, 2FA, batch user creation, unused logins |
| 4 | `ns_audit_script_performance_ss.js` | Scheduled | `audit_result_perf.json` — script execution health |
| 5 | `ns_audit_system_config_ss.js` | **Map/Reduce** | `audit_result_sysconfig.json` — each config check in its own budget |
| 6 | `ns_audit_ui_performance_ss.js` | **Map/Reduce** | `audit_result_ui.json` — client scripts per record type |
| 7 | `ns_audit_sod_ss.js` | Scheduled | `audit_result_sod.json` — segregation-of-duties conflicts |
| 8 | `ns_audit_integration_perf_ss.js` | Scheduled | `audit_result_integration.json` — execution stats, ecosystem deployments, token age/rotation |
| 9 | `ns_audit_bundle_inventory_ss.js` | Scheduled | `audit_result_bundle.json` — bundles by name match (`script.bundleid` is absent on some accounts) |
| 10 | `ns_audit_phd_ss.js` | Scheduled | `audit_result_phd.json` — process-health diagnostics |
| 11 | `ns_audit_company_config_ss.js` | Scheduled | `audit_result_company_config.json` — password policy (preset **and** legacy models) + IP restrictions |
| 12 | `ns_audit_script_log_ss.js` | **Map/Reduce** | `audit_result_script_log.json` — a *filtered* `scriptexecutionlog` search runs for **any** role only inside a Map/Reduce |
| 13 | `ns_audit_saved_search_ss.js` | **Map/Reduce** | `audit_result_saved_search.json` — date-filter efficiency + join complexity for **every** search |
| 14 | `ns_audit_login_monitor_ss.js` | Scheduled | `audit_result_login_monitor.json` — failure rate, top failing users/IPs |
| 15 | `ns_audit_sched_freq_mr.js` | **Map/Reduce** | `audit_result_sched_freq.json` — real cadence from `scriptdeployment.recurringevent`; SuiteQL exposes no schedule field |
| 16 | `ns_audit_workflow_mr.js` | **Map/Reduce** | `audit_result_workflow.json` — per-workflow triggers, logging, run-as-admin, states and actions (powers §3.4) |
| 17 | `ns_audit_analytics_mr.js` | **Map/Reduce** | `audit_result_analytics.json` — `N/dataset` + `N/workbook` `.list()`/`.load()`; **the only path** (no SuiteQL table) (powers §4.4) |
| 18 | `ns_audit_pageperf_mr.js` | **Map/Reduce** | `audit_result_pageperf.json` — APM `EndToEndTime` + `SuiteScriptDetail`; read for the §3.1 availability note, **no score** |
| 19 | `ns_audit_trend_logger_ss.js` | Scheduled | Writes scores into the AuditAI score-history custom record (not report data) |
| 20 | `ns_audit_export_email_ss.js` | Scheduled | Emails the daily summary (not report data) |
| 21 | `ns_audit_log_reader_ss.js` | Scheduled | Execution-log reader helper |
| 22 | `ns_audit_health_dashboard_sl.js` | Suitelet | In-account dashboard page |
| 23 | `ns_audit_realtime_ue.js` | User Event | Real-time alerts on journal / vendor bill / employee saves |
| 24 | `ns_audit_deal_risk_rl.js` | RESTlet | On-request deal-risk scoring |

> **Counting note.** 17 of the 24 scripts write a result file the report reads (rows 2–18).
> `-Script all` triggers **16** of them — everything except `pageperf`.
> `trend` and `email` are excluded on purpose (not report data), and **`pageperf` is explicit-only**:
> the generator reads its result file, so on a fresh account trigger it by name
> (`-Script pageperf`) or wait for its daily run, otherwise the §3.1 note renders "not captured".
>
> **Naming note.** Four files keep an `_ss` suffix although their `@NScriptType` is
> `MapReduceScript` (`system_config`, `ui_performance`, `script_log`, `saved_search`) — a residue of
> the conversions. The `@NScriptType` tag, not the filename, is what NetSuite binds.

---

## 6. Phase 3b — The two read-only deep-dives + operator inputs

Two classes of data are unreachable by **any** deployed script, so they cannot be captured in
Phase 3 at all.

```mermaid
flowchart TD
    N0["audit-docx-tool/build.ps1"] --> N1["Step 0 - scripts/audit-workflow-actions.ps1"]
    N0 --> N2["Step 0b - scripts/audit-customform-defs.ps1"]
    N1 --> N3["Create a THROWAWAY project under the OS temp dir<br/>copy in only suitecloud.config.js, project.json, manifest.xml, deploy.xml"]
    N2 --> N3
    N3 --> N4["suitecloud object:import --scriptid ALL --excludefiles<br/>workflow  /  transactionForm + entryForm + addressForm"]
    N4 --> N5{"Any XML imported?"}
    N5 -->|"no"| N6["Warn: check the role can view SuiteFlow / Forms<br/>- the section degrades to NA"]
    N5 -->|"yes"| N7["workflow_action_parser.py  /  parse_form_xml.py"]
    N7 --> N8(["config/work/Workflow_Action_Audit_DD_MM_YYYY.json + .md<br/>config/work/CustomForm_Defs_DD_MM_YYYY.json"])
    N3 --> N9["finally: Remove-Item -Recurse the temp project<br/>- imported client objects can NEVER enter SDF/AuditAI or a deploy"]
    N6 --> N10["NON-FATAL - the rest of the report is unaffected"]
```

| Deep-dive | Why no script can do it | What it yields |
|---|---|---|
| **Workflow actions** (`audit-workflow-actions.ps1` → `workflow_action_parser.py`) | `N/search` / `N/record.load` return action **types** only; `workflowaction` and `workflowstate` are invalid search and record types. The per-action *values* — static literals, sourced fields, formulas — exist **only in the SDF workflow XML**. | The §3.4 value-source breakdown, the top workflows by hardcoded-value count, and the scored "hardcoded action values" maintainability check |
| **Custom-form definitions** (`audit-customform-defs.ps1` → `parse_form_xml.py`) | There is no `customform` / `transactionform` / `entryform` table, no `N/record` or `N/search` form type, and `N/ui/serverWidget` only *builds* forms. A deployed script would have nothing to read. | The §4.5 field-level definitions — shown/hidden fields, forced-mandatory overrides, sublists, attached scripts — fused with the live SuiteQL `transaction.customform` **usage** layer |

Both scripts use the identical safety pattern: a throwaway temp project (never `SDF/AuditAI`, whose
`deploy.xml` would otherwise sweep imported client objects into a deploy), `object:import` only,
and unconditional cleanup in a `finally` block.

> **One real-world caveat, handled in code.** `parse_form_xml.py` writes its confirmation to
> **stderr**, and Windows PowerShell under `$ErrorActionPreference='Stop'` turns native stderr into
> a terminating error *even on exit 0*. `audit-customform-defs.ps1` therefore drops to `Continue`
> around the Python call and gates on the real exit code, so only a genuine failure propagates.

### 6.1 Operator-provided inputs — the values no client can read

| Input | Where it goes | What it drives |
|---|---|---|
| Two billing/Setup screenshots | `config/account.json` → `accountTier` | SuiteCloud processors, storage allocated/used, **Full Licensed Users** (the "Active user licenses" score), integration concurrency limit |
| Installed Bundles export | `config/manual-inputs/InstalledBundles*.csv` | The bundle inventory |
| Bundle Audit Trail export | `config/manual-inputs/BundleAuditTrail*.csv` | **Bundle version currency** — installed vs latest |
| Manage Integrations export | `config/manual-inputs/Integrations*.csv` | The §7 integration-records inventory (the `integration` table is sealed at every API level) |
| UI Scripts list export | `config/manual-inputs/UIScripts*.txt` | The §7 per-type "Source / Notes" UI-vs-SuiteQL cross-verify |

Each missing input yields `NA` (or the base label) plus a printed reminder from the generator —
never an assumed default. `config/manual-inputs/README.md` is the canonical checklist.

---

## 7. Phase 4 — Generate the 401 values

```mermaid
flowchart TD
    Q0["generate_fresh_values.py --config config/account.json --out input/values_DD_MM_YYYY.json"]
    Q0 --> Q1["Derive the run identity from today:<br/>report date, audit period, NetSuite release (H1 = year.1, H2 = year.2)<br/>and normalise the OAuth realm to the underscore-uppercase Company ID"]
    Q1 --> Q2["PHASE 1 - read 17 result files via readFileFull<br/>newest match by name - empty result is recorded as EMPTY, not as zero"]
    Q2 --> Q3["PHASE 2 - live SuiteQL inventory queries"]
    Q3 --> Q4{"Call failed?"}
    Q4 -->|"HTTPError - deterministic"| Q5["Do NOT retry - degrade this value to None"]
    Q4 -->|"URLError / TimeoutError / OSError"| Q6["Retry up to 3 attempts, backoff 2s then 5s,<br/>RE-SIGNING the header each time so the nonce is never reused"]
    Q5 --> Q7
    Q6 --> Q7["PHASE 3 - compute the 34 sub-scores<br/>each gated by an availability flag that inspects the SOURCE's error field"]
    Q7 --> Q8["PHASES 4-10 - observations, key findings, governance tables,<br/>FINDING-DRIVEN recommendations with a generic fallback, remediation roadmap"]
    Q8 --> Q9["PHASE 11 - assemble the values dict, including the 16 repeating row arrays"]
    Q9 --> Q10["PHASE 12 - write the JSON and print the key count + pillar scores"]
    Q10 --> VALS(["input/values_DD_MM_YYYY.json - 401 keys"])
```

### 7.1 The accuracy rules enforced inside the generator

| Rule | Mechanism |
|---|---|
| A failed query must never become a `0` | `count_or_none()` returns `None` (→ `NA`). A successful SuiteQL `COUNT` always returns one row, so an empty result means *the query did not run* — the opposite of a genuine zero. Plain `count()` is reserved for values where a real zero is the only possibility. |
| A blocked source must never score `100` | The availability flag checks the source's own error field, not merely that the file exists — e.g. `avail_log = bool(summary) and not summary.get('search_error')`. A zero-error result from a *failed* search had previously inflated Performance to 85. |
| A full-access-gated table must degrade, not crash | `count_or_none_admin()` tries the least-privilege read token first, then falls back to a **`SELECT`-guarded** read-only `runQuery` through the RESTlet signed with the admin token, then `None`. |
| A recommendation must never assert an unmeasured finding | Recommendations are built from the live findings with a generic fallback; the render guard additionally blocks three known demo-era recommendation phrases on **every** account. |
| Nothing may be account-specific | Realm normalisation (`accountId.upper().replace('-','_')`) makes a sandbox work unchanged; folder IDs resolve at runtime; saved searches load by `scriptid`, never by numeric ID; the report's hyperlink subdomain is rewritten at build time. |
| A heavy account must not crash the build | `timeout=120` on every request, plus graceful degradation on `(URLError, TimeoutError, OSError)`. |

---

## 8. Phase 5 — Score (34 checks, 3 pillars)

Applied in one place only — the generator — so the report can never disagree with itself.

```mermaid
flowchart TD
    SG["For each of the 34 checks"] --> S1{"Is there a live basis for it?"}
    S1 -->|"no"| NA["Score = NA - the check drops out of its pillar entirely"]
    S1 -->|"yes"| S2["Score = a disclosed banding function of the live number, 0-100"]
    NA --> S3["Collect only the SCORED checks per pillar"]
    S2 --> S3
    S3 --> S4["Pillar = round(mean of its available checks)<br/>a pillar with none available is itself NA"]
    S4 --> S5["Overall = round(sum(score x weight) / sum(weights of AVAILABLE pillars))<br/>Performance 0.33 - Optimization 0.33 - Security 0.34"]
    S5 --> S6{"Which band?"}
    S6 -->|"80-100"| B1["Good - green D4EDDA"]
    S6 -->|"65-79"| B2["Fair - amber FFF3CD"]
    S6 -->|"below 65"| B3["Needs Attention - red FDECEA"]
    S6 -->|"NA"| B4["Neutral grey F2F2F2 - no live data, no fabricated number"]
    B1 --> OUT["Section 1 banner + pillar bars + the 34-row sub-category scorecard<br/>Section 2.4 sub-check counts derived FROM the same list, so they can never go stale"]
    B2 --> OUT
    B3 --> OUT
    B4 --> OUT
```

| Pillar | Weight | Checks | The live signals |
|---|---|---|---|
| **Performance** | 33% | 12 | Max scripts on one record type · total deployed scripts · integration failure rate · script timeout count · total error-log entries · governance-limit proxy · workflows always-logging in production · frequently-scheduled workflows · multi-source workbooks · analytics load errors · max custom forms per type · avg forced-mandatory fields + hidden-heavy form share |
| **Optimization** | 33% | 13 | % of saved searches without a date filter · orphaned scripts · TESTING-status deployments · test-named scripts · SuiteScript 1.0 bundle scripts · stale transactions / open POs · inactive records · workflows not RELEASED · hardcoded action values · orphaned datasets and unresolved workbook refs · wide datasets · dead + stale custom forms · forms in use with no SDF definition |
| **Security** | 34% | 9 | Operator/admin account count · batch-created accounts and their age · SoD HIGH/CRITICAL counts · roles without 2FA · password policy (preset or legacy model) · admin roles without IP restriction · login failure rate · integration token age and role hygiene · run-as-Administrator workflows |

**Worked example — the 23 July 2026 live run on the Meri Meri sandbox.** Performance **74**,
Optimization **64**, Security **71** → overall = round(74×0.33 + 64×0.33 + 71×0.34) = **70 · Fair**.
Because an `NA` check is removed *and* its pillar's denominator shrinks with it — and an `NA`
pillar is removed *and* the overall weights are re-normalised — an unmeasurable dimension can never
silently drag the overall score up or down.

---

## 9. Phase 6 — Render (values JSON → DOCX)

```mermaid
flowchart TD
    R0(["input/values_DD_MM_YYYY.json - 401 keys"]) --> R1["build_audit_docx.py values.json, optional --output and --template"]
    R1 --> R2["compute_pillar_visuals - derive the bars, band colours<br/>and (if absent) the overall score from the four score keys -<br/>an NA pillar renders an EMPTY bar with neutral styling"]
    R2 --> R3["Resolve the output path: reports/NetSuite_Audit_Report_<date>.docx<br/>project root found by walking up for CLAUDE.md / .claude"]
    R3 --> GATE1{"GATE 1 - static-data lint on the TEMPLATE"}
    GATE1 -->|"literal data cell found"| STOP["exit 2 - NO DOCX WRITTEN"]
    GATE1 -->|"clean"| R4["office/unpack.py explodes the template into a temp dir"]
    R4 --> R5["Phase 1 - expand the repeating rows FIRST<br/>(new rows carry their own placeholders, filled on the next pass)"]
    R5 --> R6["Phase 1.5 - strip the editorial instruction paragraphs"]
    R6 --> R7["Phase 2 - normalise values, then fill the simple placeholders,<br/>collecting anything left unfilled"]
    R7 --> GATE2{"GATE 2 and 3 - the anti-fabrication render guard"}
    GATE2 -->|"violation"| STOP
    GATE2 -->|"clean"| R8["Phase 2.5 - rewrite every *.app.netsuite.com hyperlink<br/>to the configured account's subdomain (a no-op on the source account)"]
    R8 --> R9["office/pack.py rezips against the original<br/>- OOXML schema validation happens HERE"]
    R9 --> R10(["reports/NetSuite_Audit_Report_DD_MM_YYYY.docx"])
```

### 9.1 The 16 repeating tables the builder expands

| Purpose | Rows on the 23 Jul 2026 run |
|---|---|
| Extra remediation roadmap rows | 6 |
| **Sub-category scores** (the 34-check scorecard) | 34 |
| §3.3 governance-risk scripts | 20 |
| §3.3 DEBUG-in-production scripts | 60 |
| Deployment sprawl | 6 |
| Per-bundle detail | 4 |
| §3.4 workflow detail / best-practice flags | 44 / 5 |
| §3.4 action value-source breakdown / top hardcoded workflows | 4 / 8 |
| §4.4 custom datasets / custom workbooks / analytics flags | 9 / 11 / 5 |
| §4.5 in-use forms / dead forms / form flags | 21 / 31 / 4 |
| Editorial notes stripped | 13 paragraphs |

Each repeating table has an explicit row cap (10–200 depending on the table) so a pathological
account cannot produce an unbounded document.

### 9.2 Why the template is patched only by script

The placeholder template is **locked**: every change goes through an *idempotent* Python `zipfile`
maintenance script that takes a dated backup first (`audit-docx-tool/scripts/maintenance/patch_*.py`
→ `templates/*.bak_*.docx`). That is why all 21 past template changes in this project are
individually reversible, and why hand-editing it in Word is prohibited.

---

## 10. Phase 7 — Verify (the gates)

```mermaid
flowchart TD
    V0(["A build is requested"]) --> V1{"GATE 1 - check_static_data.run_lint on the template<br/>any table data cell holding a literal instead of a placeholder?"}
    V1 -->|"yes"| F1["BUILD BLOCKED - exit 2, no DOCX<br/>each offending cell is listed with its table index"]
    V1 -->|"no"| V2{"GATE 2 - 0 unfilled placeholders after the fill pass?"}
    V2 -->|"no"| F1
    V2 -->|"yes"| V3{"GATE 3 - 0 demo-account fingerprints,<br/>0 demo-era recommendation phrases,<br/>0 demo-era static assumptions?"}
    V3 -->|"no"| F1
    V3 -->|"yes"| V4{"GATE 4 - OOXML schema-valid at pack?"}
    V4 -->|"no"| F2["PACK FAILED - build aborts"]
    V4 -->|"yes"| V5{"GATE 5 - smoke_test.py<br/>scores are int-or-NA and in 0..100 -<br/>overall equals the weighted mean when all four are numeric -<br/>legend + appendix bands consistent, no phantom 1.5x weighting -<br/>no leftover editorial text - US spelling - all 9 sections present"}
    V5 -->|"fail"| F3["Investigate - either the build changed or the record drifted"]
    V5 -->|"pass"| V6["Operator opens it in Word:<br/>page count, no blank pages"]
    V6 --> V7(["Accepted deliverable"])
```

**What the lint allows as legitimately static** (everything else with a digit is a failure): row
numbers in the first column, the `Score (/100)` header, the fixed 33% / 34% pillar weights, the
"Equal weight across N sub-checks" phrasing, the score-band legend, platform constants (the 1,000
governance-unit ceiling, a storage-capacity figure, the `folder -15` fallback), advisory remediation
timeframes ("30 days"), and the term "2FA" — whose only digit is its own "2".

**Re-derive the counts, never quote them from memory:**

```bash
python -c "import zipfile,re;x=zipfile.ZipFile(P).read('word/document.xml').decode('utf8');print(len(re.findall(r'<w:tbl>.*?</w:tbl>',x,re.S)),len(re.findall(r'\{\{[A-Z0-9_]+\}\}',x)))"
```

> The debug-only escape hatch `AUDIT_ALLOW_UNVERIFIED=1` disables Gates 1–3. It exists for local
> template work and must **never** be set for a deliverable.

---

## 11. Cross-cutting — the live-or-NA decision tree

This is the rule that governs **every single value** in the report. It is the project's core
anti-fabrication contract.

```mermaid
flowchart TD
    Q0["A data point is needed for the report"] --> Q1{"Can the read-only token read it<br/>with a SuiteQL SELECT?"}
    Q1 -->|"yes"| A1["LIVE - via sql() / count()"]
    Q1 -->|"no"| Q2{"Can a DEPLOYED script read it<br/>via N/search, N/record.load, N/config or N/dataset?"}
    Q2 -->|"yes"| A2["LIVE - captured into an audit_result_*.json, read back by readFileFull"]
    Q2 -->|"no"| Q3{"Is the table visible to full-access roles only?<br/>savedreport, subsidiary, savedsearch"}
    Q3 -->|"yes"| A3["DERIVED - read-only SELECT COUNT via the admin-token runQuery fallback"]
    Q3 -->|"no"| Q4{"Is it sealed to every API?<br/>customform definitions, the integration table,<br/>workflow action parameter values"}
    Q4 -->|"yes"| A4["Read-only SDF object:import deep-dive,<br/>or an operator CSV export"]
    Q4 -->|"no"| Q5{"Is it a billing / Setup-page fact?<br/>processors, storage, licensed users, concurrency"}
    Q5 -->|"yes"| A5["Operator-provided config.accountTier"]
    Q5 -->|"no"| A6["Render the literal NA"]

    A3 -.->|"still blocked"| A6
    A4 -.->|"deep-dive skipped or CSV absent"| A6
    A5 -.->|"left null"| A6
    A6 --> Q6{"Is a HINT useful?"}
    Q6 -->|"yes"| A7["NA plus the permission or input that would resolve it,<br/>e.g. grant Lists > Customers View"]
    Q6 -->|"no"| A8["Bare NA, neutral grey score cell"]
```

| Class | Examples | Where it comes from |
|---|---|---|
| **Live (SuiteQL)** | Scripts, deployments, transactions, employees, roles, login history, custom records/lists, file storage, PDF templates | `sql()` / `count()` / `count_or_none()` |
| **Live (in-account script)** | Execution-log errors, saved-search efficiency and joins, workflows, datasets and workbooks, password policy, IP restrictions, APM timing | The 17 `audit_result_*.json` files |
| **Derived** | The 34 check scores, the 3 pillars, the overall, observations, key findings, the remediation roadmap | Computed in `generate_fresh_values.py` |
| **Deep-dive (SDF XML)** | Workflow per-action parameter values; custom-form field-level definitions | The two read-only `object:import` deep-dives |
| **Operator-provided** | Account tier, bundle versions, integration records, the UI Scripts cross-check | `account.accountTier` + `config/manual-inputs/` |
| **`NA`** | Anything the role, the tier or the platform withholds | Rendered as the literal `NA`, with a hint where one helps |

**The role boundary is the single biggest scope lever.** Granting the audit role
`Lists → Customers / Vendors / Items → View` (and `Contacts`) converts the duplicate-detection,
incomplete-master-data and entity-inventory rows from `NA` to live, with **no code change**. Two
permissions are non-obvious prerequisites: **`SuiteAnalytics Workbook`** (under Reports) gates the
SuiteQL REST endpoint entirely, and **`Allow JS / HTML Uploads`** is required for the SDF file
deploy.

---

## 12. Traceability — node → source

Every diagram node above resolves to real code. Verified 2026-08-03.

| Flow node | File | Lines |
|---|---|---|
| Java 21 detection | [scripts/bootstrap.ps1](scripts/bootstrap.ps1) | 26–56 |
| SuiteCloud CLI install | [scripts/bootstrap.ps1](scripts/bootstrap.ps1) | 58–74 |
| account.json creation + javaHome write-back | [scripts/bootstrap.ps1](scripts/bootstrap.ps1) | 76–102 |
| SDF auth registration (`account:setup:ci`) | [scripts/bootstrap.ps1](scripts/bootstrap.ps1) | 106–143 |
| **The single OAuth 1.0a signer** | [scripts/NsAuth.psm1](scripts/NsAuth.psm1) `New-NsAuthHeader` | 17–65 |
| Query-string params folded into the signature base | [scripts/NsAuth.psm1](scripts/NsAuth.psm1) | 37–47 |
| Sandbox realm normalisation | [scripts/NsAuth.psm1](scripts/NsAuth.psm1) | 59–63 |
| Deploy env (JAVA_HOME, CI passkey) | [scripts/deploy.ps1](scripts/deploy.ps1) | 44–47 |
| Admin-domain injection + restore | [scripts/deploy.ps1](scripts/deploy.ps1) | 72–112 |
| **The only deploy command** | [scripts/deploy.ps1](scripts/deploy.ps1) | 66, 103 |
| Obfuscate → deploy → restore → lock | [scripts/secure_deploy.ps1](scripts/secure_deploy.ps1) | 18–133 |
| Admin-token swap for RESTlet triggering | [scripts/run-audit.ps1](scripts/run-audit.ps1) | 42–46 |
| Script → scriptId / deployId / result-file map | [scripts/run-audit.ps1](scripts/run-audit.ps1) | 62–82 |
| **The 16 writers `-Script all` triggers** | [scripts/run-audit.ps1](scripts/run-audit.ps1) | 87–89 |
| Trigger loop, warn-and-continue | [scripts/run-audit.ps1](scripts/run-audit.ps1) | 91–118 |
| SuiteQL runner + the WinPS 5.1 `-UseBasicParsing` requirement | [scripts/query.ps1](scripts/query.ps1) | 39–50 |
| Deployer RESTlet — `runScript` | [SDF/.../ns_audit_deployer_rl.js](SDF/AuditAI/src/FileCabinet/SuiteScripts/AuditAI/ns_audit_deployer_rl.js) | 128–198 |
| Deployer RESTlet — `readLogs` / `readAllResults`, newest-wins | same file | 199–268 |
| Deployer RESTlet — `readFileFull` (the generator's read path) | same file | 269–298 |
| Deployer RESTlet — `ensureDebugProdSearch` / `ensureDebugAllSearch` | same file | 337–399 |
| Deployer RESTlet — `runQuery` | same file | 400–411 |
| Deployer RESTlet — **the four FORBIDDEN write actions** | same file | 31–123 |
| Workflow deep-dive: throwaway project + import + cleanup | [scripts/audit-workflow-actions.ps1](scripts/audit-workflow-actions.ps1) | 40–82 |
| Form deep-dive: import loop + stderr handling + cleanup | [scripts/audit-customform-defs.ps1](scripts/audit-customform-defs.ps1) | 52–93 |
| build.ps1 Step 0 / Step 0b / Step 1 / Step 2 | [audit-docx-tool/build.ps1](audit-docx-tool/build.ps1) | 78–100 / 102–118 / 120–133 / 135–147 |
| Generator: config load, realm, output path | [generate_fresh_values.py](audit-docx-tool/scripts/generate_fresh_values.py) | 36–80 |
| Generator: OAuth signing (read + admin tokens) | same file | 85–116 |
| Generator: transient-only retry policy | same file | 117–144 |
| Generator: `sql()` / `count()` | same file | 149–158 |
| **Generator: `count_or_none()` — the no-fabricated-zero rule** | same file | 160–166 |
| Generator: admin-token read-only `runQuery` fallback | same file | 168–202 |
| Generator: `read_result()` | same file | 224–226 |
| Generator: score-band helpers + the strict data guardrail | same file | 235–360 |
| Generator: PHASE 1 — the 13 core result files | same file | 435–497 |
| Generator: the 4 further result reads (workflow, analytics, pageperf, sched_freq) | same file | 962, 1073, 1076, 2809 |
| Generator: availability flags (a blocked source scores `NA`, not 100) | same file | 841–856 |
| **Generator: pillar + overall assembly** | same file | 1238–1243 |
| **Generator: the 34 `SUBCAT_ROWS` checks** | same file | 1253–1288 |
| Generator: §2.4 sub-check counts derived from that list | same file | 1290–1301 |
| Generator: PHASE 12 write + summary print | same file | 3400–3412 |
| Builder: pillar bars / colours / overall | [build_audit_docx.py](audit-docx-tool/scripts/build_audit_docx.py) `compute_pillar_visuals` | 449–504 |
| **Builder: GATE 1 — static-data lint** | same file | 547–563 |
| Builder: the 16 repeating-table definitions | same file | 603–650 |
| Builder: strip editorial notes | same file | 653–656 |
| Builder: fill + collect unfilled | same file | 658–668 |
| **Builder: GATES 2 & 3 — the anti-fabrication render guard** | same file | 670–716 |
| Builder: account-agnostic hyperlink subdomain rewrite | same file | 721–737 |
| Builder: pack + OOXML validation | same file | 739–750 |
| Lint: allowlist of legitimately static cells | [check_static_data.py](audit-docx-tool/scripts/check_static_data.py) | 28–39 |
| Lint: the cell scan | same file | 41–56 |
| Smoke test: the six assertions | [smoke_test.py](audit-docx-tool/scripts/smoke_test.py) | 59–115 |
| Governance: prompt-trigger matching | [scripts/precautions_hook.ps1](scripts/precautions_hook.ps1) | 59–76 |
| Governance: auto-build Triggers A and B | [auto_build_hook.ps1](audit-docx-tool/scripts/auto_build_hook.ps1) | 33–50 |
| Both hooks registered | [.claude/settings.json](.claude/settings.json) | 1–25 |

---

## 13. On disk but NOT in the flow

Drawn here so the chart is **complete as well as correct** — these files exist but no arrow in the
working pipeline touches them.

```mermaid
flowchart LR
    subgraph DEP["Deprecated - warns loudly, kept for history"]
        A["scripts/generate-report.ps1 - a Word-COM report builder<br/>with a HARDCODED overall score"]
    end
    subgraph SUP["Superseded"]
        B["maintenance/fetch_live_values.py - the generator's predecessor"]
        C["scripts/cross_reference.py - standalone form usage-vs-definition fuser -<br/>the generator now does this fusion inline for 4.5"]
    end
    subgraph DORM["Dormant artifacts"]
        D["14 .js.bak_folder files INSIDE the deployable FileCabinet path,<br/>referenced by no Objects/*.xml"]
        E["50 dated template backups + 13 SDF/_backups + temp_docx_extract"]
        F["Unused halves of the bundled OOXML toolkit:<br/>helpers/, soffice.py, the pptx and redlining validators"]
        G["8 never-called Deployer RESTlet actions,<br/>4 of them FORBIDDEN by Guardrail 2"]
    end
```

- **The deprecated builder is safe:** it prints a `Write-Warning` on every invocation stating that
  its score is hardcoded and that the `audit-docx-tool` pipeline is canonical.
- **The dormant artifacts are the reversibility trail** — one dated backup per past template patch
  is exactly why every change in this project can be undone. They are not clutter, but the 14
  `.js.bak_folder` files are the one group worth removing, since they sit in a path `deploy.xml`
  sweeps (`~/FileCabinet/*`).
- **The gates actually in force** are the five in §10. `check_static_data.py` and `smoke_test.py`
  are both live and wired — `check_static_data.run_lint` is imported directly by the builder as a
  hard gate.

### 13.1 Two portability defects found while tracing

Recorded here because this document is meant to be accurate, not flattering. Neither is fixed by
this document — it changed no code.

| Defect | Where | Why it matters |
|---|---|---|
| A hardcoded File Cabinet folder ID | [scripts/secure_deploy.ps1:117](scripts/secure_deploy.ps1#L117) — `folderId=763` | Every other path in the project resolves the AuditAI folder at runtime. `763` is one account's internal ID, so the Step-3 folder lock is a no-op or wrong on any other account. The script does print the manual UI fallback when the call fails. |
| Un-archived work | `reports/` + `audit-docx-tool/input/` | The newest report and values file are dated **23 July 2026** (Overall 70), while the newest `CHANGELOG.md` entry and session archive stop at **10 July 2026** (Overall 69). Nothing in the tooling detects this — only the day-start ritual does. |

---

## 14. Phase 8 — Governance: change, save, archive

```mermaid
flowchart TD
    D0["Every operator prompt passes through<br/>scripts/precautions_hook.ps1 (UserPromptSubmit)"]
    D0 --> D1{"Which trigger phrase?"}
    D1 -->|"good morning / hey claude code"| D2["Load config/WorkStartWithThis/EveryDayStartPromp.md"]
    D1 -->|"(save OR make/making) + chang*"| C1["Load config/precautions/01_making_changes.md"]
    D1 -->|"update full/whole project, save + project, save everything"| S1["Load config/precautions/02_full_project_update.md"]
    D1 -->|"none"| D9["Emit nothing - normal turn"]

    D2 --> D3["Steps 1-2: read CLAUDE.md, .agent/00-05, CONTRIBUTING,<br/>CHANGELOG, the session index and recent archives"]
    D3 --> D4["Steps 3-4: list every file, RE-DERIVE every claim from DISK,<br/>confirm 24 scripts / 28 objects / 8 Map/Reduce, reconcile record vs disk"]
    D4 --> D5["Step 5: report state, then STOP and wait"]

    C1 --> C2["A - trace the real source first - be 100% sure - diagnose, never guess"]
    C2 --> C3["B - hard guardrails: read-only, SDF-only, live-or-NA, secrets stay put"]
    C3 --> C4["C + C2 - additive, backward-compatible,<br/>and it must work on THIS account AND every future account"]
    C4 --> C5["D - template patched only by an idempotent script with a backup -<br/>intermediates to config/work/, deliverable only in reports/"]
    C5 --> C6["E - run the FULL pipeline: deep-dives, generate, gated build,<br/>smoke_test, Word render, verify no blank pages"]
    C6 --> C7["F - keep it reversible - retain the *.bak_* backups"]

    S1 --> S2["A - go folder by folder - no git, persist into the source files"]
    S2 --> S3["B - update CURRENT-STATE only -<br/>PRESERVE every historical and dated entry"]
    S3 --> S4["C - keep counts consistent everywhere:<br/>scripts, objects, Map/Reduce, result files, the 401 key count"]
    S4 --> S5["E - the session archive must contain NO real user, credential,<br/>name or email trace - replace with the labelled placeholders"]
    S5 --> S6["The save is NOT complete until ALL FOUR exist"]
    S6 --> S7["1. dated CHANGELOG.md entry"]
    S6 --> S8["2. memory: project_status.md + the MEMORY.md index line"]
    S6 --> S9["3. session_NN_DD_MM_YYYY_summary.jsonl, redaction-verified"]
    S6 --> S10["4. Files-table row + milestone in the session README"]
    S7 --> F0["F - verify: re-run smoke_test to prove the deliverable is unchanged<br/>if the pass was docs-only"]
    S8 --> F0
    S9 --> F0
    S10 --> F0
    F0 --> D0
```

> **Known limitation of the automation.** The second hook,
> `audit-docx-tool/scripts/auto_build_hook.ps1`, fires on every `Write` and rebuilds the report when
> a `values_*.json` lands in `input/` (Trigger A) or an `Audit_Analysis_*.md` lands in `reports/`
> (Trigger B). Nothing, however, detects that a **completed run was never archived** — which is
> exactly the 23 July 2026 gap in §13.1. The day-start ritual, not any hook, is the source of truth.

---

## 15. The exact command sequence

A complete audit, from a configured machine to a delivered report. Steps 1–4 are the only ones that
contact NetSuite.

```powershell
# --- ONE TIME ---------------------------------------------------------------
Copy-Item config\account.example.json config\account.json   # then fill it in
.\scripts\bootstrap.ps1                                      # Java 21 + SDF CLI + auth + first deploy

# --- 1. Deploy / update the audit engine (THE ONLY deployment method) -------
.\scripts\deploy.ps1                                         # suitecloud project:deploy
# .\scripts\deploy.ps1 -DryRun                               # validate without deploying
# .\scripts\secure_deploy.ps1                                # obfuscated build for a client delivery

# --- 2. Trigger the audit inside the account -------------------------------
.\scripts\run-audit.ps1 -Script all                          # the 16 report result-writers
.\scripts\run-audit.ps1 -Script pageperf                     # explicit-only: powers the 3.1 APM note
# Map/Reduce writers take a few minutes; the report builds regardless
# (a missing or empty result renders NA, never a fabricated zero).

# --- 3. Optional: check anything ad hoc ------------------------------------
.\scripts\query.ps1 -SQL "SELECT COUNT(*) AS cnt FROM script WHERE isinactive = 'F'"

# --- 4. Deep-dives + generate + build, all in one -------------------------
.\scripts\run-report.ps1                                     # logged wrapper -> reports\logs\
#   which runs audit-docx-tool\build.ps1:
#     Step 0  scripts\audit-workflow-actions.ps1   -> config\work\Workflow_Action_Audit_<date>.json
#     Step 0b scripts\audit-customform-defs.ps1    -> config\work\CustomForm_Defs_<date>.json
#     Step 1  generate_fresh_values.py             -> input\values_<date>.json   (401 keys)
#     Step 2  build_audit_docx.py                  -> reports\NetSuite_Audit_Report_<date>.docx
# Reuse fresh deep-dive files and skip Step 0/0b:
#   cd audit-docx-tool ; .\build.ps1 -SkipWorkflowActions

# --- 5. Verify (offline) ---------------------------------------------------
python audit-docx-tool\scripts\check_static_data.py           # template lint, standalone
python audit-docx-tool\scripts\smoke_test.py                  # builds to a temp file and asserts
# Then open the DOCX in Word: page count, no blank pages.

# --- 6. reports\ keeps ONLY the deliverable + logs\ ------------------------
```

Re-rendering the report from values that already exist is **fully offline** — no NetSuite call:

```powershell
python audit-docx-tool\scripts\build_audit_docx.py audit-docx-tool\input\values_23_07_2026.json
```

---

*Document version 1.0 — authored 2026-08-03. Every node traced from the source files on disk; no
value in this document was taken on trust from another document. Read-only: producing this document
made no NetSuite call and changed no code.*
