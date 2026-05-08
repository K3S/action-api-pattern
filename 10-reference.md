---
title: Reference
nav_order: 11
permalink: /reference/
---

# Reference
{: .no_toc }

Quick-reference tables for daily use. The earlier chapters explain the *why*; this page is the lookup card you keep open while you're writing code.

1. TOC
{:toc}

---

## Program prefix conventions

| Prefix | Type | Purpose |
|--------|------|---------|
| `AC_` | CL `*PGM` | The Action wrapper. Catches messages, opens/closes the API log, calls the RPG. This is what external clients invoke. |
| `AR_` | RPG `*PGM` | The Action implementation. The actual business logic. Called only by `AC_<verb>`. |
| `TS_` | CL `*PGM` | The test harness. Hardcoded sample parameters, calls `AC_<verb>`, displays the result via `SNDPGMMSG`. Runnable from a 5250 command line. |
| `SP_` | RPG `*SRVPGM` | Shared service-program procedures. Validation helpers, key generators, lookups used by multiple Actions. |
| `AR_LOG*` | RPG `*PGM` | Logging Actions (`AR_LOGOPN`, `AR_LOGCLS`). Themselves Action APIs, called from every `AC_<verb>`. |

## Parameter prefix conventions

| Prefix | Direction | Meaning |
|--------|-----------|---------|
| `ID...` | input | **Identifier.** A field used to find or identify a record. Immutable in update operations. |
| `SB...` | input | **Submit value.** A new value being submitted as input. Updatable. |
| `RV...` | output | **Return value.** A value the program returns to the caller. |
| (no prefix) | input/output | The standard envelope: `K3SOBJ`, `COMP`, `COMPCOD`, `USER`, `ERRORS`, `ERRMSG`, `ERRFIELD`. |

## Standard envelope (every Action, every time)

| Position | Name | Type | Direction | Notes |
|----------|------|------|-----------|-------|
| 1 | `K3SOBJ` | char(10) | input | K3S object library name |
| 2 | `COMP` | char(1) | input | Company indicator |
| 3 | `COMPCOD` | char(3) | input | Company code |
| 4 | `USER` | char(10) | input | Calling user's IBM i profile |
| 5 | `ERRORS` | char(1) | output | `Y` = error, `N` = success |
| 6 | `ERRMSG` | char(100) | output | Human-readable message if `ERRORS=Y` |
| 7 | `ERRFIELD` | char(20) | output | Parameter name that caused the error |

After position 7, action-specific parameters follow in the order: identifiers (`ID*`), submitted values (`SB*`), return values (`RV*`).

## Verb conventions

### Commands (mutate state)

| Verb | Meaning | Typical inputs | Typical outputs |
|------|---------|----------------|-----------------|
| `ADD` | Create a new record | parent `ID*` + `SB*` for the new record | optional `RV*` for generated key |
| `UPD` | Update an existing record | record `ID*` + `SB*` for changed fields | none |
| `DEL` | Delete a record | record `ID*` only | none |
| `CHG` | Change a single attribute (alias of `UPD` for narrow updates) | record `ID*` + one `SB*` | none |
| `CPY` | Copy a record to a new identifier | source `ID*` + destination `SB*` | optional `RV*` for new key |
| `RUN` | Execute a process or batch run | run parameters as `SB*` | optional `RV*` for run ID, count |
| `PRC` | Process a queue or batch (alias for `RUN` in older code) | as above | as above |

### Queries (read state)

| Verb | Meaning | Typical inputs | Typical outputs |
|------|---------|----------------|-----------------|
| `GET` | Retrieve a single record | record `ID*` | scalar `RV*` for each field |
| `LST` | List records matching filters | filter `ID*`/`SB*`, paging | result set (user space, work file, SQL cursor, or JSON) |
| `SCH` | Search (similar to `LST`, often free-text) | search `SB*`, paging | as `LST` |
| `CNT` | Count matching records | filter `ID*`/`SB*` | `RVCOUNT` |
| `RPT` | Generate a report | report parameters as `SB*` | report output (spool, IFS file, or JSON) |

## Error indicator semantics

| `ERRORS` | Meaning | HTTP status when called from PHP | MCP behavior |
|----------|---------|----------------------------------|--------------|
| `N` | Success. `ERRMSG` and `ERRFIELD` are empty. | 200 | `isError: false` |
| `Y` | Failure. `ERRMSG` is human-readable. `ERRFIELD` may identify the offending parameter. | 400 (validation) or 500 (toolkit) | `isError: true`, `text` includes `ERRMSG` |
| anything else | **Protocol violation.** Treat as failure. Log loudly. | 500 | `isError: true` |

`ERRORS=Y` from a normal validation path differs from `ERRORS=Y` set by `MONMSG` in the CL wrapper. The latter triggers a support-team email via the API log; the former is a routine user-facing error.

## Result-set return mechanisms (queries)

| Approach | When to use | Pros | Cons |
|----------|-------------|------|------|
| User space (`*USRSPC`) | Caller is another RPG/CL program | Native, fast, no SQL overhead | Hard to consume from PHP/MCP |
| Work file | Caller needs to re-read or paginate | Easy SQL access, supports paging | Cleanup required, ties contract to physical file |
| SQL result set | Caller is PHP via toolkit, or any SQL-aware client | Modern, paginates cleanly, integrates with `getResultSets()` | Caller must know how to fetch result sets |
| JSON emit | Caller is MCP or any JSON-native consumer | Universal, no transformation | Verbose to produce in RPG, size limits |

## API log schema (high level)

