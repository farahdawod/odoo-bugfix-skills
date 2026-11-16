# Basis: Write the fix

A skill built from this basis takes the result of the investigation (root
cause, responsible module, target branch) and produces the smallest change
that removes the root cause, in the right module, within what a stable branch
allows.

Out of scope: finding the root cause (stage 1), the regression test
(stage 3), verification (stage 4).

Read first: `shared/stable-rules.md`, `shared/code-rules.md`.

## Facts

### The right module

- The fix belongs to the module that owns the wrong assumption, which the
  investigation named. That is not always the module where the error shows.
- A module cannot reference fields, models or XML ids of a module it does
  not depend on. If the bug needs both A and B installed, the fix goes in
  the one that depends on the other, or in their bridge module, not as a
  reference to B inside A.
- If A offers no way for B to adjust its behaviour, the stable-safe form is
  usually a small private hook in A (a method returning a value or a domain)
  that B overrides. Adding a private method is allowed in stable; changing
  an existing signature is not.
- A fix can have a half in community and a half in enterprise. The two
  halves always live on branches of the same name, one in each repository,
  and each half must be safe if the other is not yet pulled.

### Target branch and forward-port

- The fix is written on the oldest supported branch where the bug
  reproduces, which the investigation established.
- A bot on GitHub forward-ports the merged fix to the newer versions. The
  fix is written once; it is not re-submitted per branch.
- The forward-port conflicts where the newer branches differ from the
  target. Look at the same code on the newer branches and note in the
  final summary where it differs, so the conflict is expected and not a
  surprise.

### Smallest change

- Smallest means the fewest changed lines that remove the root cause, not
  the fewest lines that make the report go away.
- Fix the writer of the bad state rather than every reader of it.
- If two entry points share the defect (for example an onchange and the
  `create` path), the fix covers both, ideally by fixing what they share.
- Do not move code, rename, reformat or reorder while fixing. If a line must
  change, change only what the fix needs on that line.
- Keep `super()` calls, argument lists, return types and return shapes. Other
  modules call and override the method.

### Look for an existing helper before writing code

A helper that already exists makes the diff shorter, matches how the rest of
the module solves the same problem, and has already met the edge cases.

Where to look, nearest first:

1. The same model and its mixins: methods named `_get_*`, `_prepare_*`,
   `_is_*`, `_can_*`, `_check_*`. Many models already expose the value or the
   condition the fix needs.
2. How the neighbours do it: grep the same module, then the modules it
   depends on, for the same pattern (the same field compared, the same
   record searched).
3. The parent model or the comodel: currency, unit of measure, company and
   partner records carry their own rounding, conversion and comparison
   methods.
4. Framework tools: float comparison and rounding, date and timezone
   conversion (`context_today`, `context_timestamp`), formatting of amounts
   and dates for a language, recordset operations (`filtered`, `mapped`,
   `sorted`, grouping), `Command` for x2many writes, `with_company`,
   company consistency checks.

Confirm the helper exists on the target branch and behaves the same there. A
helper added in a later version is not available to a fix in an older one.

### Things a diff changes without showing it

For each method, field, template, XML id or context key the fix touches, find
who else uses it across every repository on the addons path:

- Overrides: `git grep "def <name>("`. Do they still call `super()` with
  valid arguments? Does the new behaviour hold with what they add?
- A value a method returns, or keys of a dict it builds: every caller and
  every override that extends the dict, including the JavaScript side when
  the data goes to the client.
- A template or view node: every inheriting view whose xpath anchors on it.
- A changed condition (a domain, an `invisible` expression): every place that
  duplicates the same condition in Python, XML or JavaScript.

### After the fix, look at the state again

Removing one cause can expose a second. A value that was always dropped, or
a path that always failed early, was never exercised; once the fix lets it
through, its correctness matters for the first time.

- Run the reproduction again with the fix and inspect the stored values of
  every field involved, not only the symptom that was reported. Compare each
  with the value it should have, not with "the report no longer happens".
- Pay most attention to what the fix makes newly saved, newly read or newly
  reachable.
- A second defect found this way that violates the same invariant belongs
  to the same fix. Take it back to the investigation first: state its cause
  the same way as the first one before changing more code.

