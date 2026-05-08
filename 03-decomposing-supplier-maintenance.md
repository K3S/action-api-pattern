---
title: Decomposing Supplier Maintenance
nav_order: 4
permalink: /decomposing-supplier-maintenance/
---

# Decomposing Supplier Maintenance
{: .no_toc }

1. TOC
{:toc}

---

## The starting point

Imagine — or recall, if you have one of these in your codebase — a green-screen program called `SPMAINT`. Supplier Maintenance. A single RPG program with a DDS display file, a couple of subfile screens, and several thousand lines of code. It does everything supplier-related: it lets a buyer add a new supplier, change an existing one, delete one, copy one to a new code, view supplier history, and search through the supplier file.

A typical interaction looks like this:

1. User signs on, navigates the menu, selects "Supplier Maintenance"
2. `SPMAINT` displays a search/filter screen — buyer code, location, partial supplier name
3. User enters criteria, presses Enter
4. `SPMAINT` displays a subfile of matching suppliers
5. User puts a 2 (Change), 4 (Delete), 5 (Display), or 7 (Copy) next to a supplier
6. `SPMAINT` either branches to a detail screen, prompts for confirmation, or copies the record
7. User completes the action, presses Enter
8. `SPMAINT` updates the database, displays a success message, returns to the subfile

That program has been running for twenty years. It works. The buyers know it cold. They don't want it to go away.

It also has every problem we discussed in [Why Headless]({% link 01-why-headless.md %}). The validation logic is glued to the display file. The database access is interleaved with screen formatting. The business rules — *a supplier can't be deleted if it has open purchase orders, a supplier code must be unique within the company, the buyer code must be valid* — live inside subroutines that are only callable from inside this one program.

We're not going to throw it away. We're going to decompose it.

## Step 1: Inventory the actions

The first move is to list every distinct *action* a user can perform inside `SPMAINT`. Don't think about screens. Don't think about subroutines. Think about the actions, the verbs, the things that change or read state.

For Supplier Maintenance, the inventory looks like this:

| User does | Action verb | What changes |
|-----------|-------------|--------------|
| Adds a new supplier (F6 → fill in the form → Enter) | **ADD** | A new row in the supplier file |
| Changes an existing supplier (option 2 → edit fields → Enter) | **UPD** | One row updated |
| Deletes a supplier (option 4 → confirm) | **DEL** | One row deleted |
| Views supplier detail (option 5) | **GET** | Nothing changes; data is read |
| Copies a supplier to a new code (option 7 → enter new code) | **CPY** | A new row created from an existing one |
| Searches/filters suppliers (the initial screen) | **LST** | Nothing changes; a list is returned |

Six actions. Six verbs. Each one becomes an Action API:

- `AR_ADDSUPL` / `AC_ADDSUPL` / `TS_ADDSUPL`
- `AR_UPDSUPL` / `AC_UPDSUPL` / `TS_UPDSUPL`
- `AR_DELSUPL` / `AC_DELSUPL` / `TS_DELSUPL`
- `AR_GETSUPL` / `AC_GETSUPL` / `TS_GETSUPL`
- `AR_CPYSUPL` / `AC_CPYSUPL` / `TS_CPYSUPL`
- `AR_LSTSUPL` / `AC_LSTSUPL` / `TS_LSTSUPL`

