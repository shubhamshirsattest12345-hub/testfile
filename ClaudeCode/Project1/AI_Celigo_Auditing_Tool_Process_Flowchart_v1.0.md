# AI Celigo Auditing Tool — Complete Process Flowchart (v1.0)

> **What this is.** The end-to-end process of this project, from an empty machine to a
> delivered DOCX audit report, drawn as flowcharts. It covers **every** stage: setup,
> the three-layer read-only enforcement, the live extraction, the offline derivation,
> the scoring model, the DOCX render, the verification gates, and the governance
> (save / archive) loop.
>
> **Accuracy basis.** Every node below was traced from the **source files on disk**
> (not from the README or the CHANGELOG) on **2026-08-03**. Each node maps to a real
> file and line range — see [§11 Traceability](#11-traceability--node--source). Where a
> component exists on disk but is **not part of the working flow** (stub or dormant
> NetSuite-era code), it is drawn dashed and listed in [§12](#12-on-disk-but-not-in-the-flow).
>
> **How to read.** Diagrams are Mermaid — they render in VS Code (Markdown Preview,
> Mermaid extension), GitHub, Obsidian, and most Markdown viewers. Colour/shape legend:
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
3. [Phase 1 — Read-only enforcement (the three layers)](#3-phase-1--read-only-enforcement-the-three-layers)
4. [Phase 2 — Extract (the only live step)](#4-phase-2--extract-the-only-live-step)
5. [Phase 3 — Derive (offline analysis modules)](#5-phase-3--derive-offline-analysis-modules)
6. [Phase 4 — Score (scoring model v1)](#6-phase-4--score-scoring-model-v1)
7. [Phase 5 — Render (Markdown → DOCX)](#7-phase-5--render-markdown--docx)
8. [Phase 6 — Verify (the gates)](#8-phase-6--verify-the-gates)
9. [Cross-cutting — the live-or-NA decision tree](#9-cross-cutting--the-live-or-na-decision-tree)
10. [Phase 7 — Governance: change, save, archive](#10-phase-7--governance-change-save-archive)
11. [Traceability — node → source](#11-traceability--node--source)
12. [On disk but NOT in the flow](#12-on-disk-but-not-in-the-flow)
13. [The exact command sequence](#13-the-exact-command-sequence)

---

## 1. Master flow — the whole project in one view

```mermaid
flowchart TD
    subgraph P0["Phase 0 — Setup (once per machine)"]
        A1["Operator fills config/account.json<br/>apiToken + cli.mode=read + proxy block"]
        A2["scripts/bootstrap.ps1<br/>Node 22+ check, install/verify Celigo CLI,<br/>force mode=read, validate token, read-only connectivity test"]
        A1 --> A2
    end

    subgraph P1["Phase 1 — Read-only enforcement (always on)"]
        B1["Layer 3: config invariant<br/>cli.mode pinned to read"]
        B2["Layer 2: CeligoAuth.psm1<br/>Assert-CeligoReady + read-verb default-deny<br/>+ Invoke-CeligoApi GET/HEAD only"]
        B3["Layer 1: celigo-readonly-proxy.mjs<br/>holds the token, loopback only,<br/>GET/HEAD forwarded, everything else 405"]
    end

    subgraph P2["Phase 2 — Extract (the ONLY live contact)"]
        C1["scripts/extract-celigo.ps1<br/>7 core resources via the proxy"]
        C2["Supplementary CLI reads<br/>users, licenses, notifications, account stats/lint, jobs"]
        C3["4 fetch-*.mjs modules via the proxy<br/>scripts review, connection dependencies,<br/>flow errors, /v1/usage"]
        C4(["config/work/snapshot_DD_MM_YYYY/<br/>raw JSON, gitignored"])
        C1 --> C4
        C2 --> C4
        C3 --> C4
    end

    subgraph P3["Phase 3-4 — Derive + Score (100% OFFLINE)"]
        D1["gen-live-report-full.mjs<br/>resolves the latest snapshot"]
        D2["8 derive-*.mjs modules<br/>run-history, error-MTTR, connection-usage,<br/>export-classification, user-security,<br/>resource-classification, subscription-usage, stale-flows"]
        D3["Scoring model v1<br/>9 risk-weighted domains, NA excluded and re-normalised"]
        D4(["reports/Celigo_Audit_Report_LIVE_DD_MM_YYYY.md"])
        D1 --> D2 --> D3 --> D4
    end

    subgraph P5["Phase 5-6 — Render + Verify"]
        E1["build-live-docx-full.py<br/>neutral live cover, no fabricated client"]
        E2["build_celigo_template.py<br/>shared markdown to OOXML converter"]
        E3["office/pack.py<br/>OOXML schema validation at pack"]
        E4{"Gates: 0 unfilled placeholders,<br/>0 fabrication fingerprints,<br/>0 stale NetSuite header,<br/>table count re-derived from disk"}
        E5(["reports/Celigo_Audit_Report_LIVE_DD_MM_YYYY.docx<br/>THE DELIVERABLE"])
        E1 --> E2 --> E3 --> E4
        E4 -->|pass| E5
        E4 -->|fail| D1
    end

    subgraph P7["Phase 7 — Governance"]
        F1["Operator review"]
        F2["Save: CHANGELOG + memory + session archive + Files row"]
        F3["Next day-start ritual re-reconciles record vs disk"]
        F1 --> F2 --> F3
    end

    A2 --> B1 --> B2 --> B3 --> C1
    C4 --> D1
    D4 --> E1
    E5 --> F1
    F3 -.->|"next audit cycle"| C1
```

**The single most important property of this flow:** live Celigo contact happens **only
in Phase 2**. Phases 3–6 read the snapshot from disk and make **zero** network calls, so
a report can be regenerated any number of times without touching the client account.

---

## 2. Phase 0 — One-time setup & preflight

```mermaid
flowchart TD
    S1["Copy config/account.example.json to config/account.json"] --> S2["Fill apiToken, baseUrl/region,<br/>cli.mode=read, proxy host/port/allowedMethods"]
    S2 --> S3["Run scripts/bootstrap.ps1"]
    S3 --> S4["Import CeligoAuth.psm1 and call Assert-CeligoReady"]
    S4 --> G1{"node on PATH<br/>and major version >= 22?"}
    G1 -->|no| X1["THROW: Node 22+ required"]
    G1 -->|yes| G2{"celigo CLI installed?"}
    G2 -->|no| S5["npm install -g @celigo/celigo-cli<br/>unless CELIGO_NO_AUTOINSTALL=1"]
    G2 -->|yes| S6["Assert-CeligoReadOnly"]
    S5 --> S6
    S6 --> G3{"celigo config get mode == read?"}
    G3 -->|no| S7["celigo config set mode read"]
    G3 -->|yes| S8["Get-CeligoConfig"]
    S7 --> G4{"re-check: mode == read?"}
    G4 -->|no| X2["THROW: refusing to continue"]
    G4 -->|yes| S8
    S8 --> G5{"apiToken present and not the placeholder?<br/>cli.mode == read?"}
    G5 -->|no| X3["THROW: fix config/account.json"]
    G5 -->|yes| S9["Read-only connectivity test<br/>a single list call"]
    S9 --> S10["Ready"]
```

**Key guarantees established here:** the CLI can no longer mutate (`mode=read` rejects
mutations client-side), and the token is validated without ever being written to
`~/.celigo/config.json` — `config/account.json` stays the single source of truth and is
supplied to the CLI through an environment variable at call time only.

---

## 3. Phase 1 — Read-only enforcement (the three layers)

The account's API token is **full-permission** (no read-only token exists on this
account), so read-only is guaranteed by **architecture**, not by trust in the token.

```mermaid
flowchart LR
    subgraph L3["Layer 3 — Config invariant"]
        C["config/account.json<br/>cli.mode pinned to read<br/>proxy.allowedMethods = GET, HEAD"]
    end
    subgraph L2["Layer 2 — Guarded module (CeligoAuth.psm1)"]
        M1["Invoke-CeligoCli<br/>read-verb whitelist, DEFAULT-DENY"]
        M2["Invoke-CeligoApi<br/>ValidateSet GET/HEAD at the parameter"]
    end
    subgraph L1["Layer 1 — Egress proxy (strongest)"]
        P["celigo-readonly-proxy.mjs<br/>the ONLY holder of the token at runtime"]
    end
    C --> M1
    C --> M2
    C --> P
    M1 --> API["Celigo integrator.io REST API"]
    M2 --> API
    P --> API
```

### 3.1 What a request actually does

```mermaid
sequenceDiagram
    participant Client as Audit client - holds NO token
    participant Proxy as celigo-readonly-proxy.mjs
    participant Log as config/work/celigo-proxy-access.log
    participant Celigo as api.integrator.io

    Client->>Proxy: GET /_proxy/health
    Proxy-->>Client: 200 {status ok, readOnly true} (never reaches Celigo)

    Client->>Proxy: GET /v1/flows (no Authorization header)
    Proxy->>Proxy: method in allowedMethods? YES
    Proxy->>Proxy: strip any client Authorization, inject Bearer token
    Proxy->>Celigo: GET /v1/flows + Authorization
    Celigo-->>Proxy: 200 JSON
    Proxy->>Log: ALLOW GET /v1/flows -> 200 (token redacted)
    Proxy-->>Client: 200 JSON

    Client->>Proxy: DELETE /v1/flows/123
    Proxy->>Proxy: method in allowedMethods? NO
    Proxy->>Log: BLOCKED DELETE -> 405
    Proxy-->>Client: 405 read_only_proxy
    Note over Proxy,Celigo: The mutation is NEVER sent to Celigo
```

### 3.2 The proxy's own refusal conditions (fail-closed at startup)

```mermaid
flowchart TD
    R1["node scripts/celigo-readonly-proxy.mjs"] --> R2{"config file exists?"}
    R2 -->|no| Z1["exit 1"]
    R2 -->|yes| R3{"apiToken real, not the placeholder?"}
    R3 -->|no| Z2["exit 1: refusing to start"]
    R3 -->|yes| R4{"cli.mode == read?"}
    R4 -->|no| Z3["exit 1: project is READ-ONLY"]
    R4 -->|yes| R5{"bind host is loopback?<br/>127.0.0.1 / localhost / ::1"}
    R5 -->|no| Z4["exit 1: refusing non-loopback bind"]
    R5 -->|yes| R6["LISTEN — forwards only GET/HEAD"]
```

**Verb enforcement in Layer 2.** `Invoke-CeligoCli` scans the arguments for the first
non-flag token and treats it as the verb: on the read whitelist → allowed; on the
explicit mutating list → blocked; **unrecognised → blocked** (default-deny).

| Allowed (read) | Blocked (explicit mutating) |
|---|---|
| `list`, `get`, `show`, `whoami`, `audit`, `errors`, `resolved-errors`, `error`, `dependencies`, `used-by`, `download`, `info`, `tokeninfo`, `account` | `create`, `update`, `set`, `delete`, `run`, `clone`, `resolve-errors`, `add`, `remove`, `rename`, `enable`, `disable`, `install`, `deploy`, `import`, `revoke`, `rotate`, `retry`, `cancel`, `replay`, `pause`, `resume`, `activate`, `deactivate` |

---

## 4. Phase 2 — Extract (the only live step)

```mermaid
flowchart TD
    E0["scripts/extract-celigo.ps1"] --> E1{"Is the proxy already healthy?"}
    E1 -->|no| E2["Start-CeligoProxy<br/>and remember to stop it afterwards"]
    E1 -->|yes| E3["Reuse the running proxy"]
    E2 --> E4
    E3 --> E4
    E4["Create config/work/snapshot_DD_MM_YYYY/<br/>stamp = today, dd_MM_yyyy"]
    E4 --> E5["For each resource:<br/>integrations, flows, connections, exports,<br/>imports, scripts, stacks"]
    E5 --> E6["Invoke-CeligoProxyGetRaw v1/RESOURCE<br/>RAW JSON string, no PS 5.1 parsing"]
    E6 --> E7{"Response body empty?<br/>e.g. /v1/stacks returns 204"}
    E7 -->|yes| E8["Write literal empty array<br/>an empty collection, never fabricated"]
    E7 -->|no| E9["Write raw bytes verbatim, UTF-8 no BOM"]
    E6 --> E10{"Call threw?<br/>token scope, network, 4xx"}
    E10 -->|yes| E11["Warn RESOURCE -> NA<br/>continue with the next resource, never crash"]
    E8 --> E12(["RESOURCE.json in the snapshot"])
    E9 --> E12
    E12 --> E13["Count records by shelling out to node<br/>PS 5.1 ConvertFrom-Json truncates large payloads"]
    E13 --> E14["Stop the proxy if this script started it"]
```

### 4.1 Everything that lands in a snapshot

```mermaid
flowchart LR
    subgraph CORE["Core 7 — extract-celigo.ps1 via proxy"]
        F1(["integrations.json"])
        F2(["flows.json"])
        F3(["connections.json"])
        F4(["exports.json"])
        F5(["imports.json"])
        F6(["scripts.json"])
        F7(["stacks.json"])
    end
    subgraph SUPP["Supplementary — guarded CLI reads, mode=read"]
        G1(["users.json — users list, the ashares record"])
        G2(["licenses.json — subscriptions licenses"])
        G3(["notifications.json — notifications list"])
        G4(["accountstats.json — account stats"])
        G5(["lint.json — account lint, Celigo's own anomaly detector"])
        G6(["jobs.json — jobs list --created-gte, retention-bounded"])
    end
    subgraph FETCH["Fetch modules — proxy GET, must be running"]
        H1["fetch-review-scripts.mjs"] --> I1(["scripts_review.json<br/>name, lines, rating, note — NEVER raw code"])
        H2["fetch-connection-usage.mjs"] --> I2(["connection_usage.json<br/>/v1/connections/ID/dependencies for every connection"])
        H3["fetch-flow-errors.mjs"] --> I3(["errors_open.json + errors_resolved.json<br/>timestamps/code/classification only — never the message"])
        H4["fetch-subscription-usage.mjs"] --> I4(["subscription_usage.json<br/>GET /v1/usage"])
    end
```

**Two privacy rules enforced inside the fetch modules:** `fetch-review-scripts.mjs`
persists only line count, heuristic rating and a note — **never the script source**;
`fetch-flow-errors.mjs` persists only `occurredAt` / `resolvedAt` / `code` / `source` /
`classification` — **never the error `message`, `exportDataURI`, `traceKey` or retry
keys**, because those carry the client's business data.

---

## 5. Phase 3 — Derive (offline analysis modules)

Every module below is **read-only, offline, account-agnostic and live-or-`NA`** — none
of them makes a Celigo call; each degrades to `NA` on missing/empty input rather than
fabricating a `0`, and none hardcodes an account id, date or path.

```mermaid
flowchart LR
    SNAP(["snapshot_DD_MM_YYYY/"]) --> M1["derive-run-history.mjs"]
    SNAP --> M2["derive-error-mttr.mjs"]
    SNAP --> M3["derive-connection-usage.mjs"]
    SNAP --> M4["derive-export-classification.mjs"]
    SNAP --> M5["derive-user-security.mjs"]
    SNAP --> M6["derive-resource-classification.mjs"]
    SNAP --> M7["derive-subscription-usage.mjs"]
    SNAP --> M8["derive-stale-flows.mjs"]

    M1 --> GEN["gen-live-report-full.mjs"]
    M2 --> GEN
    M3 --> GEN
    M4 --> GEN
    M5 --> GEN
    M6 --> GEN
    M7 --> GEN
    M8 --> GEN
```

| # | Module | Reads | Produces | Accuracy rule it enforces |
|---|---|---|---|---|
| 1 | `derive-run-history.mjs` | `jobs.json` | Observed job window (min→max `createdAt`), success rate **with its denominator**, `purgeAt` record horizon, caveats | The window is read **from the data** — never labelled a hard "30-day"; zero jobs → `NA`, not 0% |
| 2 | `derive-error-mttr.mjs` | `errors_open.json`, `errors_resolved.json` | Backlog, resolved counts (auto vs manual), MTTR mean/median, error taxonomy by code | MTTR comes from `occurredAt`→`resolvedAt`, **not** from job `numResolved` (which is 0); no resolved errors → `NA` |
| 3 | `derive-connection-usage.mjs` | `connection_usage.json`, `flows.json`, `lint.json` | orphaned / referenced-but-in-no-flow / used-by-flow; offline-but-used-by-enabled-flow | Uses Celigo's authoritative `/dependencies`, not partial JSON traversal; cross-checked against `account lint` |
| 4 | `derive-export-classification.mjs` | `exports.json` | Three **independent** facets: adaptor type, sync mode, role (lookup vs standalone) | A blank sync mode is "unspecified", never "(none)"/unknown |
| 5 | `derive-user-security.mjs` | `users.json`, `licenses.json` | Three independent SSO facts (plan capability / account-required / per-user linked), account MFA-required, access governance | "SSO licensed" is never reported as "SSO enforced"; an unset flag is "unset", never coerced to false |
| 6 | `derive-resource-classification.mjs` | `flows.json`, `connections.json`, `integrations.json`, `lint.json`, `connection_usage.json` | Additive tags: disabled / orphaned / test-dev-candidate / duplicate / production, plus a production subset | Tags are **additive** — nothing is deleted or hidden; the name regex and flagged names are emitted for operator review |
| 7 | `derive-subscription-usage.mjs` | `subscription_usage.json`, `licenses.json` | Processing-time totals, per-year, earliest/latest period | Empty or non-200 → `NA`, never a fabricated 0; the entitlement **%** stays a UI-screenshot item |
| 8 | `derive-stale-flows.mjs` | `flows.json` | Enabled-but-idle and never-run flows, bucketed never / >1y / 6-12m / 3-6m / <3m | Uses the persistent `lastExecutedAt`, so it is **not** bound by the short `/v1/jobs` retention |

> **Naming note.** The project docs refer to this set as "the 10 read-only
> derivation/fetch modules". On disk that is **8 `derive-*` + 4 `fetch-*` files**; the
> shorthand counts the eight *concerns* above with their paired fetchers, and
> `fetch-review-scripts.mjs` predates the accuracy workstream. The generator imports the
> **8 `derive-*`** modules and prints an availability flag for each on every run.

---

## 6. Phase 4 — Score (scoring model v1)

Finalised 2026-07-28. Applied **identically** in the live generator and the static
template, so the two deliverables can never disagree on method.

```mermaid
flowchart TD
    SG["For each of the 9 domains"] --> S1{"Is there a live signal for it?"}
    S1 -->|no| NA["Score = NA<br/>domain drops out entirely"]
    S1 -->|yes| S2["Score = round of the disclosed formula, 0-100"]
    NA --> S3["Collect only the SCORED domains"]
    S2 --> S3
    S3 --> S4["scoredWeight = sum of their weights"]
    S4 --> S5["Overall = round of sum of score x weight divided by scoredWeight<br/>i.e. weight-normalised mean, NA excluded and re-normalised"]
    S5 --> S6{"Which band?"}
    S6 -->|"0-59"| B1["Weak — red"]
    S6 -->|"60-74"| B2["Needs Improvement — orange"]
    S6 -->|"75-89"| B3["Satisfactory — yellow"]
    S6 -->|"90-100"| B4["Good — green"]
    B1 --> OUT["Section 1.1 banner + rating bar + progress bar<br/>Section 1.2 nine-domain scorecard"]
    B2 --> OUT
    B3 --> OUT
    B4 --> OUT
```

| # | Domain | Weight | Live signal the score is derived from |
|---|---|---|---|
| 1 | Security & Governance | 18% | mean of (SSO-linked users ÷ users) and (MFA-required users ÷ users); token rotation & audit log are UI-blocked |
| 2 | Error Management & Alerting | 16% | 50 points for having ≥1 notification subscription + 50 points for a zero open-error backlog |
| 3 | Connections & Authentication | 13% | online connections ÷ total connections |
| 4 | Flow Configuration | 12% | exports using delta or webhook sync ÷ total exports |
| 5 | Flow Lifecycle & Idle-Flow Hygiene | 10% | (active flows − idle/never-run active flows) ÷ active flows |
| 6 | Data Mapping & Transformation | 9% | imports with a field mapping ÷ total imports |
| 7 | Architecture & Tile Design | 9% | (all resources − orphaned exports/imports/connections) ÷ all resources |
| 8 | Scheduling & Performance | 8% | job success rate over the observed window |
| 9 | Documentation & Naming | 5% | not computed live this pass → **NA**, excluded |

**Worked example (the 2026-07-27 live run).** Eight domains scored, Documentation `NA`
→ `scoredWeight` = 95, weighted sum ÷ 95 = **43/100 🔴 Weak**. Because a `NA` domain is
removed *and* the denominator shrinks with it, an unmeasurable domain can never silently
drag the overall score up or down.

---

## 7. Phase 5 — Render (Markdown → DOCX)

```mermaid
flowchart TD
    R0(["Live Markdown report in reports/"]) --> R1["build-live-docx-full.py content out.docx cover_date"]
    R1 --> R2["Import build_celigo_template.py by file path<br/>via importlib — the shared converter is NEVER modified"]
    R2 --> R3["Monkeypatch COVER_SWAPS only<br/>neutral live cover, no fabricated client name, date passed in"]
    R3 --> R4["Call the converter's main()"]

    R4 --> T1["office/unpack.py explodes the V03 template<br/>into a temp dir: styles, theme, fonts, header/footer, logo"]
    T1 --> T2["Header fix, idempotent:<br/>rewrite any stale 'NetSuite Account Audit Report' running header<br/>to 'Celigo iPaaS Integration Audit Report',<br/>move the right tab stop to 10080 and right-align the title"]
    T2 --> T3["Cover swaps applied by regex on the cover's text runs only"]
    T3 --> T4["build_body walks the Markdown line by line"]
    T4 --> T5["office/pack.py rezips against the original<br/>OOXML schema validation happens here"]
    T5 --> T6(["The DOCX deliverable"])
```

### 7.1 How each Markdown construct is rendered

```mermaid
flowchart TD
    L["Read the next Markdown line"] --> D1{"Starts with '## '?"}
    D1 -->|yes| H1["Heading1 — 17pt bold cyan 35B4DA<br/>pageBreakBefore on every H1 after the first"]
    D1 -->|no| D2{"Starts with '### '?"}
    D2 -->|yes| H2["Heading2 — 14pt bold 1E8FAF"]
    D2 -->|no| D3{"Starts with '> '?"}
    D3 -->|yes| CO["callout(): a SINGLE-CELL cantSplit table box<br/>light-blue DEEAF6 + blue left bar<br/>+ an empty spacer paragraph after it"]
    D3 -->|no| D4{"Starts with a pipe character?"}
    D4 -->|yes| TB["table(): consume all consecutive pipe rows,<br/>drop separator rows, render as a real Word table"]
    D4 -->|no| PR["Body paragraph"]
    H1 --> L
    H2 --> L
    CO --> L
    TB --> L
    PR --> L
```

### 7.2 The table renderer's automatic formatting rules

| Trigger detected in the table | What the converter does |
|---|---|
| Header row is exactly `Metric \| Value \| Metric \| Value` | Draws a 1.5pt black **panel separator** down the middle boundary, so §1.3 reads as two panels |
| First row is a single cell above a wider header row | Renders it as a full-width **spanning title row** via `gridSpan` — this is how "Reviewers & Approvers" is titled |
| A column whose header is exactly `Score` | Boxes that column's **body** cells in bright green `43A047` |
| A body row containing `◀` | Boxes the whole **current-band row** green and renders `◀ current` bold + green |
| A cell containing a `█…░` bar | `_bar_runs()` renders bordered blocks, filled in the band colour keyed on the 🔴🟠🟡🟢 dot in the same cell, empty blocks grey `D9D9D9` |
| A cell starting with ✅ / ⚠️ / ❌ | Strips the icon and renders the remaining word **bold in that severity's colour** (green `2E7D32` / amber `E68A00` / red `C00000`) |
| A cell containing 🔴🟠🟡🟢 | **Left alone** — severity and rating dots stay as circular icons by design |

> **Why callouts are tables.** A `>` note used to be a shaded *paragraph*, which ghosted
> and duplicated itself when it landed on a page boundary. Rendering each as a
> single-cell `cantSplit` table makes it physically unsplittable. Consequence to
> remember: **callout boxes count toward the docx `<w:tbl>` total**, which is why the
> authoritative table counts live in one place only — `.agent/03_AUDIT_PLAYBOOK.md`.

---

## 8. Phase 6 — Verify (the gates)

```mermaid
flowchart TD
    V0(["Freshly built DOCX"]) --> V1{"OOXML schema-valid?<br/>enforced inside office/pack.py"}
    V1 -->|no| F1["PACK FAILED — build aborts, nothing is written"]
    V1 -->|yes| V2{"0 unfilled placeholders?"}
    V2 -->|no| F2["Reject — anti-fabrication gate"]
    V2 -->|yes| V3{"0 fabrication fingerprints?<br/>Acme, Demo Customer, TBD, Lorem, XXX"}
    V3 -->|no| F3["Reject"]
    V3 -->|yes| V4{"0 stale 'NetSuite Account Audit Report' headers?"}
    V4 -->|no| F4["Reject"]
    V4 -->|yes| V5{"Table count re-derived FROM DISK<br/>matches the note in .agent/03?"}
    V5 -->|no| F5["Investigate: either the build changed or the record drifted"]
    V5 -->|yes| V6["0 mojibake, structure check:<br/>15 numbered sections + Reviewers preamble"]
    V6 --> V7["Operator opens it in Word:<br/>pagination, no blank pages, no split callouts"]
    V7 --> V8(["Accepted deliverable"])
```

**Re-derive the counts, never quote them from memory:**

```bash
python -c "import zipfile,re;x=zipfile.ZipFile(P).read('word/document.xml').decode('utf8');t=re.findall(r'<w:tbl>.*?</w:tbl>',x,re.S);print(len(t),sum('DEEAF6' in i for i in t))"
```

---

## 9. Cross-cutting — the live-or-NA decision tree

This is the rule that governs **every single value** in the report. It is the project's
core anti-fabrication contract.

```mermaid
flowchart TD
    Q0["A data point is needed for the report"] --> Q1{"Can the read-only token read it<br/>directly from the API/CLI?"}
    Q1 -->|yes| A1["LIVE value"]
    Q1 -->|no| Q2{"Can it be COMPUTED from live data<br/>by auditor logic?"}
    Q2 -->|yes| A2["DERIVED value — disclose the basis"]
    Q2 -->|no| Q3{"Blocked by the token's role?<br/>401/403 'restricted to the token's own user'"}
    Q3 -->|yes| A3["Render 'UI Screenshot needed'<br/>admin supplies it into config/manual-inputs/"]
    Q3 -->|no| Q4{"Is it a business/client input<br/>that does not exist in the account?"}
    Q4 -->|yes| A4["Leave BLANK for the client to complete"]
    Q4 -->|no| A5["Render the literal NA"]

    A5 --> Q5{"Is the WHOLE COLUMN unresolvable<br/>for this account?"}
    Q5 -->|yes| A6["DROP the column entirely<br/>never a full column of NA<br/>restore it when a better token or a manual input resolves it"]
    Q5 -->|no| A7["Single-cell gap stays NA"]
```

| Class | Examples | Where it comes from |
|---|---|---|
| **Live** | tiles, flows, connections, exports/imports, scripts, users, licences, lint, jobs | Direct `GET` via proxy / CLI `mode=read` |
| **Derived** | MTTR, connection→flow usage, idle flows, success rate, schedule contention, scores | The 8 `derive-*` modules |
| **Auditor-generated** | methodology, severity definitions, remediation roadmap, glossary | Standard content the tool produces |
| **UI Screenshot needed** | account name/ID, sandbox, on-premise agents, audit log/owner, API tokens, entitlement %, per-user MFA-enabled, per-flow auto-retry, credential rotation date | Token-role blocked — 401/403 |
| **Blank** | reviewers & sign-off, business SLAs, RACI, RTO/RPO | Client business input |

**The token-role boundary is the single biggest scope lever:** an owner/admin token would
convert several "UI Screenshot needed" rows into live values with no code change.

---

## 10. Phase 7 — Governance: change, save, archive

```mermaid
flowchart TD
    D0["Day starts"] --> D1["Trigger: 'Hey Claude, I am here please ready'<br/>runs config/WorkStartWithThis/EveryDayStartPromp.md"]
    D1 --> D2["Steps 1-2: read CLAUDE.md, .agent/00-06, CONTRIBUTING,<br/>CHANGELOG, session index + recent archives"]
    D2 --> D3["Steps 3-4: list every file, RE-DERIVE every claim from DISK,<br/>reconcile record vs disk"]
    D3 --> D4["Step 5: report state, then STOP and wait"]

    D4 --> C0{"Operator asks for a change"}
    C0 --> C1["Trigger: 'make this change'<br/>runs config/precautions/01_making_changes.md"]
    C1 --> C2["A: 360-degree trace of the real source first — diagnose, never guess"]
    C2 --> C3["B: hard guardrails — read-only, live-or-NA, secrets stay in account.json"]
    C3 --> C4["C + C2: additive, backward-compatible,<br/>and it must work on EVERY future client account"]
    C4 --> C5["D: intermediate files to config/work/, deliverable only in reports/"]
    C5 --> C6["E: run the full pipeline and all gates"]
    C6 --> C7["F: keep it reversible — back up to the session scratchpad first"]

    C7 --> S0{"Operator says 'save the project'"}
    S0 --> S1["config/precautions/02_full_project_update.md"]
    S1 --> S2["B2: the save is NOT complete until all FOUR artifacts exist"]
    S2 --> S3["1. CHANGELOG.md entry"]
    S2 --> S4["2. memory update"]
    S2 --> S5["3. session_NN_DD_MM_YYYY_summary.jsonl, redaction-verified"]
    S2 --> S6["4. Files-table row + milestone in the session README"]
    S3 --> S7["SessionStart hook scripts/check-session-archive.mjs<br/>nudges at the next day-start if source looks un-archived"]
    S4 --> S7
    S5 --> S7
    S6 --> S7
    S7 --> D0
```

> **Known limitation of the hook (observed 2026-08-03).** `check-session-archive.mjs`
> compares the newest **source** mtime against the newest archive. A session that only
> regenerates the deliverable in `reports/` therefore does **not** trip it — exactly what
> happened on 2026-07-31, whose report regeneration is still un-archived. The day-start
> ritual, not the hook, remains the source of truth.

---

## 11. Traceability — node → source

Every diagram node above resolves to real code. Verified 2026-08-03.

| Flow node | File | Lines |
|---|---|---|
| Preflight (Node/CLI/mode/token) | [scripts/CeligoAuth.psm1](../scripts/CeligoAuth.psm1) `Assert-CeligoReady` | 101–106 |
| Read-verb whitelist / mutating list | [scripts/CeligoAuth.psm1](../scripts/CeligoAuth.psm1) | 25–35 |
| Default-deny verb test | [scripts/CeligoAuth.psm1](../scripts/CeligoAuth.psm1) `Test-ReadOnlyVerb` | 108–118 |
| Guarded CLI call, token via env | [scripts/CeligoAuth.psm1](../scripts/CeligoAuth.psm1) `Invoke-CeligoCli` | 120–142 |
| GET/HEAD-only REST wrapper | [scripts/CeligoAuth.psm1](../scripts/CeligoAuth.psm1) `Invoke-CeligoApi` | 144–155 |
| Proxy helpers (start/stop/health/get/raw) | [scripts/CeligoAuth.psm1](../scripts/CeligoAuth.psm1) | 165–239 |
| Proxy fail-closed startup checks | [scripts/celigo-readonly-proxy.mjs](../scripts/celigo-readonly-proxy.mjs) | 29–61 |
| Access log, token redacted | [scripts/celigo-readonly-proxy.mjs](../scripts/celigo-readonly-proxy.mjs) | 63–71 |
| Health endpoint | [scripts/celigo-readonly-proxy.mjs](../scripts/celigo-readonly-proxy.mjs) | 78–82 |
| **The 405 enforcement** | [scripts/celigo-readonly-proxy.mjs](../scripts/celigo-readonly-proxy.mjs) | 85–95 |
| Token injection + forward | [scripts/celigo-readonly-proxy.mjs](../scripts/celigo-readonly-proxy.mjs) | 97–119 |
| Snapshot loop, empty→`[]`, per-resource NA | [scripts/extract-celigo.ps1](../scripts/extract-celigo.ps1) | 37–55 |
| Latest-snapshot resolution | [audit-docx-tool/live-prototype/gen-live-report-full.mjs](../audit-docx-tool/live-prototype/gen-live-report-full.mjs) | 28–42 |
| Snapshot loads with graceful defaults | [gen-live-report-full.mjs](../audit-docx-tool/live-prototype/gen-live-report-full.mjs) | 44–58 |
| Region derived from config | [gen-live-report-full.mjs](../audit-docx-tool/live-prototype/gen-live-report-full.mjs) | 60–62 |
| The 8 derive modules invoked | [gen-live-report-full.mjs](../audit-docx-tool/live-prototype/gen-live-report-full.mjs) | 64–72 |
| **Scoring model v1** | [gen-live-report-full.mjs](../audit-docx-tool/live-prototype/gen-live-report-full.mjs) | 90–120 |
| Report body emission (§1–§15 + appendices) | [gen-live-report-full.mjs](../audit-docx-tool/live-prototype/gen-live-report-full.mjs) | 122–420 |
| Markdown written to `reports/` | [gen-live-report-full.mjs](../audit-docx-tool/live-prototype/gen-live-report-full.mjs) | 422–426 |
| Converter imported, cover monkeypatched | [build-live-docx-full.py](../audit-docx-tool/live-prototype/build-live-docx-full.py) | 22–37 |
| Stale running-header fix | [audit-docx-tool/scripts/build_celigo_template.py](../audit-docx-tool/scripts/build_celigo_template.py) | 349–365 |
| Cover text swaps | [build_celigo_template.py](../audit-docx-tool/scripts/build_celigo_template.py) | 372–374 |
| Markdown dispatch (`##`/`###`/`>`/`\|`) | [build_celigo_template.py](../audit-docx-tool/scripts/build_celigo_template.py) `build_body` | 297–325 |
| Table auto-rules (separator, span, Score, ◀) | [build_celigo_template.py](../audit-docx-tool/scripts/build_celigo_template.py) `table` | 205–256 |
| Cell rules (result marks, borders, span) | [build_celigo_template.py](../audit-docx-tool/scripts/build_celigo_template.py) `cell` | 152–203 |
| Progress bar renderer | [build_celigo_template.py](../audit-docx-tool/scripts/build_celigo_template.py) `_bar_runs` | 96–120 |
| `cantSplit` callout box | [build_celigo_template.py](../audit-docx-tool/scripts/build_celigo_template.py) `callout` | 272–295 |
| Pack + OOXML validation | [build_celigo_template.py](../audit-docx-tool/scripts/build_celigo_template.py) | 381–385 |
| Session-archive reminder hook | [scripts/check-session-archive.mjs](../scripts/check-session-archive.mjs) + `.claude/settings.json` | — |

---

## 12. On disk but NOT in the flow

Drawn here so the chart is **complete as well as correct** — these files exist but no
arrow in the working pipeline touches them.

```mermaid
flowchart LR
    subgraph STUB["Stubs — declared, not implemented"]
        A["scripts/run-report.ps1 — 2-line TODO"]
        B["scripts/generate-report.ps1 — 2-line TODO"]
        C["scripts/run-audit.ps1 — preflight works,<br/>check orchestration is a TODO"]
    end
    subgraph DORM["Dormant NetSuite-era code"]
        D["generate_fresh_values.py — queries NetSuite SuiteQL"]
        E["build_audit_docx.py — NetSuite placeholder template"]
        F["check_static_data.py / smoke_test.py / inject_placeholders.py"]
        G["build.ps1 + scripts/auto_build_hook.ps1 — PostToolUse auto-build,<br/>NOT wired in .claude/settings.json"]
    end
    subgraph SUPER["Superseded but kept for history"]
        H["gen-live-report.mjs — the 2026-07-13 prototype"]
        I["build_live_docx.py — its renderer"]
    end
```

- The **stubs** are safe: `run-audit.ps1` still runs the full read-only preflight before
  reporting that orchestration is unimplemented.
- The **dormant NetSuite scripts** lost their target when the `PLACEHOLDERS` template was
  deleted on 2026-07-10. `check_static_data.py` and `smoke_test.py` are therefore **not
  runnable build gates today** — the gates actually in force are the ones in §8.
- The **superseded prototype** is retained deliberately as the artifact the frozen data
  scope was first validated against.

---

## 13. The exact command sequence

A complete audit, from a configured machine to a delivered report. Steps 1–3 are the only
ones that contact Celigo.

```powershell
# --- ONE TIME -------------------------------------------------------------
Copy-Item config\account.example.json config\account.json   # then fill in apiToken
.\scripts\bootstrap.ps1                                      # preflight + read-only connectivity test

# --- 1. Start the read-only egress proxy (its own terminal) ---------------
.\scripts\proxy.ps1                                          # GET/HEAD only; 405s every mutation

# --- 2. Extract the core snapshot ----------------------------------------
.\scripts\extract-celigo.ps1                                 # -> config/work/snapshot_<DD_MM_YYYY>/

# --- 3. Supplementary live reads (guarded CLI, mode=read) + fetch modules -
#     users list | subscriptions licenses | notifications list |
#     account stats | account lint | jobs list --created-gte <date>
node audit-docx-tool/live-prototype/fetch-review-scripts.mjs     <projectRoot>
node audit-docx-tool/live-prototype/fetch-connection-usage.mjs   <projectRoot>
node audit-docx-tool/live-prototype/fetch-flow-errors.mjs        <projectRoot>
node audit-docx-tool/live-prototype/fetch-subscription-usage.mjs <projectRoot>
# Stop the proxy here — nothing below touches Celigo.

# --- 4. Derive + score + emit Markdown (OFFLINE) --------------------------
node audit-docx-tool/live-prototype/gen-live-report-full.mjs <projectRoot> [snapshotDir]

# --- 5. Render the DOCX (OFFLINE) -----------------------------------------
python audit-docx-tool/live-prototype/build-live-docx-full.py `
       reports/Celigo_Audit_Report_LIVE_<DD_MM_YYYY>.md `
       reports/Celigo_Audit_Report_LIVE_<DD_MM_YYYY>.docx `
       "<DD Mon YYYY>"

# --- 6. Verify (§8), then keep reports/ to the deliverable + logs/ only ----
```

The **static illustrative template** is a separate, parallel deliverable built from its
own source with the same shared converter — it never touches Celigo:

```powershell
python audit-docx-tool/scripts/build_celigo_template.py
# defaults: templates/celigo_audit_report_v01_source.md -> templates/celigo_audit_report_template_v01.docx
```

---

*Document version 1.0 — authored 2026-08-03. Every node traced from source files on
disk; no value in this document was taken on trust from another document. Read-only:
producing this document made no Celigo call and changed no code.*
