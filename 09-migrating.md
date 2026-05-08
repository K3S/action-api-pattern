---
title: Migrating Existing Programs
nav_order: 10
permalink: /migrating/
---

# Migrating Existing Programs
{: .no_toc }

1. TOC
{:toc}

---

This chapter is for the realistic situation: you have an existing IBM i codebase with hundreds of green-screen programs, you can't pause development for a multi-year rewrite, and you need to move toward Action APIs without breaking anything that already works. The pattern is fifty years old in software architecture; the IBM i specifics are what we'll focus on.

## The strangler fig pattern

Martin Fowler named it after a kind of tree that grows around a host tree, gradually overtaking it until the host is gone. Applied to software: you don't replace the legacy system. You build the new system *around* it, redirecting one piece of functionality at a time, until the legacy system is no longer reachable and can be retired.

For Action APIs on the IBM i, the strangler fig works like this:

1. Pick one action inside an existing green-screen program.
2. Implement it as a proper Action API (`AR_/AC_/TS_`).
3. Verify the Action API produces identical results to the green-screen path.
4. Redirect new clients (the React frontend, the mobile app, the MCP server) to the Action API.
5. Leave the green screen alone, still calling its internal subroutines.
6. Pick the next action. Repeat.
7. When all actions are migrated, decide what to do with the green screen.

At step 7 you have three options: retire the green screen entirely; keep it as a power-user tool but rewrite its internals to call the Action APIs (the green screen becomes a *client* of the Action layer, not a competitor); or leave it as-is for as long as the legacy users want it. All three are legitimate. K3S has examples of each.

## The five-step migration of one action

Let's walk through migrating *Delete Supplier* out of the legacy `SPMAINT`.

### Step 1: Read the existing code

Before you write a line of new code, read the existing logic. Inside `SPMAINT`, find the subroutine that handles the delete path. Read it carefully. Note:

- What validation does it do?
- What database operations does it perform?
- What side effects does it have (audit log, history file, related-record cleanup)?
- What error messages does it raise?

You'll often find that the legacy code does things you wouldn't have thought of. Maybe it cascades a delete to a supplier-history file. Maybe it sets a deactivation flag instead of actually deleting. Maybe it checks a permissions table that nobody documented. *All of that is a feature*. The migration is not a chance to redesign — it's a chance to extract the behavior into a cleaner shape.

### Step 2: Write `AR_DELSUPL`

Reproduce the legacy behavior, exactly, in the new program. Same validation, same side effects, same error messages. The Action API replaces the subroutine; it does not improve it. (Improvements are a separate project, after the migration is verified.)

The thing to be careful about: the legacy subroutine raised errors by writing to the message subfile and returning to the screen. Your Action returns errors through `ERRMSG` and `ERRFIELD`. The translation has to preserve the *semantics* — every legacy error message becomes an `ERRMSG` value, every error pointing at a screen field becomes an `ERRFIELD` value pointing at the corresponding parameter.

### Step 3: Verify equivalence

This is the step most teams skip and regret. Before you redirect any client to the new path, prove that the new path produces the same outputs as the old path.

The cleanest way: take a snapshot of test data, run the legacy path on a copy, run the new path on another copy, compare the resulting database state. They should match exactly. Same rows updated, same rows deleted, same audit records written, same edge cases handled.

Where they differ, investigate. *Most* differences are cases where you missed something in the legacy logic. *Some* differences are cases where the legacy logic had a bug — which means you have a decision to make: reproduce the bug for compatibility, or fix it in the new path and document the change?

K3S typically reproduces the bug for the migration phase, then fixes it in a follow-up release with explicit release notes. The migration itself is a strict equivalence; improvements come later under their own change control.

### Step 4: Redirect new clients

Now the new path goes live for new clients. The web frontend's "delete supplier" button calls the Mezzio handler from [Calling Actions from PHP]({% link 06-calling-from-php.md %}), which calls `AC_DELSUPL`, which calls `AR_DELSUPL`. The legacy `SPMAINT` is untouched; the green-screen users still get the old subroutine.

Watch the API log for the first week or two. Compare error rates between the new path and the green-screen path (you can pull green-screen errors from job logs if you have to). They should be similar. If the new path is generating more errors, you missed something — a validation case, a permission check, a side effect.

### Step 5: Decide what to do with the green screen

Once the new path is stable and the new clients are routing through it, you have options.

**Option A: Leave it.** The legacy users keep their familiar tool. The new clients use the new path. The two coexist. This is fine and often the right answer for niche legacy tools.

**Option B: Refactor the green screen to call the Action.** The legacy `SPMAINT` keeps its DDS, its subfile, its F-keys — but its internal "delete supplier" subroutine is replaced by a `CALL AC_DELSUPL`. Now both paths run identical code; bug fixes apply uniformly; the green screen is just another Action API client.

This is our preferred outcome where feasible. It eliminates the duplication permanently. The risk is that you have to touch `SPMAINT` to make the change, which means recompiling, regression-testing, and dealing with whatever undocumented behavior the program has accumulated. For high-risk programs, that's a project unto itself.