That's the API surface. Eighteen objects total (six trios). Some teams find that overwhelming at first; six pieces of business logic feels like one program, not six. The unit of decomposition matters here. *Each action is its own program.* No exceptions, no shortcuts. We'll come back to why in [What You Don't Do]({% link 08-discipline.md %}).

{: .tip }
> If you can't articulate the verb in three letters or fewer (ADD, UPD, DEL, GET, LST, CPY, CHG), you probably haven't decomposed enough. Multi-word verbs are usually disguised compound actions.

## Step 2: Identify the parameters for each action

For each action, list the parameters it needs. Use the prefix conventions from [The Action API Pattern]({% link 02-action-api-pattern.md %}): `ID*` for identifiers, `SB*` for inputs, `RV*` for outputs.

### `ADDSUPL` — Add a supplier

Adds a new row to the supplier file under a given buyer/location.

| Parameter | Direction | Notes |
|-----------|-----------|-------|
| `IDBUYR` | input | Buyer code (the supplier belongs to a buyer) |
| `IDLOCN` | input | Location code (the supplier serves a specific location) |
| `SBSUPL` | input | The new supplier code being created |
| `SBSUPLSUB` | input | Sub-supplier qualifier (often blank) |
| `SBSUPLNAM` | input | Supplier name |
| `SBPHONE` | input | Phone number |
| `SBADDR1`, `SBADDR2` | input | Address lines |
| `SBCITY`, `SBSTATE`, `SBZIP` | input | City, state, postal code |
| `SBLEADTIME` | input | Lead time in days |
| `SBORDCYCLE` | input | Order cycle in days |
| `RVSUPLID` | output | Internal unique ID assigned to the new supplier (if your file design has one) |

### `UPDSUPL` — Update a supplier

Updates an existing supplier identified by buyer + location + supplier code.

| Parameter | Direction | Notes |
|-----------|-----------|-------|
| `IDBUYR`, `IDLOCN`, `IDSUPL`, `IDSUPLSUB` | input | The compound identifier — these find the row |
| `SBSUPLNAM`, `SBPHONE`, `SBADDR1`, ... | input | All updatable fields. Same as ADD minus the identifier fields. |

Note the symmetry: an update is a create minus the act of creation. Same fields, just `ID` instead of `SB` for the identifying parts.

### `DELSUPL` — Delete a supplier

Identified by the same compound key. No `SB` fields — delete is pure identification.

| Parameter | Direction | Notes |
|-----------|-----------|-------|
| `IDBUYR`, `IDLOCN`, `IDSUPL`, `IDSUPLSUB` | input | Compound key |

### `GETSUPL` — Get a supplier

Same key, but returns the row data through `RV*` output parameters.

| Parameter | Direction | Notes |
|-----------|-----------|-------|
| `IDBUYR`, `IDLOCN`, `IDSUPL`, `IDSUPLSUB` | input | Compound key |
| `RVSUPLNAM`, `RVPHONE`, `RVADDR1`, ... | output | The current values |

### `CPYSUPL` — Copy a supplier

Source identifier on the input side, destination identifier on the input side, output is the new record.

| Parameter | Direction | Notes |
|-----------|-----------|-------|
| `IDBUYR`, `IDLOCN`, `IDSUPL`, `IDSUPLSUB` | input | Source (the one being copied from) |
| `SBNEWSUPL`, `SBNEWSUPLSB` | input | Destination supplier code |
| `RVSUPLID` | output | New supplier's internal ID |

### `LSTSUPL` — List suppliers

Filters and returns a result set. This one doesn't fit cleanly in scalar parameters; we'll address result-set returns in [Commands and Queries]({% link 05-commands-queries.md %}).

| Parameter | Direction | Notes |
|-----------|-----------|-------|
| `IDBUYR`, `IDLOCN` | input | Optional filters |
| `SBNAMEFLT` | input | Partial name filter |
| `SBLIMIT`, `SBOFFSET` | input | Pagination |
| Result set | output | Via user space, work file, or JSON-emit pattern |

## Step 3: Map the SPMAINT subroutines to the new programs

Now go back into the existing `SPMAINT` source. Identify the subroutines (or procedures, in modern free-format) that correspond to each action.

A typical legacy `SPMAINT` might contain subroutines like:

- `SR_ADDSUP` — handles the add path
- `SR_UPDSUP` — handles the update path
- `SR_DELSUP` — handles the delete path
- `SR_DSPSUP` — handles the display path (this becomes `GETSUPL`)
- `SR_CPYSUP` — handles the copy path
- `SR_LDSFL` — loads the subfile (this becomes `LSTSUPL`)

Plus utility subroutines that all of those share:

- `SR_VALSUP` — validates supplier fields
- `SR_CHKBUYR` — validates that the buyer code exists
- `SR_CHKDUP` — checks for duplicate supplier codes
- `SR_LOGCHG` — writes an audit log entry

The action-specific subroutines move into the corresponding `AR_` programs, basically as-is. The utility subroutines are interesting: they're shared across multiple Actions. We have two options:

**Option A: Copy the utility logic into each Action.** Some duplication, but each Action is self-contained. Easier to reason about. Easier to deploy (no shared dependency).

**Option B: Extract the utilities into a service program (`*SRVPGM`) that each Action binds to.** No duplication. Single source of truth. But now you have a binding-time dependency, and changes to the service program ripple to every Action that uses it.

We recommend **Option B** for genuine business utilities (validation, key generation, cross-file lookups) and **Option A** for trivial logic (formatting a name, padding a number). The split mirrors what we'd do in any other language: shared libraries for shared logic, inline code for one-off transformations.

The K3S codebase uses a `K3S_5LIB` service-program library where these utilities live: `SP_VALSUP`, `SP_CHKBUYR`, `SP_CHKDUP`, etc. Each `AR_<verb>` binds to it via the `BNDDIR` directive. Calls into the service program are sub-microsecond — there's no performance argument against this approach.

## Step 4: Discard what doesn't survive

After decomposition, certain parts of `SPMAINT` no longer have a home. They were tied to the screen, and the screen is going away.

**The display file (DDS).** Gone. The new web frontend has its own UI. The old 5250 emulator users will, for a transition period, still call `SPMAINT`, but eventually that program retires.

**The cursor and keyboard handling.** Gone. F-keys, screen positioning, ROLLUP/ROLLDOWN — all of it lived to drive the 5250 protocol. None of it has a home in headless RPG.

**The screen-level error display.** Restructured. The errors used to be displayed at the bottom of the screen, in a message subfile. Now they come back through the `ERRMSG` and `ERRFIELD` envelope. The web frontend renders them however it likes — inline next to the offending input, in a toast notification, in a modal, whatever the UX calls for. The RPG's job is to *report* the error, not to *display* it.

**The control flow.** Gone. The old program's control flow — "show search, read filter, load subfile, read selection, branch to detail, read changes, validate, update, return to subfile" — was a state machine driven by the screen. The new world has no central state machine. Each Action API runs once, returns once. The state machine, if there is one, lives on the client side: the web app decides what screen to show next based on what just happened.

What remains, after all that, is the *business logic*. The validation, the database access, the rules. That's what you're keeping. Everything else was UI plumbing in disguise.

## Step 5: Plan the migration

You don't decompose `SPMAINT` in one weekend. The realistic path is incremental:

1. **Pick one action.** `DELSUPL` is a good first one — it's small, focused, and the validation rules are mostly *can this row be safely deleted*.
2. **Build the trio.** Write `AR_DELSUPL`, `AC_DELSUPL`, `TS_DELSUPL`. Test it via `TS_DELSUPL` from a 5250 command line.
3. **Wire it to the web frontend.** Add a Mezzio handler that calls `AC_DELSUPL` via the toolkit. Now your web users can delete a supplier through the new path.
4. **Leave SPMAINT alone.** The 5250 users still use it. They don't care that something new exists. The two coexist.
5. **Pick the next action.** Repeat.

Over the course of a few months, the entire supplier action set moves to Action APIs. Each one independently, on its own timeline, without a high-stakes cutover. The green screen `SPMAINT` either continues to exist as a power-user tool, or eventually gets replaced by a green-screen *wrapper* that calls the same Action APIs. (Yes, you can — and probably should — write a green-screen menu that calls headless RPG. The pattern is symmetric.)

We treat this in detail in [Migrating Existing Programs]({% link 09-migrating.md %}). For now the point is: you don't have to commit to a big-bang rewrite. You commit to a pattern, and you migrate at the pace that fits.

[Next: Writing Your First Action]({% link 04-first-action.md %}){: .btn .btn-primary }
