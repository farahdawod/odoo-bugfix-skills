# Basis: Investigation

A skill built from this basis takes a bug report and ends with a final
summary: a confirmed reproduction, the module responsible, the root cause
stated as a violated assumption, and optionally the commit that introduced
the bug.

It has three steps. The third is optional by design.

Out of scope: changing code (stage 2), writing the test (stage 3).

Read first: `shared/odoo-context.md`.

## Facts

### The report and the environment

**What a report contains.** It always has the steps to reproduce. It usually
has the expected and the actual behaviour. It sometimes has a suspected
cause, and a traceback when the bug raises one.

- When expected and actual behaviour are missing, work them out from the
  steps before anything else: the investigation needs a statement of what is
  wrong.
- A suspected cause in the report is a lead, not a finding. Check it like
  any other theory; reports often name the place where the bug shows.

**Two ways to reproduce: server side and in the browser.**

- *Server side.* The shell, a throwaway test, the server-side `Form` helper
  (which plays a form view, onchanges included), or RPC calls. Steps written
  for the UI are translated into the calls the client would make: the
  button's method, the onchange sequence, the `create` or `write` with the
  fields the view contains. Fast and scriptable, but it skips everything the
  client does.
- *In the browser.* Whether the agent can drive one depends on where it
  runs. An agent with browser tools (for example Copilot in VS Code, or one
  with a Playwright server) opens the page against a local server and
  follows the steps literally. Without browser tools, a throwaway `HttpCase`
  that runs a tour drives the headless Chrome that Odoo's tests use, when
  Chrome is installed. Rendering and client-side logic (Owl state, browser
  storage, debounced saves) can only be reproduced this way. Play the steps
  at a human pace, and for a bug that "does not happen every time" once
  slowly and once fast. Wait for the page to settle before reading it:
  restores and RPCs are asynchronous, and an early read shows the state
  before the bug.
- When the bug can only be seen in the browser and no browser can be
  driven, say so and ask the person to observe it.

### Step 1: reading a traceback

**Shape.** An Odoo traceback starts in framework frames (HTTP dispatch, RPC
call, ORM) and ends in addon frames. The last frame is where the error was
raised, which is often not where the bug is.

**The same method name in several consecutive frames is a `super()` chain.**
Each frame is one module's override of that method, from the last loaded
module down to the base definition. Reading the chain tells you which modules
take part in the call.

**The traceback hides as much as it shows.**

- An override that does not call `super()` cuts the chain: the modules below
  it never appear.
- An override that ran and returned normally before the error is gone from
  the stack, though it may have produced the bad value.
- Computes, constraints and flushes run later than the line that triggered
  them. A traceback through `flush` or `_recompute` points at the field being
  computed, not at the write that made it stale.

**What the exception type suggests.**

| Exception | Usual meaning | Where to look |
| --- | --- | --- |
| `AccessError` | Access rights or a record rule; the message names model, operation and user | Does it reproduce as admin? Is a `sudo()` missing or is the access legitimately denied? |
| `MissingError` | Record deleted, or an id from another model or a stale cache | Who deleted it; unlink order; ids kept across a transaction |
| `ValidationError` | A Python constraint or an SQL constraint | Grep the message; find which write violated it |
| `UserError` | A deliberate business check | Grep the English source text of the message; the check may be right and the caller wrong |
| `ValueError: Expected singleton` | Code written for one record received several, or none | The caller's recordset; a missing loop or `ensure_one` |
| `KeyError` or `AttributeError` on a field name | The field does not exist in this module set or version | Missing dependency in the manifest; module not installed; renamed field |
| `psycopg2` integrity errors | `NOT NULL`, unique or foreign key violation at flush | The values of the create or write, defaults, `ondelete` |
| Serialization or concurrency error | Two transactions touched the same rows | Usually retried by the framework; look for long transactions or a cron |
| View or QWeb error | The message names the view or template and often the line | Inherited views of that arch; a field or node an xpath expected |
| `RecursionError`, "compute failed to assign" | Computes depending on each other, or a branch that assigns nothing | The compute and its `@api.depends` |
| JavaScript error dialog | A client error, or an RPC error wrapping a server traceback | If it wraps a server traceback, read that one first |

**Finding the other participants.**

- Overrides of a method: `git grep -n "def <method>("` across every
  repository on the addons path.
- Extensions of a model: `git grep -n "_inherit = '<model>'"` (also the list
  form).
