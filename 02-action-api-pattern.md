---
title: The Action API Pattern
nav_order: 3
permalink: /action-api-pattern/
---

# The Action API Pattern
{: .no_toc }

1. TOC
{:toc}

---

This chapter defines the contract. Every Action API in your codebase looks like this. Once a developer knows the contract, they can read or write any Action without surprise.

## The trio: AC_, AR_, TS_

Every Action API consists of three programs:

| Program | Type | Purpose |
|---------|------|---------|
| `AC_<verb>` | CL | The wrapper. Receives parameters, monitors for messages, calls the RPG, returns. This is what callers invoke. |
| `AR_<verb>` | RPG | The implementation. The actual business logic — validation, database access, the work. |
| `TS_<verb>` | CL | The test harness. Hardcoded sample parameters, calls `AC_`, displays the result. Runnable from a 5250 command line. |

Examples:
- `AC_DELSUPL` / `AR_DELSUPL` / `TS_DELSUPL`
- `AC_ADDSUPL` / `AR_ADDSUPL` / `TS_ADDSUPL`
- `AC_UPDSUPL` / `AR_UPDSUPL` / `TS_UPDSUPL`

All three live in the K3S object library — historically `K3S_5OBJ` for our customers. (Substitute your own library naming if you're applying this pattern outside K3S.)

### Why three programs and not one?

It's reasonable to ask why we don't just write the RPG. The CL wrapper looks like ceremony. The test program looks like duplication. They aren't.

**The CL exists for three reasons:**

First, **MONMSG**. CL has clean, declarative message-monitoring syntax. RPG has it too (MONITOR / ON-ERROR), but at the boundary between RPG and the rest of the system — particularly from PHP through the IBM i Toolkit — having a CL layer that catches every possible message and translates it into a clean `ERRORS=Y` / `ERRMSG=...` envelope is far simpler than trying to handle it all in RPG. If the RPG hits an unhandled exception, the CL catches it, logs it, and the caller sees a normal error response instead of an XMLSERVICE failure or a stuck job in MSGW.

Second, **callable from CL**. Not everything that calls an Action API comes from PHP. Sometimes a scheduled job, a batch process, or an operator's own CL program needs to invoke `DELSUPL`. CL programs call CL programs naturally. They can call RPG too, but the parameter passing is cleaner CL-to-CL, and you don't have to worry about library list nuances or program type mismatches.

Third, **parameter normalization**. The CL is the right place to apply defaults, trim trailing blanks, validate that required parameters were actually supplied (vs. coming through as nulls), and handle the "calling with the wrong number of parameters" case. By the time control reaches the RPG, the parameter list is clean.

**The TS_ program exists because tests have to be easy to run.** A unit test that requires you to set up a PHP environment, fire a request through XMLSERVICE, and inspect the response is not a unit test anyone will run. A `TS_DELSUPL` you can invoke with `CALL K3S_5OBJ/TS_DELSUPL` from the command line is a unit test that gets run.

The TS_ also has a second function: it documents the Action by example. A new developer reading `TS_DELSUPL` sees, immediately, what parameters `AC_DELSUPL` expects, in what order, with what shape. The TS_ is the smallest possible working example of every Action.

## The standard parameter envelope

Every Action API receives the same first set of parameters in the same order. Memorize this list — it's the same in every program in the codebase.

| Position | Name | Type | Direction | Purpose |
|----------|------|------|-----------|---------|
| 1 | `K3SOBJ` | char(10) | input | The K3S object library (e.g., `K3S_5OBJ`). Used so the program can find dependent objects without hardcoding the library. |
| 2 | `COMP` | char(1) | input | Company indicator. Single-character key into the multi-tenant company file. |
| 3 | `COMPCOD` | char(3) | input | Company code. Three-character secondary identifier. |
| 4 | `USER` | char(10) | input | The IBM i user profile of the calling user. Used for audit logging and for any user-specific business logic. |
| 5 | `ERRORS` | char(1) | output | `Y` if an error occurred, `N` if the action completed successfully. |
| 6 | `ERRMSG` | char(100) | output | Human-readable error message if `ERRORS=Y`. Empty if `ERRORS=N`. |
| 7 | `ERRFIELD` | char(20) | output | The name of the input field that caused the error, if applicable. Empty otherwise. |

Then come the action-specific parameters, named according to the prefix conventions below.

{: .note }
> The first four parameters are *always* input. The next three are *always* output. The error envelope is the contract between the Action and its caller; every Action signals success or failure the same way.

### Why the K3SOBJ parameter

Hardcoding the object library inside RPG is a common mistake that makes the same code uninstallable at customer sites with different library naming. By passing `K3SOBJ` in, the same compiled object can run at any customer with any library name. The CL wrapper or PHP layer is responsible for knowing the right library and passing it through.

### Why USER as a parameter

The IBM i job already has a current user. Passing `USER` separately matters because of how the PHP toolkit calls programs: the job runs under a service profile (often the toolkit's own user), but the *business* user — the human who initiated the action through the web app — is whoever is logged into the application. We need that human's identity for audit logging and authorization, not the service profile's identity.

## Field-level prefix conventions

After the standard envelope, action-specific parameters follow three prefixes that signal their role:

| Prefix | Meaning | Used for |
|--------|---------|----------|
| `ID...` | **Identifier.** Identifies a record. Immutable in update operations. | `IDSUPL`, `IDLOCN`, `IDBUYR`, `IDSUPLSUB` |
| `SB...` | **Submit value.** A new value being submitted as input. May be updated. | `SBSUPLNAM`, `SBPHONE`, `SBADDRESS` |
| `RV...` | **Return value.** Output from the program back to the caller. | `RVSUPLID` (newly created supplier ID), `RVCOUNT` |

The conventions matter because they're load-bearing during update operations. In `AR_UPDSUPL`, the `ID*` parameters tell the program *which* supplier to update. The `SB*` parameters tell it *what to set*. The program will not use an `ID*` field as an updatable column; the `ID` prefix means "look up by this, don't change this." If you want to update the identifier itself (rare and usually wrong), that's a different operation and a different program.

The prefixes also make API documentation generation trivial. A page generator that reads the parameter list of `AR_UPDSUPL` knows automatically that `IDSUPL` is the supplier identifier (input only), `SBSUPLNAM` is the supplier name (input, updatable), and `RVUPDCNT` is the count of records updated (output). No additional metadata required.

## Length and type rules

All character parameters are fixed-length, padded with blanks. All numeric parameters are zoned or packed depending on existing K3S conventions for the underlying field. The Action *assumes* the caller has formatted parameters correctly — alphanumeric where alphanumeric is expected, numeric where numeric is expected, no longer than the declared size. The Action validates business rules; it does not validate that you passed a string into a string field.

This sounds risky and it isn't. The CL wrapper enforces the types via the `CRTCMD` definition (or via prototype declarations), and the PHP toolkit (`AddParameterChar`, `AddParameterPackDec`, etc.) enforces them on the calling side. Type mismatches surface as XMLSERVICE errors, not as garbage data inside the RPG. By the time the RPG receives the parameters, they're known to be the right shape.

{: .tip }
> If you find yourself writing data-type validation inside the RPG, that's a signal: the parameter type is probably wrong, or you're allowing the caller to pass strings where you should be requiring numerics. Fix it at the boundary.

## The error contract

`ERRORS` is a one-character flag: `Y` or `N`. `Y` means the operation failed. `N` means it succeeded. Treat anything else as a fatal protocol error — it should never happen, and if it does, the caller should treat the whole call as failed.

`ERRMSG` is a 100-character human-readable message. It's intended to be shown to the end user, so it should be in English (or whatever language your customers expect), not a system message ID. *"Supplier 12345 not found"*, not *"CPF5006"*.

`ERRFIELD` is the field name (using the parameter naming convention) of the field that caused the error. *"IDSUPL"* if the supplier ID was the problem; *"SBPHONE"* if the phone number didn't validate. The web frontend uses this to highlight the offending input field. If the error isn't tied to a specific field, leave it blank.

This three-field envelope is enough to handle every error case we've ever encountered. We have considered richer error structures (error codes, multi-error arrays, severity levels) and consistently concluded that the simplicity of `ERRORS` / `ERRMSG` / `ERRFIELD` outweighs the benefits. Most callers want to know one thing: did it work? If not, why not, and where? The envelope answers exactly those questions.

## MONMSG: catching the unexpected

Every CL wrapper monitors for messages. The pattern looks like this:

```cl
PGM PARM(&K3SOBJ &COMP &COMPCOD &USER &ERRORS &ERRMSG &ERRFIELD +
        &IDBUYR &IDLOCN &IDSUPL &IDSUPLSUB)

  DCL VAR(&K3SOBJ)    TYPE(*CHAR) LEN(10)
  DCL VAR(&COMP)      TYPE(*CHAR) LEN(1)
  DCL VAR(&COMPCOD)   TYPE(*CHAR) LEN(3)
  DCL VAR(&USER)      TYPE(*CHAR) LEN(10)
  DCL VAR(&ERRORS)    TYPE(*CHAR) LEN(1)
  DCL VAR(&ERRMSG)    TYPE(*CHAR) LEN(100)
  DCL VAR(&ERRFIELD)  TYPE(*CHAR) LEN(20)
  DCL VAR(&IDBUYR)    TYPE(*CHAR) LEN(5)
  DCL VAR(&IDLOCN)    TYPE(*CHAR) LEN(5)
  DCL VAR(&IDSUPL)    TYPE(*CHAR) LEN(10)
  DCL VAR(&IDSUPLSUB) TYPE(*CHAR) LEN(10)

  /* Default outputs */
  CHGVAR VAR(&ERRORS)   VALUE('N')
  CHGVAR VAR(&ERRMSG)   VALUE(' ')
  CHGVAR VAR(&ERRFIELD) VALUE(' ')

  /* Call the RPG implementation */
  CALL PGM(AR_DELSUPL) PARM(&K3SOBJ &COMP &COMPCOD &USER +
                             &ERRORS &ERRMSG &ERRFIELD +
                             &IDBUYR &IDLOCN &IDSUPL &IDSUPLSUB)
  MONMSG MSGID(CPF0000 MCH0000) EXEC(DO)
    CHGVAR VAR(&ERRORS)   VALUE('Y')
    CHGVAR VAR(&ERRMSG)   VALUE('Unhandled error in AR_DELSUPL')
    CHGVAR VAR(&ERRFIELD) VALUE(' ')
  ENDDO

ENDPGM
```

The `MONMSG MSGID(CPF0000 MCH0000)` catches every CPF (system) and MCH (machine check) message. If the RPG hits an unmonitored exception, the CL catches it instead of the program halting in MSGW. The caller sees `ERRORS=Y` with a generic message, the API log records the failure, and the support team gets an email with the parameters and the original error. *No program ever ends in MSGW state from an Action API.* That's the rule.

## Logging

Every Action API call is logged. The log table records:

| Field | Purpose |
|-------|---------|
| Program name | `AR_DELSUPL`, etc. |
| User | The calling user (from the `USER` parameter) |
| Start time | When the call began |
| End time | When the call returned |
| Duration (ms) | End minus start |
| Error indicator | `Y`/`N` from the response |
| Error message | The `ERRMSG` value if there was one |
| Parameter snapshot | A serialized record of inputs (for diagnostics — be careful about PII here) |

Logs are kept for a configurable retention window. The K3S default is 30 days. They serve three purposes: troubleshooting (when a call failed, what did it receive?), performance analysis (which Actions are slow, which are called most often?), and audit (who did what, when?).

The logging itself is implemented as another Action — `AR_LOGAPI` — called from inside `AC_<verb>` before and after the `AR_<verb>` call. Yes, this means the logging adds two extra program calls per request. It's worth it, and the cost is negligible (sub-millisecond on any current Power hardware).

{: .warning }
> Be careful what you put in the parameter snapshot. If an Action accepts a credit card number, an SSN, or any other sensitive data, redact it before it hits the log table. The same retention window that protects you operationally creates risk if it stores data that should never have been retained.

## Errors that escalate to support

When an Action returns `ERRORS=Y` *and* the error wasn't a normal validation failure (i.e., it was caught by MONMSG, not raised intentionally by the RPG), the system emails the parameter snapshot and the error to the K3S support inbox. This catches programmer mistakes — null pointers, file-not-found, division-by-zero — without the customer needing to file a ticket. By the time the customer notices, support already has the data needed to investigate.

The classification of "normal validation error" vs. "unexpected error" is a single flag in the log record. The RPG sets it explicitly when it raises an `ERRMSG` for a known reason. MONMSG-caught errors default to "unexpected" and trigger the email.

## Pulling it all together

The Action API Pattern, in one paragraph, is this: every business action is a single RPG program with a single purpose, wrapped in a CL that catches messages and normalizes the response, exercised by a test program with hardcoded parameters, identified by a verb prefix (`ADD`, `DEL`, `UPD`, `CHG`, `GET`, `LST`), receiving a standard envelope of identification, user, and error parameters in a fixed order, and logged on every invocation. Any caller — RPG, CL, PHP, MCP, curl from a developer's laptop — can invoke it the same way and get the same shape of response.

The chapters that follow apply this pattern to a real example: Supplier Maintenance, decomposed from a hypothetical green screen into a complete Action API set. By the end you'll have a reference template for every Action you'll ever build.

[Next: Decomposing Supplier Maintenance]({% link 03-decomposing-supplier-maintenance.md %}){: .btn .btn-primary }
