# AI Celigo Auditing Tool — Project Abstract

The AI Celigo Auditing Tool is an automated health-audit tool for **Celigo integrator.io** accounts.

It connects to an account **read-only**, takes a single snapshot of the whole setup — integration tiles, flows, connections, mappings, scripts, users, errors and run history — and then analyses that snapshot completely offline.

From it, the tool scores the account across **9 risk-weighted audit domains** (security, error management, connections, flow configuration, idle flows, data mapping, architecture, scheduling, documentation) into a single **Overall Account Health score out of 100**.

The output is a professionally formatted **Word audit report** containing the health scorecard, the detailed findings, the evidence behind each one, and a prioritised remediation roadmap.

Two rules are absolute: the tool **never creates, changes, deletes or runs anything** in the client's account, and it **never invents a value** — anything it cannot read is shown honestly as `NA`, flagged for an admin screenshot, or left blank.

The same process runs unchanged against **any** Celigo account, and a report can be regenerated from a snapshot at any time without going back to the live account.