- Definition of a field: `git grep -n "<field> = fields\."`.
- Users of an XML id, a template, a context key: grep Python, XML and
  JavaScript, since each can reference the others.

**Which module is responsible.** Not always the one that raises. Three
candidates for every error: the frame that raised (its check may be correct),
the caller that passed bad data, and an override that changed arguments, a
return value, or skipped `super()`. The responsible module is the one whose
code breaks an assumption the others are entitled to make.

**No traceback.** For a wrong value or a UI problem, find the entry point
instead: the button's `name` in the view gives the method; the view's fields
and their onchanges give what the client sends; the server log gives the
order of calls.

When the view has no button for the action, the entry point is usually on
the client side: an Owl component, a widget, a client action or a systray
item calls the server itself. Grep `static/src` of the module and of the
modules that depend on it for `orm.call`, for `rpc(` (routes), and for the
model and method names. Older branches use other client APIs, so the method
name is the search that works on every branch. The person can supply the
route and the RPC payload from their browser's network tab when the entry
point is not clear from the code.

### Step 2: finding the root cause

**Trace the bad value back to where it was written.** A wrong value was put
there by one of: a `create` or `write`, a default, a compute, an onchange, an
inverse, a data file, an import, or raw SQL. The symptom site reads the value;
the root cause is the writer, or the missing writer.

**An access error has no bad value.** For an `AccessError` the root cause is
a read or a write done in the wrong environment, or a check that is right.
Compare the model's access rights and record rules with the groups that can
reach the flow: who may open the parent record but may not read this model.
Then decide whether the access is legitimately denied, or the code must read
in a narrow `sudo()` because the data it needs is not the user's to choose.
The reader, not a writer, is then where the bug is.

**A client-side bug has no server value.** When the wrong state lives in the
browser, trace it to the code that sets, saves, restores or clears it, from
the component's handler in `static/src` and the `patch()` of that component
in other modules.

**Look for the same bug already fixed next to the suspect code.** A class
of bug is rarely fixed everywhere at once. Before building a theory:

- read the comments in the view, the method and the data file around the
  suspect code: a comment explaining an odd line (an invisible field, an
  extra reset, a special case) often describes the exact mechanism;
- read the history of the file (`git log` on it, `git log -S` on the odd
  line) for earlier fixes of the same kind;
- then check the siblings the earlier fix did not cover: fields with the
  same role, methods with the same shape, the other views of the same
  model. The bug at hand is often the one that was missed.

**Tests for a band-aid.** A change is probably a band-aid if:

- it sits where the wrong state is observed, not where it is produced;
- it special-cases the reported scenario (`if` on one type, one company, one
  state) without saying why that case is special;
- the same bad state would still break another reader of the same data;
- the same method reached from another entry point still misbehaves (see
  "Ways the same business flow is entered" in `shared/odoo-context.md`).
  Note the other entry points as candidates in the final summary; they are
  tried in the pre-commit check, not here;
- you cannot state the invariant that was violated in one sentence.

**Root causes that are often missed.**

*XML wiring.* The Python is right but something declarative is not connected.

- A data or view file is missing from the manifest's `data` list, or loaded
  before a record it references.
- An inheriting view has the wrong `inherit_id`, or its xpath matches a
  different node than intended since the parent changed.
- An action lacks the `view_id`, `domain` or `context` the flow relies on
  (a `default_*` key, a `search_default_*` filter, `active_test`).
- A field that an onchange sets is not in the view. The client saves only
  the fields the view contains, so the value is computed in memory and
  silently dropped at save. The record is stored with the field's default
  while the visible fields set by the same onchange are saved, and the two
  no longer agree. The usual remedy is the field in the view as invisible.
- A field that the view's expressions or an onchange read is not in the
  view, so the client never loads it. How the client treats this differs by
  version.
- A `groups` attribute hides a node for the reporting user only.
- A report action's `report_name` does not match the template id; a
  `t-call` targets a template from a module that is not a dependency.
- A JavaScript or SCSS file is missing from the `assets` bundle.
- A record is `noupdate`, so the corrected file never reached existing
  databases.

*Stored and computed fields.* Check what happens at each recomputation, not
only the first one.

- `@api.depends` misses a field the compute reads: the stored value goes
  stale and nothing recomputes it.
- The compute reads something that is not a field dependency: the current
  company, user, language, date, or a context key. A stored value then
  freezes whatever was current at the last recomputation. That value belongs
  in a non-stored compute with `@api.depends_context`.