### Fixes that need a module update

Avoid a fix that only takes effect after a module update: many databases
pull the code and restart without updating. When a view or data change is
unavoidable, the Python must work correctly on a database that still has the
old XML, and the final summary says that an update is needed.

### What a fix does not reach

- Rows already stored. Correcting a stored compute, a default or a
  constraint changes future writes only.
- `noupdate` records in existing databases.
- Views in databases that pull the code without updating the module.

These are not reasons to do more in the diff. They are things the person
must be told with the fix.

## Fixed rules

| Rule | Why |
| --- | --- |
| Everything in `shared/stable-rules.md`. | A stable branch runs in production. |
| The fix removes the root cause named by the investigation. If writing it shows the root cause was wrong, go back; do not adapt the fix to a new theory silently. | Otherwise the test and the checks that follow verify the wrong thing. |
| After the fix, re-run the reproduction and check the stored value of every field the fix makes newly saved or newly used, not only the reported symptom. | A fix can bring a dormant defect to life; "the symptom is gone" does not show that the data is right. |
| A fix with a community half and an enterprise half uses the same branch name in both repositories. | The two halves are found, tested and merged as one change by that name. |
| Search for an existing helper, through the four levels in this basis in order, before adding logic, and say in the final summary what was searched. | Shorter diff, consistent behaviour, and review will ask. |
| Check overrides and consumers of everything the fix touches, in every repository on the addons path. | A break in another module shows only at runtime, or only when that module is installed. |
| Confirm on the target branch every API the fix uses. | See the version traps. |
| New lines follow `shared/code-rules.md`; untouched lines stay as they are. | Guidelines for what you add, minimal diff for what exists. |
| No new `sudo()`, raw SQL, public method or route without a stated reason. | Each one widens what an attacker can reach; see `basis/reference/security-for-addons.md`. |
| When the correct fix is not allowed in stable, stop and report both the proper fix and the stable-safe alternative. | The choice between them is the person's, and often the reviewer's. |

## Decisions

Ask these; each has a default.

1. **Do you want to see a plan before any code is written?** *Why:* some
   people want to approve the approach, others want a diff to react to.
   *Default:* a three-line plan (where, what, why there), then the diff.
2. **How small is "smallest" for you?** When a clean fix needs a small new
   private method and a dirty one fits in the existing line, which do you
   pick in stable? *Why:* reviewers on different teams answer this
   differently. *Default:* the shortest diff that is still readable; a new
   private method only when another module needs to override it or two call
   sites share it.
3. **Comments in the fix: when?** *Default:* only where the code would look
   wrong without one; never restating the ticket.
4. **Existing wrong data: mention it, or propose a separate script?**
   *Why:* a code fix does not repair stored values. In both cases the fix
   itself changes no data. *Default:* mention it in the final summary with a
   query that finds the affected rows.
5. **What must it never do without asking?** *Default:* add a non-stored
   field, touch a file outside the responsible module, edit XML data.
   (What stable forbids outright, such as changing a signature or adding a
   stored field, is not on this list: asking does not make it allowed.)
6. **What do you want in the final summary, next to the diff?** *Default:* the root
   cause in one sentence, why the fix is in that module, helpers considered,
   consumers checked, where the code differs on newer branches, whether a
   module update is needed, what the fix does not reach.
7. **Which review comments do you get most on your fixes?** *Why:* the
   skill should check for exactly those before it finishes.

## Your material

- Helpers you reach for often in your area (for example the accounting,
  stock or website helpers you know by heart).
- Any team convention on plans, comments or final summaries.
- Optional: two or three merged fixes of yours with the review discussion,
  especially ones where the first version was rejected.

## Test prompts (optional)

Testing the built skill is optional. The agent asks the person first whether
they want to do it, and skips this section if they do not.

Give the built skill a past ticket's root cause and the code at the commit
before the fix. A good run puts the change in the same module as the merged
fix, is not larger than it, uses the helper the merged fix used when there
was one, and leaves signatures and data model untouched. Include one ticket
where the obvious fix was in the wrong module and one where the proper fix
was not allowed in stable.
