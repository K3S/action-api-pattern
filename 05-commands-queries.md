---
title: Commands and Queries
nav_order: 6
permalink: /commands-and-queries/
---

# Commands and Queries
{: .no_toc }

1. TOC
{:toc}

---

## Two kinds of Actions

Every Action API is one of two kinds:

- **A command** changes state. The supplier file gets a new row, an existing row gets updated, or a row gets deleted. The caller wants to know whether it succeeded.
- **A query** reads state. The supplier file is unchanged. The caller wants the data.

This is the core idea behind **Command-Query Responsibility Segregation (CQRS)**, an architectural pattern from the broader software world. CQRS says: don't mix the two. The thing that mutates state should not also return data, and the thing that returns data should not mutate state. Separate them, and the system gets simpler.

Apply that to Action APIs and you get a clean rule: every program is either a command or a query, never both. The verb tells you which.

| Kind | Verb prefixes | Returns |
|------|---------------|---------|
| **Command** | `ADD`, `UPD`, `DEL`, `CHG`, `CPY`, `RUN`, `PRC` | Standard envelope (`ERRORS`/`ERRMSG`/`ERRFIELD`), optionally a generated ID via `RV*` |
| **Query** | `GET`, `LST`, `SCH`, `CNT`, `RPT` | Standard envelope plus the data, either as `RV*` scalars or as a result set |

`AR_DELSUPL`, `AR_ADDSUPL`, `AR_UPDSUPL` are commands. `AR_GETSUPL`, `AR_LSTSUPL` are queries. The shape of the parameter list and the implementation strategy differ in subtle but important ways between the two.

## How commands look

Commands have a uniform shape: identifiers in (`ID*`), submitted values in (`SB*`), error envelope out, optionally a few `RV*` outputs that report something the command produced (a new internal ID, a count of affected rows, a generated reference number).

`AR_DELSUPL`, from the previous chapter, is the simplest command: identifiers in, envelope out, no `SB*`, no `RV*`. A pure delete.

`AR_ADDSUPL` is more typical:

```
inputs:
  K3SOBJ, COMP, COMPCOD, USER       (envelope)
  IDBUYR, IDLOCN                     (the new supplier's parent context)
  SBSUPL, SBSUPLSUB                  (the new supplier code)
  SBSUPLNAM                          (the name)
  SBPHONE, SBADDR1, SBADDR2          (contact info)
  SBCITY, SBSTATE, SBZIP             (address)
  SBLEADTIME, SBORDCYCLE             (replenishment defaults)
outputs:
  ERRORS, ERRMSG, ERRFIELD           (envelope)
  RVSUPLID                           (the internal ID assigned)
```

Note the asymmetry: `IDBUYR` and `IDLOCN` are `ID*` because they identify the *parent* — the buyer and location the new supplier is being created under. `SBSUPL` is `SB*`, not `ID*`, because it's a value being submitted, not an identifier of an existing record.

This distinction matters. When `AR_UPDSUPL` runs, `SBSUPL` doesn't exist — the supplier code is `IDSUPL`, because at update time the supplier already exists and is being identified, not submitted. The same field plays a different role in different commands. The prefix tells the program which.

`AR_UPDSUPL`'s shape:

```
inputs:
  K3SOBJ, COMP, COMPCOD, USER
  IDBUYR, IDLOCN, IDSUPL, IDSUPLSUB  (the compound key — find this row)
  SBSUPLNAM                          (any updatable field becomes SB*)
  SBPHONE, SBADDR1, SBADDR2
  SBCITY, SBSTATE, SBZIP
  SBLEADTIME, SBORDCYCLE
outputs:
  ERRORS, ERRMSG, ERRFIELD
```

No `RV*` — the update doesn't produce anything new for the caller to consume. The success/failure envelope is enough.

### Idempotency and command design

A well-designed command is *idempotent under a given input* — calling it twice with the same parameters produces the same final state, even if the second call is a no-op or an error.

`AR_DELSUPL` is naturally idempotent: the first call deletes, the second call returns "supplier not found." The end state is the same — the supplier doesn't exist.

`AR_UPDSUPL` is naturally idempotent: applying the same updates twice yields the same row.

`AR_ADDSUPL` is *not* idempotent under naive design: calling it twice with the same `SBSUPL` would either create two suppliers (if your file allows duplicate keys) or fail the second time. The right design fails the second call cleanly with `ERRORS=Y`, `ERRMSG=Supplier already exists`, `ERRFIELD=SBSUPL`. That gives the caller a deterministic outcome, even though it isn't strictly idempotent in the mathematical sense.

Idempotency matters because callers — especially HTTP-based ones — retry. Network blips happen. A command that creates duplicates on retry is a bug waiting to surface in production.

{: .tip }
> If you find yourself designing a command where retry would be dangerous (e.g., `AR_CHARGE_CARD`), build idempotency in explicitly. Add a caller-supplied request ID, store it, refuse to process the same ID twice, and return the original response on retry. This is what payment APIs do.

## How queries look

Queries return data. The way they return it depends on the shape of the result.

### Single-record queries: `GETSUPL`

If the result is a single record with a known set of fields, return it through `RV*` parameters:

```
inputs:
  K3SOBJ, COMP, COMPCOD, USER
  IDBUYR, IDLOCN, IDSUPL, IDSUPLSUB
outputs:
  ERRORS, ERRMSG, ERRFIELD
  RVSUPLNAM
  RVPHONE, RVADDR1, RVADDR2
  RVCITY, RVSTATE, RVZIP
  RVLEADTIME, RVORDCYCLE
  RVCREATED, RVCREATEDBY
  RVMODIFIED, RVMODIFIEDBY
```

The implementation is mechanical: `chain` to find the row, copy fields to `RV*` parameters, return. If not found, set the error envelope.