- An editable computed field (`readonly=False`): when a dependency changes,
  the recompute overwrites what the user typed.
- A compute that does not assign in every branch, or assigns only for some
  records of `self`.
- A compute that depends on the record being saved: in an onchange, records
  are new, `id` is not a database id, and the saved record is `_origin`.
- `precompute` fields are computed before the record exists; code that needs
  the record's id or its lines sees nothing.
- A dependency through an x2many (`line_ids.amount`) recomputes the parent
  for every line change, which is correct but can be the performance bug.
- Fixing a stored compute does not fix rows already stored. Existing
  databases keep the wrong values until something triggers a recompute.
- A stored `related` field is a stored compute with the same rules.
- Raw SQL that writes a dependency does not trigger recomputation, and raw
  SQL that reads a field may run before the pending values are flushed.

*Onchange and in-memory records.* The logic is right for one run on a saved
record, and wrong for how the form actually calls it.

- The same onchange runs several times while a form is filled in: once with
  the default values, then again each time one of its fields changes. It
  works on the same in-memory record every time.
- Anything an onchange accumulates on the record (a running total, a
  counter, an "already done" amount) must be reset at the start of each run.
  If one such field is left out of the reset, the amount from an earlier run
  (for example the one made with the default date) is added to the final
  one, and the saved value is too large.
- When an onchange sets several fields that must stay consistent, check each
  of them twice: is it reset on every run, and is it in the view so that it
  is saved.
- Code that previews or simulates on an in-memory copy
  (`new(origin=record)`) starts from the stored values of the original. A
  bookkeeping field stored wrong at creation gives a wrong simulation much
  later and far from where it was written.

*Controllers.* The model method is right but is called in a different
environment than the backend gives it.

- `auth='public'`: the user is the public user; record rules apply to it,
  and a `sudo()` may be hiding or causing the problem.
- No backend context: no allowed companies, a different language, a website
  in the context. `env.company` is not what the backend flow had.
- Parameters arrive as strings from the query; an id compared to an integer
  never matches.
- The route `type` does not match how the client calls it.
- An override in another module was not re-decorated, changed the arguments,
  or does not pass through the parent's parameters.
- Access is granted through a token on some routes and through the session
  on others, and the two paths are not checked the same way.

*Client state.* The server is right; the browser keeps or rebuilds the
wrong state.

- State persisted in the browser (`localStorage`, `sessionStorage`,
  IndexedDB) is written in one place and cleared in another. After a
  storage change (for example from `localStorage` to IndexedDB), one of them
  still uses the old store.
- A debounced or throttled save: a save still pending when the component
  unmounts, or fired after a clear, writes stale content back; a save that
  already ran leaves a stored copy that nothing removes. A bug that "does not
  happen every time" often depends on the delay: reproduce once slowly and
  once fast.
- Mount and unmount order: `onMounted` restores, `onWillUnmount` saves. A
  component closed through another path (a dialog callback, a toggle) skips
  the cleanup that the normal path does.
- Two paths that do the same job (sending from the inline composer and from
  a full-screen dialog) with different cleanup.
- A `patch()` in another module overrides the handler without calling
  `super`.

*Other regular suspects.*

- Multi-company: a record read or created under the wrong company; a missing
  `with_company`; a company-dependent field read in the wrong company.
- Access rights: it works for the admin and fails for a normal user, or a
  test ran as superuser and hid it.
- Rounding and currency: float equality instead of the currency's or the
  unit's rounding helpers.
- Dates: a datetime is stored in UTC; "today" for the user is not the
  server's `date.today()`.
- Batch: a method correct for one record and wrong for several, or for an
  empty recordset.
- Context leakage: a `default_*` key set for one model applied to another
  created underneath.
- Archived records: `active_test` hiding or including them.
- Copy: a field that should have `copy=False` duplicated with the record.
- Module combination: the bug needs two apps installed, and neither module
  alone is wrong.

### Step 3: finding the introducing commit (optional)

**What it is for.** Knowing what the faulty commit was trying to do tells you
what the fix must not undo, and where else the same mistake may have been
made.

**Cheap first.**

- `git blame` on the faulty lines, with `-w -C -C` to see through moves and
  whitespace.
- `git log -L :<function>:<file>` for the history of one function.
- `git log -S '<string>'` to find when a string appeared or disappeared.
- Blame often lands on a forward-port, a merge of a move, or a mass refactor.
  Go one step further back with `git blame <commit>^ -- <file>`.
- The commit may be in the other repository.

