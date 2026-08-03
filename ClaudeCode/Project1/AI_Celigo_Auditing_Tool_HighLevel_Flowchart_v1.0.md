# AI Celigo Auditing Tool — High-Level Process Flow

**In one sentence:** the tool takes a single read-only snapshot of a Celigo account,
analyses it offline, scores its health out of 100, and produces a Word audit report with
a remediation roadmap — without ever changing anything in the client's account.

```mermaid
flowchart TD
    A["1 · CONNECT — read-only<br/>the tool can only read; every change request is blocked"]
    B["2 · SNAPSHOT the account<br/>tiles, flows, connections, mappings, scripts,<br/>users, errors, run history"]
    C["3 · DISCONNECT<br/>all analysis runs offline on the snapshot"]
    D["4 · ANALYSE<br/>configuration, run health, errors, security,<br/>idle flows, scheduling, orphaned resources"]
    E{"Can the value be read<br/>from the account?"}
    F["Use the live or calculated value"]
    G["Mark it NA, 'UI screenshot needed',<br/>or leave it blank for the client<br/>— never a guessed number"]
    H["5 · SCORE 9 audit domains<br/>each weighted by risk"]
    I["6 · OVERALL ACCOUNT HEALTH<br/>a single 0–100 score with a Weak / Needs Improvement /<br/>Satisfactory / Good rating"]
    J["7 · GENERATE the Word audit report<br/>findings, evidence and a remediation roadmap"]
    K{"Quality checks pass?<br/>nothing invented, nothing left blank by mistake"}
    L["8 · DELIVER to the client"]
    M["The client's team applies the fixes<br/>— the tool never changes the account itself"]

    A --> B --> C --> D --> E
    E -->|"yes"| F --> H
    E -->|"no"| G --> H
    H --> I --> J --> K
    K -->|"no"| J
    K -->|"yes"| L --> M
    M -.->|"re-run any time for a fresh audit"| A

    classDef step fill:#eaf3fb,stroke:#1d57be,color:#12325c;
    classDef check fill:#fff8e1,stroke:#e8a33d,color:#3b3b3b;
    classDef result fill:#eafaef,stroke:#2e7d32,color:#14421c;
    class A,B,C,D,F,G,H,J step;
    class E,K check;
    class I,L,M result;
```

### The 9 scored domains

| Domain | Weight | Domain | Weight |
|---|---|---|---|
| Security & Governance | 18% | Data Mapping & Transformation | 9% |
| Error Management & Alerting | 16% | Architecture & Tile Design | 9% |
| Connections & Authentication | 13% | Scheduling & Performance | 8% |
| Flow Configuration | 12% | Documentation & Naming | 5% |
| Flow Lifecycle & Idle Flows | 10% | | |

### Three things that make this audit trustworthy

- **It cannot break anything.** The account is only ever read. Any attempt to create,
  change, delete or run something is blocked before it reaches Celigo.
- **Nothing is invented.** Every number in the report comes from the account or is
  calculated from it. Anything that can't be read is shown honestly as `NA`, as a value
  the client's admin must screenshot, or left blank — never filled with a guess.
- **It's repeatable and account-agnostic.** The same process runs against any Celigo
  account, and a report can be regenerated from a snapshot at any time without going
  back to the live account.

---

*High-level view — for a detailed technical breakdown see
[AI_Celigo_Auditing_Tool_Consolidated_Flowchart_v1.0.md](AI_Celigo_Auditing_Tool_Consolidated_Flowchart_v1.0.md)
(full single chart) or [AI_Celigo_Auditing_Tool_Process_Flowchart_v1.0.md](AI_Celigo_Auditing_Tool_Process_Flowchart_v1.0.md)
(phase-by-phase).*
