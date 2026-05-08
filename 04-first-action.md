---
title: Writing Your First Action
nav_order: 5
permalink: /first-action/
---

# Writing Your First Action
{: .no_toc }

1. TOC
{:toc}

---

This chapter walks through `DELSUPL` end to end. By the time you finish, you'll have a working `AR_DELSUPL`, `AC_DELSUPL`, and `TS_DELSUPL` you can compile and run. We start with delete because it's the smallest of the supplier actions: identification in, success or failure out, no field-level update logic.

## What `DELSUPL` does

`DELSUPL` deletes a supplier identified by a compound key (`IDBUYR`, `IDLOCN`, `IDSUPL`, `IDSUPLSUB`). Before deleting, it must verify:

1. The buyer + location combination exists.
2. A supplier with that code exists under that buyer + location.
3. The supplier has no open purchase orders. (You can't delete a supplier the company is mid-transaction with.)

If any check fails, the program returns `ERRORS=Y` with a clear `ERRMSG` and the relevant `ERRFIELD`. If everything passes, it deletes the row and returns `ERRORS=N`.

Audit: every successful delete is logged to a supplier-history file with the user, timestamp, and the key of the deleted record. The delete is a hard delete in this example; soft-delete patterns are a design choice we'll touch on at the end.

## The RPG: `AR_DELSUPL`

Free-format RPGLE. Modern style. Comments inline.

```rpgle
**FREE
//=================================================================
// Program     : AR_DELSUPL
// Pattern     : Action API
// Purpose     : Delete a supplier
// Inputs      : K3SOBJ, COMP, COMPCOD, USER (standard envelope)
//             : IDBUYR, IDLOCN, IDSUPL, IDSUPLSUB (compound key)
// Outputs     : ERRORS, ERRMSG, ERRFIELD (standard envelope)
// Service Pgm : Binds to SP_VALIDATE for buyer/location lookups
//=================================================================

ctl-opt nomain option(*nodebugio:*srcstmt) bnddir('K3S_5BIND');

//-----------------------------------------------------------------
// File declarations
//-----------------------------------------------------------------
dcl-f K_SUPPL  usage(*update:*delete) keyed;
dcl-f K_PORDER usage(*input)          keyed;

//-----------------------------------------------------------------
// Procedure prototype (the main entry)
//-----------------------------------------------------------------
dcl-proc AR_DELSUPL export;

  dcl-pi *n;
    K3SOBJ    char(10) const;
    COMP      char(1)  const;
    COMPCOD   char(3)  const;
    USER      char(10) const;
    ERRORS    char(1);
    ERRMSG    char(100);
    ERRFIELD  char(20);
    IDBUYR    char(5)  const;
    IDLOCN    char(5)  const;
    IDSUPL    char(10) const;
    IDSUPLSUB char(10) const;
  end-pi;

  //---------------------------------------------------------------
  // External service program prototypes (declared in copybook)
  //---------------------------------------------------------------
  dcl-pr ChkBuyrLocn ind;
    iBuyr char(5) const;
    iLocn char(5) const;
  end-pr;

  dcl-pr SuplHasOpenOrders ind;
    iBuyr     char(5)  const;
    iLocn     char(5)  const;
    iSupl     char(10) const;
    iSuplSub  char(10) const;
  end-pr;

  dcl-pr LogSuplHistory;
    iAction   char(3)  const;
    iUser     char(10) const;
    iBuyr     char(5)  const;
    iLocn     char(5)  const;
    iSupl     char(10) const;
    iSuplSub  char(10) const;
  end-pr;

  //---------------------------------------------------------------
  // Initialize outputs to "success"
  //---------------------------------------------------------------
  ERRORS   = 'N';
  ERRMSG   = *blanks;
  ERRFIELD = *blanks;

  //---------------------------------------------------------------
  // Validation 1: buyer + location must exist
  //---------------------------------------------------------------
  if not ChkBuyrLocn(IDBUYR : IDLOCN);
    ERRORS   = 'Y';
    ERRMSG   = 'Invalid buyer/location combination';
    ERRFIELD = 'IDBUYR';
    return;
  endif;

  //---------------------------------------------------------------
  // Validation 2: supplier must exist under this buyer/location
  //---------------------------------------------------------------
  chain (IDBUYR : IDLOCN : IDSUPL : IDSUPLSUB) K_SUPPL;
  if not %found(K_SUPPL);
    ERRORS   = 'Y';
    ERRMSG   = 'Supplier ' + %trim(IDSUPL) + ' not found';
    ERRFIELD = 'IDSUPL';
    return;
  endif;

  //---------------------------------------------------------------
  // Validation 3: cannot delete a supplier with open POs
  //---------------------------------------------------------------
  if SuplHasOpenOrders(IDBUYR : IDLOCN : IDSUPL : IDSUPLSUB);
    ERRORS   = 'Y';
    ERRMSG   = 'Cannot delete: supplier has open purchase orders';
    ERRFIELD = 'IDSUPL';
    return;
  endif;

  //---------------------------------------------------------------
  // Perform the delete
  //---------------------------------------------------------------
  delete K_SUPPL;
  if %error;
    ERRORS   = 'Y';
    ERRMSG   = 'Delete failed: database error';
    ERRFIELD = *blanks;
    return;
  endif;

  //---------------------------------------------------------------
  // Audit log
  //---------------------------------------------------------------
  LogSuplHistory('DEL' : USER : IDBUYR : IDLOCN : IDSUPL : IDSUPLSUB);

  return;

end-proc;
```

A few things to notice:

**`nomain` and a single exported procedure.** The Action is implemented as a procedure, not a cycle-driven program. This is the modern free-format pattern. The procedure is exported so the CL wrapper can `CALL` it — but really, in production, the CL calls `AR_DELSUPL` as a `*PGM`, and the build process handles the bind. For pedagogical clarity here we're showing it as a single procedure-based program; you can also use `MAIN(AR_DELSUPL)` if you prefer.

**Const inputs.** The input parameters are declared `const`. This signals to the compiler — and to the next developer reading the code — that `IDBUYR` is not going to be modified inside the program. The output parameters (`ERRORS`, `ERRMSG`, `ERRFIELD`) deliberately are *not* const, because we set them.

**Service program calls for shared validation.** `ChkBuyrLocn` and `SuplHasOpenOrders` aren't open-coded inside the RPG. They're prototype declarations that resolve to procedures in `SP_VALIDATE`. Other Actions (`AR_ADDSUPL`, `AR_UPDSUPL`) reuse the same procedures. Single source of truth for the validation rules.

**Early returns on validation failure.** No nested IFs, no monstrous boolean expression. Each check is independent: if it fails, set the error envelope and return. The happy path falls through to the bottom. This keeps each branch readable and the cyclomatic complexity low.

**`ERRFIELD` points at the offending parameter.** Web frontends use this to highlight the input field. We don't say `'Buyer code'` — we say `'IDBUYR'`, the name of the field as the API exposes it, so the frontend's mapping is mechanical.

{: .tip }
> Notice the absence of any screen handling, any DSPLY, any subfile manipulation. Notice the absence of any `CALL` to other RPG programs that might have side effects. The procedure does one thing — delete a supplier — and reports its result through the envelope. That's the entire definition of "headless."

## The CL: `AC_DELSUPL`

```cl
/*-----------------------------------------------------------------*/
/* Program     : AC_DELSUPL                                        */
/* Pattern     : Action API                                        */
/* Purpose     : CL wrapper for AR_DELSUPL                          */
/*-----------------------------------------------------------------*/

PGM PARM(&K3SOBJ &COMP &COMPCOD &USER +
        &ERRORS &ERRMSG &ERRFIELD +
        &IDBUYR &IDLOCN &IDSUPL &IDSUPLSUB)

  /* Standard envelope */
  DCL VAR(&K3SOBJ)    TYPE(*CHAR) LEN(10)
  DCL VAR(&COMP)      TYPE(*CHAR) LEN(1)
  DCL VAR(&COMPCOD)   TYPE(*CHAR) LEN(3)
  DCL VAR(&USER)      TYPE(*CHAR) LEN(10)
  DCL VAR(&ERRORS)    TYPE(*CHAR) LEN(1)
  DCL VAR(&ERRMSG)    TYPE(*CHAR) LEN(100)
  DCL VAR(&ERRFIELD)  TYPE(*CHAR) LEN(20)

  /* Action-specific parameters */
  DCL VAR(&IDBUYR)    TYPE(*CHAR) LEN(5)
  DCL VAR(&IDLOCN)    TYPE(*CHAR) LEN(5)
  DCL VAR(&IDSUPL)    TYPE(*CHAR) LEN(10)
  DCL VAR(&IDSUPLSUB) TYPE(*CHAR) LEN(10)

  /* Locals for logging */
  DCL VAR(&LOGID)     TYPE(*CHAR) LEN(20)

  /* Default outputs to "success" */
  CHGVAR VAR(&ERRORS)   VALUE('N')
  CHGVAR VAR(&ERRMSG)   VALUE(' ')
  CHGVAR VAR(&ERRFIELD) VALUE(' ')

  /* Establish the K3S object library at the top of the list      */
  /* so AR_DELSUPL and the service program resolve correctly.     */
  ADDLIBLE LIB(&K3SOBJ) POSITION(*FIRST)
  MONMSG MSGID(CPF2103)

  /* Adopt the calling user's authority for the duration of the   */
  /* call. This relies on a USER-profile-style model where USER    */
  /* maps to a real *USRPRF.                                       */
  /* (Implementation depends on your authentication strategy.)    */

  /* Open the API log entry */
  CALL PGM(AR_LOGOPN) PARM('AR_DELSUPL' &USER &LOGID)
  MONMSG MSGID(CPF0000 MCH0000)

  /* Call the RPG implementation */
  CALL PGM(AR_DELSUPL) PARM(&K3SOBJ &COMP &COMPCOD &USER +
                             &ERRORS &ERRMSG &ERRFIELD +
                             &IDBUYR &IDLOCN &IDSUPL &IDSUPLSUB)
  MONMSG MSGID(CPF0000 MCH0000) EXEC(DO)
    CHGVAR VAR(&ERRORS)   VALUE('Y')
    CHGVAR VAR(&ERRMSG)   VALUE('Unhandled error in AR_DELSUPL')
    CHGVAR VAR(&ERRFIELD) VALUE(' ')
  ENDDO

  /* Close the API log entry */
  CALL PGM(AR_LOGCLS) PARM(&LOGID &ERRORS &ERRMSG)
  MONMSG MSGID(CPF0000 MCH0000)

  /* Remove the library list addition */
  RMVLIBLE LIB(&K3SOBJ)
  MONMSG MSGID(CPF2104)

ENDPGM
```

The CL is plumbing. It does six things:

1. Declares the standard envelope plus action-specific parameters.
2. Defaults the output envelope to "success" so a caller that ignores the response still sees a sane state.
3. Adds the K3S library to the library list (with `MONMSG` for the case where it's already there).
4. Opens an API log record (`AR_LOGOPN` returns a log ID we use later to close it).
5. Calls `AR_DELSUPL`, with `MONMSG` to catch any unhandled exception and translate it into the error envelope.
6. Closes the log record with the final outcome and removes the library from the list.

Steps 3 and 6 — the library list manipulation — are optional depending on how your environment is configured. Some shops set the library list at job startup and never touch it. Others do it per-call. Pick whichever your operations team is comfortable with.

{: .note }
> The `AR_LOGOPN` / `AR_LOGCLS` pair is itself an Action API. We're not going to write it out here, but its existence demonstrates a useful property: the pattern composes. Actions can call other Actions. The logging is just another Action that happens to be invoked by the wrapper.

## The test program: `TS_DELSUPL`

```cl
/*-----------------------------------------------------------------*/
/* Program     : TS_DELSUPL                                        */
/* Pattern     : Action API Test Harness                            */
/* Purpose     : Test AC_DELSUPL with hardcoded sample parameters  */
/* Run from    : 5250 command line — CALL K3S_5OBJ/TS_DELSUPL      */
/*-----------------------------------------------------------------*/

PGM

  /* Test parameters — the values you'd hand to a real call */
  DCL VAR(&K3SOBJ)    TYPE(*CHAR) LEN(10) VALUE('K3S_5OBJ')
  DCL VAR(&COMP)      TYPE(*CHAR) LEN(1)  VALUE('1')
  DCL VAR(&COMPCOD)   TYPE(*CHAR) LEN(3)  VALUE('K3S')
  DCL VAR(&USER)      TYPE(*CHAR) LEN(10) VALUE('TESTUSER')

  DCL VAR(&IDBUYR)    TYPE(*CHAR) LEN(5)  VALUE('00001')
  DCL VAR(&IDLOCN)    TYPE(*CHAR) LEN(5)  VALUE('00001')
  DCL VAR(&IDSUPL)    TYPE(*CHAR) LEN(10) VALUE('TESTSUP')
  DCL VAR(&IDSUPLSUB) TYPE(*CHAR) LEN(10) VALUE(' ')

  /* Output envelope */
  DCL VAR(&ERRORS)    TYPE(*CHAR) LEN(1)
  DCL VAR(&ERRMSG)    TYPE(*CHAR) LEN(100)
  DCL VAR(&ERRFIELD)  TYPE(*CHAR) LEN(20)

  /* For DSPLY */
  DCL VAR(&MSGOUT)    TYPE(*CHAR) LEN(132)

  /* First, ensure a known test supplier exists.                  */
  /* TS_ADDSUPL creates 'TESTSUP' if it doesn't already exist.    */
  CALL PGM(TS_ADDSUPL)
  MONMSG MSGID(CPF0000)

  /* Now exercise the Action under test */
  CALL PGM(AC_DELSUPL) PARM(&K3SOBJ &COMP &COMPCOD &USER +
                             &ERRORS &ERRMSG &ERRFIELD +
                             &IDBUYR &IDLOCN &IDSUPL &IDSUPLSUB)

  /* Display the outcome */
  CHGVAR VAR(&MSGOUT) VALUE('ERRORS=' *CAT &ERRORS *TCAT +
                              ' ERRMSG=' *CAT &ERRMSG *TCAT +
                              ' ERRFIELD=' *CAT &ERRFIELD)
  SNDPGMMSG MSG(&MSGOUT)

ENDPGM
```

The test program calls `TS_ADDSUPL` first to seed a known test supplier (`TESTSUP`), then calls `AC_DELSUPL` to delete it, then displays the response envelope via `SNDPGMMSG`. Run it from a 5250 command line:

```
CALL K3S_5OBJ/TS_DELSUPL
```

You should see the message line at the bottom of the screen showing `ERRORS=N`, an empty `ERRMSG`, and an empty `ERRFIELD`.

If you run it twice in a row without `TS_ADDSUPL` re-seeding, the second run will fail with `ERRORS=Y`, `ERRMSG=Supplier TESTSUP not found`, `ERRFIELD=IDSUPL`. That's the validation working — the program correctly refuses to delete a supplier that doesn't exist.

{: .tip }
> The TS_ for a destructive operation should always seed its own test data. A `TS_DELSUPL` that requires manual setup is a TS_ that nobody will run after the first time. Pair every destructive test with a `TS_ADDSUPL` (or equivalent) that establishes the precondition.

## Compiling

Three commands, in order:

```cl
/* Compile the RPG implementation */
CRTBNDRPG PGM(K3S_5OBJ/AR_DELSUPL) +
          SRCFILE(K3S_5SRC/QRPGLESRC) +
          SRCMBR(AR_DELSUPL) +
          DBGVIEW(*SOURCE) +
          BNDDIR(K3S_5BIND)

/* Compile the CL wrapper */
CRTBNDCL  PGM(K3S_5OBJ/AC_DELSUPL) +
          SRCFILE(K3S_5SRC/QCLSRC) +
          SRCMBR(AC_DELSUPL)

/* Compile the test program */
CRTBNDCL  PGM(K3S_5OBJ/TS_DELSUPL) +
          SRCFILE(K3S_5SRC/QCLSRC) +
          SRCMBR(TS_DELSUPL)
```

If you're using VS Code with the Code for IBM i extension, those compile commands fire from the *Object Browser* with right-click → *Compile* — you don't have to type them. (See [the K3S RPG Tutorial](https://rpgtutorial.k3s.com) for the VS Code setup walkthrough.)

`DBGVIEW(*SOURCE)` is worth keeping on for any Action API in development. It lets you debug the running RPG with `STRDBG PGM(AR_DELSUPL)` or via VS Code's debugger and step through the source. Drop to `*STMT` only when you're ready to ship.

## What you've built

You now have a working Action API. `TS_DELSUPL` exercises it end-to-end on the IBM i with no external dependency. Anything that can call a CL program with parameters — another CL program, an RPG program, the PHP toolkit, a REST gateway — can call `AC_DELSUPL` and get the same response shape.

In the next chapter we'll generalize from this one Action to the broader command-vs-query distinction: how `DEL`, `ADD`, and `UPD` are *commands* that share one shape, and how `GET` and `LST` are *queries* that need a slightly different shape — particularly for returning result sets.

[Next: Commands and Queries]({% link 05-commands-queries.md %}){: .btn .btn-primary }