**Bisect is expensive in Odoo.** Each step needs a checkout of both
repositories at matching dates, a database that works at that revision, and
a reproduction. It is cheap only when the reproduction is an automated test
that can be given to `git bisect run`.

**When to skip it.** Skip when blame already answers the question; when the
bug has been there since the feature was written (there is no regression to
find); when the reproduction is manual and long; when the fix is obvious and
does not depend on the history.

**Target branch.** The investigation works on the branch the person gives.
It does not search older versions for the oldest one where the bug
reproduces. If the target branch is not clear, ask. Read the branch as
described in "Reading branches that cannot be fetched" in
`shared/odoo-context.md`.

## Fixed rules

| Rule | Why |
| --- | --- |
| Reproduce before explaining. No root cause is stated for a bug that has not been reproduced, unless reproduction is impossible and the report says so. | An explanation that fits the traceback but was never run is a guess, and the fix built on it is a band-aid by construction. |
| Record the module set and the branch of the reproduction. | The same steps give a different result with different modules installed. |
| Confirm every API and attribute on the branch under investigation. | See the version traps. |
| Search overrides and consumers across every repository on the addons path, not only the one the traceback points at. | Enterprise and bridge modules override community code. |
| State the root cause as the invariant that was violated and the code that violated it. | It is the only form the next stage can check a fix against. |
| Do not modify code in this stage beyond temporary logging, and remove that. | Investigation that leaves changes behind contaminates the fix. |

## Decisions

Ask these; each has a default.

1. **How do you reproduce?** On a fresh local database, a copy of the
   customer's, a hosted test instance? With demo data or without? *Why:*
   decides the first commands the skill runs. *Default:* fresh local database,
   minimal module set, demo data as the branch defaults.
2. **When there is no traceback, where do you start?** *Why:* people differ:
   the view, the logs, a debugger, the payload from the network tab.
   *Default:* from the button or route to the method; from `static/src`
   when the view has no button.
3. **When a runtime value decides the path, how is it observed?** The agent
   reads the code only, adds temporary logging, or proposes breakpoints
   (file, line, expressions to inspect) for you to run in your debugger and
   report back? *Why:* people who already work under a debugger do not want
   logging lines in their tree, and the agent cannot attach to their
   debugger itself. *Default:* read the code; temporary logging when the
   code alone does not settle it, removed afterwards.
4. **Which root causes do you see most in your area?** Go through the missed
   root causes above: which ones are yours, what is missing? *Why:* the skill
   should check your frequent ones first. *Default:* the order given here.
5. **Introducing commit: when is it worth it?** Blame only? A limit on
   bisect steps or on time? Never without asking? *Why:* this step can cost
   more than the rest of the fix. *Default:* blame and `git log` always;
   bisect only with an automated reproduction and at most a fixed number of
   steps, otherwise skip and say so.
6. **What do you want in the final summary, and where should it go?** In
   the conversation, as notes for the ticket, nowhere? *Default:* a short
   summary in the conversation: reproduction, module set, responsible
   module, root cause, any failed fetch with the date of the code read,
   other entry points to try, introducing commit or "skipped, because".
7. **When should it stop and ask you?** *Default:* when it cannot
   reproduce; when the bug can only be seen in a browser and no browser can
   be driven; when the root cause is in the framework rather than an addon;
   when the reported behaviour looks intended.
8. **Does the agent reproduce in the UI, and who confirms it?** Server side
   only, in the browser itself (with its browser tools or a throwaway
   tour), or both? Do you also confirm in your own browser before it looks
   for the cause? *Why:* a browser run is slower and needs a running
   server, but it is the only way to see client-side bugs and catches what
   the translation to server calls misses; some people want to see the bug
   with their own eyes before trusting any explanation. *Default:* server
   side for model logic; in the browser when the bug is visible in the UI
   and a browser can be driven, with a screenshot of the result; no
   confirmation from you unless it could not drive a browser.

## Your material

- The commands you use to create a database and start Odoo.
- Optional: two or three tickets where the first explanation was wrong,
  with what the real cause turned out to be.
- Optional: band-aids that review sent back to you or to colleagues.

## Test prompts (optional)

Testing the built skill is optional. The agent asks the person first whether
they want to do it, and skips this section if they do not.

Give the built skill a past ticket at the commit before its fix, with the
report as it originally arrived. A good run reproduces the bug, names the
same responsible module and root cause as the merged fix addressed, and does
not stop at the frame that raised. Include one ticket with a traceback and
one without.