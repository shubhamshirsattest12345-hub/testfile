# AI NetSuite Auditing Tool — Project Abstract

The AI NetSuite Auditing Tool (**NetSuite Audit AI**) is an automated health-audit tool for **NetSuite ERP** accounts.

It installs its own audit engine into the account through **SuiteCloud Development Framework (SDF)** — 24 SuiteScript files and 28 object definitions, nothing installed by hand — then triggers that engine on demand, **reads** the whole setup (scripts, deployments, execution logs, workflows, saved searches, analytics datasets and workbooks, custom forms, roles, users, login history, transactions, bundles, integrations, company configuration) and writes the raw findings as JSON into the account's own File Cabinet.

Everything after that runs on the operator's machine: a generator pulls those result files plus live SuiteQL queries and turns them into **401 report values**, and a builder renders them into the finished Word report.

From that data the tool scores the account across **34 risk-checks grouped into 3 weighted pillars** (Performance 33%, Optimization 33%, Security 34%) into a single **Overall Account Health score out of 100**.

The output is a professionally formatted **Word audit report** — 9 sections covering the executive dashboard, scope and methodology, the three pillar audits, the customization summary, the full account inventory, a prioritised remediation roadmap and the appendices.

Two rules are absolute: the tool **never creates, changes or deletes anything the audit framework did not create itself** in the client's account — findings are reported with remediation steps for the client's own team to apply — and it **never invents a value**: anything it cannot read live is shown honestly as the literal `NA`, and the build is blocked outright if a single fabricated or leftover value would ship.

The same process runs unchanged against **any** NetSuite account — production or sandbox, Administrator or least-privilege role — with nothing hardcoded but the contents of one gitignored credentials file.

---

*Document version 1.0 — authored 2026-08-03, traced from the source files on disk. Producing it made no NetSuite call and changed no code.*