**Option C: Retire it.** Once the legacy users adopt the new web frontend, the green screen has no more users. Pull it out of the menus, archive the source, document its retirement.

The K3S approach is usually B for moderate-complexity programs and A for either trivial ones (not worth the touch) or extremely complex ones (the recompile risk outweighs the duplication cost).

## Sequencing the migration of an entire program

Inside `SPMAINT`, you have six action subroutines (`SR_ADDSUP`, `SR_UPDSUP`, `SR_DELSUP`, `SR_DSPSUP`, `SR_CPYSUP`, `SR_LDSFL`). What order do you migrate them in?

The pragmatic answer:

1. **Start with a low-risk read-only Action.** `GETSUPL` (corresponding to `SR_DSPSUP`) reads data; it can't break anything. Build it, verify equivalence, redirect the new clients. The team gets used to the pattern with low stakes.
2. **Move to a low-risk delete or update.** `DELSUPL` for a supplier with cascading constraints is reasonably contained. `UPDSUPL` if the file has good update protection. Pick whichever is least scary.
3. **Then the harder commands.** `ADDSUPL` (which has to handle uniqueness and defaults) and `CPYSUPL` (which has nontrivial copy logic).
4. **Save `LSTSUPL` for last** if the legacy subfile loader has weird filtering. List operations are often the messiest part of legacy green-screen programs because they grew organically with new filter requirements over the years.

The whole migration of one program might take a sprint or two of focused effort, plus a few weeks of soak time per Action. A six-Action program is realistically a quarter of work, end to end, to retire. With multiple developers in parallel, you can move faster.

## Backward compatibility tricks

Sometimes you can't write a clean Action because the legacy logic depends on things you're not ready to clean up. Two patterns help.

### The CL adapter

Suppose `SPMAINT`'s delete subroutine relies on a global data area being set up by the program's initialization. You can't easily replicate that initialization in a clean Action. Instead, write `AC_DELSUPL` as an *adapter* that calls into a still-existing subroutine inside `SPMAINT` (or a stripped-down version of it). The new clients see a clean Action API; the implementation under the hood is still the legacy code.

This is a transitional pattern, not a permanent one. The point is to give external callers the right interface *now*, while you do the deeper refactoring later. The adapter has to enforce the contract — error envelope, no screens, no globals leaking out — but it can hand the work to whatever legacy plumbing is still there.

### The shadow Action

For commands where you want to verify equivalence in production (not just in test), run *both* paths and compare. The new client calls `AC_DELSUPL`. `AC_DELSUPL` internally calls both the new `AR_DELSUPL` and the legacy subroutine, against a transactional copy of the data, and compares the results. If they match, commit. If they don't, log loudly and use whichever path is canonically correct (usually the legacy one during early migration).

This is heavyweight. You wouldn't do it for every Action. You might do it for a critical one — a financial calculation, an order-creation flow — where production data is the only realistic test. K3S has used this twice in fifteen years. It worked both times, caught real divergences, and saved a customer outage.

## The customer dimension

When the K3S codebase is shared across many customer installs, migration becomes a multi-axis problem. You're migrating an Action *and* deploying it to customers *and* managing the per-customer rollout schedule.

The patterns we use:

**Per-customer feature flags.** A customer config field decides whether the new path or the legacy path is active for that customer. Roll out to one customer first, verify, then expand. The flag lets you roll back without redeploying.

**Per-Action flags, not per-program flags.** A flag for "delete via new Action API" is fine. A flag for "use new SPMAINT" is too coarse — what if the new ADDSUPL is good but the new DELSUPL has a bug? Per-Action gives you precise rollback.

**Customer-driven prioritization.** Customers asking for the web frontend get migrated faster. Customers happy with the green screen wait. There's no need to force a migration on someone who doesn't need it.

**Parallel run windows.** During the rollout to a customer, both paths are available — the green screen still works for the buyers who haven't switched, and the web app works for those who have. The two paths use the same Action APIs (option B above), so they don't drift.

## When to abandon the migration

Sometimes a legacy program is so entangled, so badly understood, or so business-critical that decomposing it Action-by-Action isn't viable. In those cases, the right answer is to *wrap* rather than *decompose*: build a single Action API that calls the legacy program through `OVRDBF` and parameter passing, accept the rough edges, and move on.

This is rare and shouldn't be the default. But it's a valid escape hatch for the worst cases. The wrapper at least lets the new clients participate in the modern stack; the program itself can be untangled later, or never. We have one of these in the K3S codebase. It's been wrapped for eleven years. It's still wrapped. The world has not ended.

## Summary

Migrate one Action at a time. Reproduce existing behavior exactly before improving anything. Verify equivalence with real data. Redirect new clients first; refactor the legacy program later. Use per-customer, per-Action feature flags. Don't be afraid to leave the green screen alone if the cost of touching it outweighs the benefit. The strangler fig grows slowly, and that's the point.

[Next: Reference]({% link 10-reference.md %}){: .btn .btn-primary }