| Field | Type | Notes |
|-------|------|-------|
| Log ID | char(20) | Unique key, generated in `AR_LOGOPN` |
| Program name | char(10) | The `AR_<verb>` being invoked |
| User | char(10) | Calling user's IBM i profile |
| Start timestamp | timestamp | When `AC_<verb>` opened the log entry |
| End timestamp | timestamp | When `AC_<verb>` closed the log entry |
| Duration (ms) | int | End minus start |
| Errors flag | char(1) | Final `ERRORS` value |
| Error message | char(100) | Final `ERRMSG` value |
| Error classification | char(1) | `V`alidation, `M`onitored exception, `E`mpty (success) |
| Parameter snapshot | varchar | Serialized inputs (sanitized — no PII) |

Logs are queryable through SQL on the underlying physical file. Standard retention is 30 days.

## File and library naming

| Object | Convention | Example |
|--------|------------|---------|
| Action library (objects) | `K3S_<n>OBJ` per environment | `K3S_5OBJ` (production), `K3S_5DEV` (dev) |
| Source file library | `K3S_<n>SRC` | `K3S_5SRC` |
| Service program library | `K3S_<n>BND` (binding directory lives here) | `K3S_5BND` |
| Source files | `QRPGLESRC` (RPG), `QCLSRC` (CL), `QDDSSRC` (DDS) | standard IBM names |
| Member name | matches program name exactly | `AR_DELSUPL` source in `K3S_5SRC/QRPGLESRC(AR_DELSUPL)` |
| Database files | `K_<entity>` | `K_SUPPL`, `K_PORDER`, `K_BUYER` |

The numeric digit (`5` in `K3S_5OBJ`) is a customer-environment identifier in K3S deployments. Adapt to your own scheme.

## Compile commands cheat sheet

```cl
/* RPG implementation (single source program with bind directory) */
CRTBNDRPG PGM(K3S_5OBJ/AR_DELSUPL) +
          SRCFILE(K3S_5SRC/QRPGLESRC) +
          SRCMBR(AR_DELSUPL) +
          DBGVIEW(*SOURCE) +
          BNDDIR(K3S_5BIND)

/* CL wrapper */
CRTBNDCL  PGM(K3S_5OBJ/AC_DELSUPL) +
          SRCFILE(K3S_5SRC/QCLSRC) +
          SRCMBR(AC_DELSUPL)

/* Test program */
CRTBNDCL  PGM(K3S_5OBJ/TS_DELSUPL) +
          SRCFILE(K3S_5SRC/QCLSRC) +
          SRCMBR(TS_DELSUPL)

/* Service program (when adding a new SP_ procedure) */
CRTRPGMOD MODULE(K3S_5BND/SP_VALIDATE) +
          SRCFILE(K3S_5SRC/QRPGLESRC) +
          SRCMBR(SP_VALIDATE)
CRTSRVPGM SRVPGM(K3S_5BND/SP_VALIDATE) +
          MODULE(K3S_5BND/SP_VALIDATE) +
          EXPORT(*ALL)
```

## Common MONMSG IDs in CL wrappers

| Message ID | Meaning | Typical handling |
|------------|---------|------------------|
| `CPF0000` | Any CPF message | Catch-all for system-level failures |
| `MCH0000` | Any machine-check message | Catch-all for low-level failures |
| `CPF2103` | Library already in library list | Ignore (already added) |
| `CPF2104` | Library not in library list | Ignore (already removed) |
| `CPF9999` | Function check | Logged; treated as fatal |

The standard wrapper monitors `CPF0000 MCH0000` around the `CALL AR_<verb>` and translates any caught message into the error envelope.

## File reference for this site

| Page | Purpose |
|------|---------|
| [Why Headless]({% link 01-why-headless.md %}) | Motivation; why the pattern matters in 2026 |
| [The Action API Pattern]({% link 02-action-api-pattern.md %}) | Formal contract; AC_/AR_/TS_, envelope, prefixes, MONMSG, logging |
| [Decomposing Supplier Maintenance]({% link 03-decomposing-supplier-maintenance.md %}) | Worked example: turning SPMAINT into Actions |
| [Writing Your First Action]({% link 04-first-action.md %}) | Full code: AR_DELSUPL, AC_DELSUPL, TS_DELSUPL |
| [Commands and Queries]({% link 05-commands-queries.md %}) | CQRS framing, return-mechanism choices |
| [Calling Actions from PHP]({% link 06-calling-from-php.md %}) | Mezzio handler pattern, AbstractActionHandler |
| [Calling Actions from MCP]({% link 07-calling-from-mcp.md %}) | TypeScript MCP server, manifest-driven tools |
| [What You Don't Do]({% link 08-discipline.md %}) | Anti-patterns, mode flags, what not to share |
| [Migrating Existing Programs]({% link 09-migrating.md %}) | Strangler fig, per-customer flags, equivalence testing |
| Reference (this page) | Quick-reference tables |

---

## Where to go next

The Action API Pattern is the middle of a series of K3S open-source IBM i tutorials:

- **[RPG Tutorial](https://rpgtutorial.k3s.com)** — Modern RPGLE and CL with VS Code. Best starting point if you're new to RPG.
- **Action API Pattern** — *this site*
- **[IBM i AI Workers](https://ibmi-ai-workers.k3s.com)** — Calling LLMs at scale from RPG batch jobs. The natural sequel for teams ready to add AI.

Other tutorials in development cover DB2 for i patterns, CL programming in depth, PHP on IBM i, and MCP server construction. Watch the [K3S GitHub organization](https://github.com/K3S) for releases.

If you spot something missing or wrong on this page, the *Edit this page on GitHub* link in the footer takes you straight to a PR. Contributions welcome.