```rpgle
chain (IDBUYR : IDLOCN : IDSUPL : IDSUPLSUB) K_SUPPL;
if not %found(K_SUPPL);
  ERRORS   = 'Y';
  ERRMSG   = 'Supplier not found';
  ERRFIELD = 'IDSUPL';
  return;
endif;

RVSUPLNAM    = SUPNAM;
RVPHONE      = SUPPHN;
RVADDR1      = SUPAD1;
// ... etc.

return;
```

Single-record queries are easy. The pain begins with result sets.

### Multi-record queries: `LSTSUPL`

`LSTSUPL` returns a *list* of suppliers matching some filter. Returning a variable-length list through scalar parameters doesn't work; we need a different mechanism.

There are four reasonable approaches. We'll cover all four because each has a use case.

#### Approach 1: User space

Write the result rows into a user space (`*USRSPC`), and return its qualified name through `RV*` parameters. The caller is responsible for reading the user space.

```
outputs:
  ERRORS, ERRMSG, ERRFIELD
  RVUSRSPC      (10-char user space name)
  RVUSRLIB      (10-char library)
  RVCOUNT       (number of records written)
```

Pros: native IBM i pattern, very fast for large result sets, the calling RPG program can read it directly. Cons: the caller has to know how to parse a user space, which is hard for PHP or any non-RPG client.

User spaces are the right answer when *another RPG program* is the caller (a batch process iterating over suppliers). They're awkward when PHP or MCP is the caller.

#### Approach 2: Work file

Write the result rows into a work file (a physical file in `QTEMP` or a designated work library), keyed by the calling user or job. The caller reads the work file.

```
outputs:
  ERRORS, ERRMSG, ERRFIELD
  RVWRKFILE     (10-char file name)
  RVWRKLIB      (10-char library)
  RVCOUNT       (number of records)
```

Pros: easy for any IBM i caller to read, supports SQL, supports paging. Cons: requires cleanup (the work file has to be cleared eventually), and the work file's record format ties the API contract to a specific physical layout.

Work files are common in K3S for queries that return wide records or need to be re-read multiple times.

#### Approach 3: SQL result set

Open an SQL result set inside the RPG and let the caller fetch from it. This is `*N` parameter style with a result set indicator.

```sql
DECLARE C1 CURSOR FOR
  SELECT * FROM K_SUPPL
  WHERE BUYR = :IDBUYR AND LOCN = :IDLOCN
  ORDER BY SUPL;

OPEN C1;
SET RESULT SETS CURSOR C1;
```

Pros: integrates cleanly with PHP/PDO, supports paging via `FETCH FIRST n ROWS ONLY` and `OFFSET`, very efficient. Cons: requires the calling environment to know how to fetch result sets (the IBM i Toolkit has `getResultSets()` for this).

SQL result sets are arguably the modern best answer for queries called from PHP or other SQL-aware clients. K3S uses this pattern in newer Action APIs.

#### Approach 4: JSON emit

Have the RPG generate a JSON string of the result and return it as a single large character output (or write it to a stream file). The caller parses the JSON.

```rpgle
dcl-pi *n;
  // ... envelope and inputs ...
  RVJSON char(65535);   // up to 64K of result
  RVCOUNT int(10);
end-pi;
```

Pros: works with any client, easy to produce with YAJL or `JSON_OBJECT`/`JSON_ARRAY` SQL functions. Cons: 64K may be too small; you may need to write to IFS instead. JSON serialization in RPG is verbose.

JSON emit is most useful when the *MCP server* is the caller — MCP wants JSON anyway, so producing it once at the source eliminates a transformation layer. We'll see this in [Calling Actions from MCP]({% link 07-calling-from-mcp.md %}).

### When to use which

A useful default:

| Caller | Approach |
|--------|----------|
| Another RPG/CL program | User space |
| PHP via the toolkit | SQL result set or work file |
| Web API (REST/JSON) | SQL result set, transformed to JSON in PHP |
| MCP tool | JSON emit (or SQL result set + PHP transform) |
| Batch report | Work file |

The K3S codebase mixes these. Older Actions tend to use user spaces; newer ones use SQL result sets. Any of them can be wrapped in PHP that produces JSON for the web frontend.

## Should commands and queries share the wrapper pattern?

Yes. Both kinds of Actions follow the same `AC_/AR_/TS_` trio, the same envelope, the same logging. The only thing that changes is the shape of the parameter list and the choice of return mechanism.

This is deliberate. The pattern is *the shape of an Action*. Whether a particular Action mutates or reads state is implementation detail; the contract — three programs, standard envelope, prefix conventions, MONMSG handling, logged invocation — is constant.

Some teams take CQRS further and split commands and queries into separate libraries (`K3S_CMD` vs `K3S_QRY`), separate buyer profiles, even separate Db2 schemas (read replicas for queries, primary for commands). That's a scaling choice you might make at very large volumes. We don't do it in the standard K3S deployment. The boundary that matters is at the *Action* level — one verb, one program, one purpose — not at the library level.

## A useful mental model

Think of it this way. A command is a *function with side effects that returns a status*. A query is a *pure function that returns data*. Either one is a single-purpose program. Either one has a single, predictable response shape. Either one can be tested by `TS_<verb>` from a 5250 command line.

The CQRS framing is mostly a tool for thinking. When someone proposes a new Action, ask:

- Is it a command or a query?
- If a command, is it idempotent? If not, does it need to be made idempotent?
- If a query, what shape is the result? Single record or set? Which return mechanism fits the calling environment?

Three questions, asked at design time, save hours of refactoring later.

[Next: Calling Actions from PHP]({% link 06-calling-from-php.md %}){: .btn .btn-primary }
