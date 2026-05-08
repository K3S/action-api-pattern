---
title: What You Don't Do
nav_order: 9
permalink: /discipline/
---

# What You Don't Do
{: .no_toc }

1. TOC
{:toc}

---

The Action API Pattern is a small set of rules. Most of the value comes from *not breaking them*. Every shortcut feels reasonable in the moment and corrupts the pattern for everyone who comes after. This chapter catalogs the temptations and explains why each one matters.

## Don't use mode flags

The first temptation. You have `AR_ADDSUPL` and `AR_UPDSUPL`, and you notice they share 80% of their code. The thought arises: *why not combine them into `AR_MNTSUPL` with a `MODE` parameter?* `MODE='A'` adds. `MODE='U'` updates. `MODE='D'` deletes. One program, three behaviors.

Don't do this. Here's why.

**It breaks the schema.** A multi-mode program has different required fields depending on mode. In add mode, `SBSUPL` is required (the new supplier code). In update mode, `IDSUPL` is required (the supplier to update). Writing the parameter list and the validation to handle "if MODE='A' then `SBSUPL` is required, else `IDSUPL` is required" turns a clean contract into a tangle of conditionals. The MCP tool schema can't even express it cleanly — JSON Schema's `oneOf` for mode-dependent required fields is technically possible but reads like a tax form.

**It breaks testing.** `TS_MNTSUPL` now needs to test three modes, each with its own edge cases. The single-purpose test program that you can run from a 5250 command line balloons into a test framework. The discipline of "one TS_ per Action" depends on each Action being one thing.

**It breaks logging and audit.** When something goes wrong with `AR_MNTSUPL`, the log entry says "AR_MNTSUPL failed" — but did the add fail? The update? The delete? You have to crack open the parameter snapshot to find out. With three programs, the program name in the log *is* the audit trail.

**It breaks code reviews.** A PR that touches `AR_MNTSUPL` could be changing add logic, update logic, delete logic, or all three. The reviewer has to read the entire program every time. Three small programs make every PR scoped.

**It breaks the mental model.** The whole point of the pattern is that an Action is a verb. A multi-mode program is a *menu*, which is what we just left behind. If you're tempted to consolidate, you're un-decomposing.

The 80% shared logic is real, and the answer is the service program (`SP_VALIDATE`, `SP_SUPPLIER_HELPERS`) we discussed in [Decomposing Supplier Maintenance]({% link 03-decomposing-supplier-maintenance.md %}). Pull the shared code into a procedure, bind to it from each Action. The Action is the boundary; the procedure is the implementation detail.

{: .caution }
> Mode flags are the single most common way the Action API Pattern decays in practice. Every IBM i shop has at least one program that started as a clean Action and became a Swiss Army knife with a `MODE` parameter. Once that pattern is established in the codebase, the next developer adds another mode rather than splitting the program, and the rot spreads.

## Don't put UI logic in `AR_`

The RPG should never know what kind of client called it. No special-casing for *"this is the green screen so format the error message in 5250 attribute bytes."* No *"this is PHP so return JSON-style escaped strings."* No *"this is MCP so include extra context."*

The contract is the contract. The RPG returns `ERRMSG` as a 100-character human-readable English string. Everything else is the caller's job.

If you find yourself writing `if (calling_via_web) ... else ...` in RPG, stop. The `calling_via_web` flag is somebody else's concern. The Action is supposed to be ignorant of who's calling it.

A subtler version of this rule: don't return data shaped for a particular renderer. If the React frontend wants supplier data formatted as a row in a table, the React frontend can format it. The RPG returns the supplier data in its natural shape. The PHP layer can transform it; the React component can transform it again. The RPG has one shape per record, period.

## Don't put screens in CL

The CL wrapper is plumbing. It marshals parameters, monitors messages, calls the RPG, returns. It does not display screens. It does not prompt for input. It does not call other Action APIs to gather data before calling the underlying RPG.

The reason is the same as the RPG rule: the CL has no idea who's calling it. A CL that displays a screen works only when called from a 5250 session. A CL that prompts for input fails when called from PHP because there's nobody to type.

If a workflow needs to chain multiple Actions ("first check supplier exists, then check open orders, then delete"), that orchestration belongs in the *caller* — the PHP handler, or the MCP server, or the React app. The Action APIs are atomic. The orchestration that strings them together is part of the application layer above.

The one exception to "no screens in CL" is `TS_<verb>` test programs, which are explicitly meant to be called from a 5250 command line and which display their result via `SNDPGMMSG`. That's not a UI; it's a debug print.

## Don't share output parameters across actions

When you have `AR_GETSUPL` returning `RVSUPLNAM`, `RVPHONE`, `RVADDR1`, etc., it's tempting to define a shared "supplier output struct" used by `AR_GETSUPL`, `AR_LSTSUPL`, `AR_ADDSUPL` (returning the row that was just added), and so on.

Resist. Each Action has its own output parameter list, even if many of them overlap. The reasons:

**Decoupling.** If you change `RVSUPLNAM` (say, expand it from 30 to 50 chars), you want to change *one* Action at a time, not all of them at once. Shared structures force coupled changes.

**Schema clarity.** When the MCP tool definition reads the parameter list of `AR_GETSUPL`, it should see exactly what GETSUPL returns — not a generic "supplier shape" that may include fields GETSUPL doesn't actually populate.

**Versioning.** When `AR_GETSUPLV2` adds new fields, the V1 contract is unchanged. With shared structures, V1 and V2 are entangled.

The repetition feels wasteful, and it isn't. Action APIs are public-facing contracts (in the sense that anything outside the program can depend on them). Public contracts deserve to be explicit.

## Don't skip the test program

Every Action needs a `TS_`. Even — especially — the trivial ones.

The pressure to skip is real: *AR_DELSUPL is five lines; obviously it works; I'll write the test next sprint.* But the test program isn't only a test. It's:

- The smallest working example of the Action, which the next developer reads to learn the Action.
- The fastest way to reproduce a bug ("what did `TS_DELSUPL` do when it broke?").
- The regression check when you change the underlying file or service program.
- The documentation of what valid parameters look like, in code that runs.

A `TS_` is fifteen minutes to write. Skipping it is the kind of cost-saving that costs you a full afternoon six months later when you need to debug the Action and have no smoke test to start from.

## Don't reach for "this Action is special"

Patterns are valuable because they're predictable. The moment one Action breaks the pattern — *this one's a query that also does an update because it's faster*, *this one returns its result through a global data area because the records are too big for parameters*, *this one skips the API log because it's called too often* — every other developer has to remember the exception.

If an Action genuinely doesn't fit, it's possible the Action shouldn't exist as designed. Maybe it's two actions: a query and a separate command. Maybe it should be a batch process, not an Action. Maybe the data structure needs rework so the result *does* fit through normal parameters.

We've made exactly two pattern exceptions in the K3S codebase in fifteen years, and both were heavily scrutinized. Most "this is special" cases turn out to be misframings on closer look.

{: .tip }
> A useful test: if you can't explain the exception to a new developer in one minute, the exception is too complicated. If you can, document it loudly — at the top of the program, in the API log, in the docs. Patterns survive on consistency, and exceptions survive on visibility.

## Don't let the green screen back in

The pattern works because the Action is decoupled from any particular UI. A common failure mode is gradual recoupling: the React frontend ships, the buyers complain that the new web form is missing the legacy "press F11 for alternate view" feature, somebody adds a `RVALTVIEW` parameter to `AR_GETSUPL` to support that feature, then somebody adds `SBALTVIEW` to control which view to return, and three years later `AR_GETSUPL` has thirty parameters most of which encode UI state.

Watch for it. Every parameter that gets added to an Action should answer the question *what business data does this represent*, not *what UI feature does this enable*. UI features belong in UIs. Actions are about business operations.

If a UI legitimately needs view-specific data, that's usually a separate Action: `AR_GETSUPLALT`, `AR_GETSUPLEXT`, or whatever the right verb is. Two Actions are easier to maintain than one Action with a flag.

## Don't optimize prematurely

The pattern is moderately verbose. The CL wrapper, the RPG, the test program, the service-program binding — there's plumbing. Some teams look at this and try to compress: *we'll skip the CL, call RPG directly from PHP, save a call.* Or: *we'll combine the open-log and close-log into one, save a call.* Or: *we'll inline the validation procedure, save the bind cost.*

These optimizations are real but tiny — sub-millisecond on any modern Power hardware — and they buy you fragility. The CL wrapper exists to catch unhandled messages; without it, an unmonitored exception lands the calling job in MSGW. The two-call log pattern exists to record duration; without it, you can't tell which Actions are slow. The service-program bind exists to share validation; without it, the same logic copy-pastes into ten Actions and drifts.

Pay the plumbing cost. The pattern's predictability pays it back many times over in debuggability and maintainability.

## Summary

Single purpose per program. No mode flags. No UI in RPG. No screens in CL. Distinct output parameters per Action. Always a TS_. Resist exceptions. Keep the green screen out. Don't compress the plumbing.

These rules are restrictive, and the restriction is the point. A developer reading any K3S Action API knows immediately what it does, what it expects, what it returns, and how to test it — because every Action follows the same shape. That predictability is worth more than any individual cleverness you might apply by breaking the pattern.

[Next: Migrating Existing Programs]({% link 09-migrating.md %}){: .btn .btn-primary }
