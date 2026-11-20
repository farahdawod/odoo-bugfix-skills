# Basis: Pre-commit check

A skill built from this basis takes a finished fix and its tests, before
anything is committed, and answers four questions:

1. Does the change follow the house rules and survive a sceptical review?
2. Does it open a security hole?
3. Does it hold on the edge cases of the flow it touches, on a real
   database?
4. Do the tests fail without each part of the fix?

It ends with a report, which is its final summary. It flags; it changes code only where the person has
said it may.

Read first: `shared/stable-rules.md`, `shared/code-rules.md`,
`basis/reference/security-for-addons.md`.

## Facts

### Check 1: rules and review

**Scope the change exactly.** Take the diff from the working tree against
the base it will be merged into, in both repositories if the change has two
halves. Note the target branch: stable or master.

**Map every changed file to the rules that apply to it.**

| Changed file | Rules |
| --- | --- |
| Any file, when the target is stable | `shared/stable-rules.md`, before everything else |
| Python | `shared/code-rules.md`: Python, translatable strings, ORM, performance |
| `views/`, `data/`, any XML records | XML |
| `controllers/` | Controllers, plus the routes section of the security reference |
| `security/` | The access control section of the security reference |
| Report templates | Reports |
| `tests/` | The facts in `basis/3-regression-test.md` |
| Anything under `static/` | JavaScript and SCSS |
| Every file | Comments, strings, commit messages |

A file left without a rule is a mapping mistake.

**Two passes.** The rules are the floor. The second pass judges the change
on its merits, as its author's most sceptical colleague would, and finds most
of the blocking defects.

For every changed hunk, look for the input or state that makes it wrong:

- empty recordset, one record, several records;
- edge values, zero, negative, rounding;
- timezones and dates at a boundary;
- another user, another company, another language;
- the second run: called twice, re-opened, duplicated;
- two transactions at once.

And check that:

- the change does what its description claims, no less and no more;
- every line of the diff is needed for the fix (a line that is not, in
  stable, is a line to remove);
- the cost at scale is acceptable: no query per record added.

**Code the diff never shows.** For each method, field, template, XML id,
context key or returned structure the diff changes, find its consumers
across every repository on the addons path and confirm they still hold:

- overrides still call `super()` with valid arguments, and the new
  behaviour holds with what they add;
- renamed or removed names are not referenced in Python, XML or JavaScript
  (a stale reference fails only at runtime, or only when that module is
  installed);
- a new key or a changed requirement is met by every producer, including
  the JavaScript twin of a Python one;
- in stable, a changed signature is a defect on its own.

### Check 2: security

Use `basis/reference/security-for-addons.md`. For a bug fix the check is
narrow: the changed lines, the methods they sit in, and what the change makes
newly reachable. Start with the table "What a bug fix most often gets wrong",
then sweep.

A pattern match is not a finding. A finding names a lower-trust actor and
what they gain.

### Check 3: edge cases on a temporary database

Tests show the scenario the author thought of. This check goes looking for
the ones they did not.

**Setup.**

- A fresh database for the check, with the modules the flow needs. Drop it
  afterwards.
- Reproduce the original bug first, on the code without the fix. A scenario
  that does not fail before the fix proves nothing after it.
- Then the same scenario with the fix.

**Ways to drive the flow**, from cheapest to most faithful:

- the shell (`odoo-bin shell`): scriptable, rolled back unless committed,
  good for model logic and for switching user and company;
- a throwaway test method run with `--test-tags`: same, with test helpers
  and `Form` to play the view;
- RPC calls against a running server: what an integration would send;
- the browser, against a running server: the only way to see rendering
  and client-side logic (Owl state, browser storage, debounced saves).

Whether the agent can drive a browser depends on where it runs. An agent
with browser tools (for example Copilot in VS Code, or one with a Playwright
server) follows the steps on the page itself. Without browser tools, a
throwaway `HttpCase` that runs a tour drives the headless Chrome that Odoo's
tests use, when Chrome is installed. Elsewhere, UI cases are driven through
`Form`, which plays the view and its onchanges.

In the browser, play the steps at a human pace, and for timing-dependent
cases once slowly and once fast. Wait for the page to settle before reading
it: an early read shows the state before the fix took effect. Keep a
screenshot of each browser case for the report.

A case that can only be seen in the browser, when no browser can be driven,
is listed as not tried, with the steps for the person to check it by hand.

**Edge cases to choose from.** Test the edge cases of what the changed code
touches, and only those. Go through the table, keep the dimensions the diff
reads or writes, and try the cases most likely to break first. Do not spend
the budget on dimensions the code does not touch.

| Dimension | Cases |
| --- | --- |
| Recordset | empty, one, several, several with mixed states or companies |
| Entry point | form with onchange, direct `create`/`write`, import, cron, portal or website route, another module's call (see `shared/odoo-context.md`) |
| User | admin, a normal internal user with only the app's base group, portal, public |
| Company | single, two companies, record of company A handled from company B, record with no company |
| Currency and numbers | foreign currency, zero, negative, rounding at the smallest unit, a different unit of measure |
| Dates | user in a timezone far from UTC, end of month or year, a locked or closed period |
| Records | archived, duplicated (`copy`), deleted afterwards, in each state of the flow |
| Module set | only the fixed module and its dependencies; with the apps commonly installed alongside; with the bridge modules |
| Existing data | data created before the fix, code updated without a module update; then with the update |
| Language | a user in another language when the fix touches strings, reports or mails |
| Repetition | the action run twice; undo and redo (cancel, reset to draft, confirm again) |
| Combinations of settings | the options and values the touched code reads, in different combinations: one set and the other not, the reverse, both, neither; a setting on the parent record against the same setting on its lines |

**The existing-data case** deserves its own run in stable: create the
database and the records on the base commit, switch the code to the fix,
restart without `-u`, and check the flow. Then update the module and check
again. This is what production does.

**Record what was tried.** Each case: what was done, what was expected, what
happened. A case that could not be tried is listed as not tried, with the
reason.

### Check 4: the tests fail without each part of the fix

**Why per part.** A test that fails when the whole fix is reverted can still
leave half of the fix untested, or reveal that half of the fix is not needed.

**Procedure.**

1. With the complete fix: the new and changed tests pass.
2. With the complete fix reverted and the tests kept: they fail, and the
   failure is the bug (the assertion on the behaviour), not a setup or
   import error.
3. For each part of the fix: revert only that part, run the tests, and
   expect at least one to fail for the reason that part exists.
4. Restore everything and confirm the working tree is identical to where it
   started (`git diff` against the saved diff).

**A part** is one logical change: a hunk, or several hunks that only make
sense together (a new helper and its call).

**Reverting one part.** Save the diff first. Apply the reverse of that
part's hunks with `git apply -R` on a patch holding only them, run the
tests, then re-apply the patch. At the end, verify that the working tree
matches the saved diff. Python needs a restart between runs; a test run that
installs or updates the module picks up XML changes.

**When a part does not make a test fail**, one of these is true:

- the part is not needed: remove it, the diff gets smaller;
- the part is needed but untested: extend the test or add a case;
- the part cannot reasonably be tested: say so in the report, with the
  manual check that covers it (often one of the edge cases of check 3).

**Wider runs.** The fix must not break what already passed: run the test
file, then the module's tests, then the tests of modules that depend on it
when the fix touched something they use.

### The report

The report is given in the conversation, in this order: findings first, then
what was run, then what was not checked and why.

For each finding: file and line, the defect, the concrete scenario in which
it fails, a proposed fix, and where it came from (the rule, the security
section, or *judgement* for the merits pass). A finding that rests on code
outside the diff names the revision it was read at.

A check that was skipped silently is worse than one that failed.

## Fixed rules

| Rule | Why |
| --- | --- |
| All four checks run, or the report says which did not and why. | The checks cover different failures; passing three says nothing about the fourth. |
| Stable rules are checked first and outrank the code rules. | In stable the existing file style wins, and a guideline "fix" is itself a violation. |
| Every changed file is mapped to at least one rule set. | A file with no rules means the mapping is wrong, not that the file is exempt. |
| Consumers and overrides are searched in every repository on the addons path. | Breaks in other modules show only at runtime. |
| Edge cases are chosen from what the changed code touches, most likely to break first. | Trying every dimension on every fix uses up the budget before the cases that matter are reached. |
| The original bug is reproduced on the code without the fix before the fix is verified. | Otherwise the scenario may never have triggered the bug. |
| Edge cases run on a temporary database that is dropped afterwards, never on a shared or customer database. | The check writes data and switches users. |
| Each part of the fix is reverted on its own, and the failure must be for the right reason. | An import error also makes a test fail. |
| The working tree is restored exactly after reverting parts. | A half-reverted fix that gets committed is worse than no check. |
| A security finding names an actor and a result; a pattern match alone is reported as a note. | Reports full of pattern matches get ignored, and the real one with them. |
| Flag by default. Change code only for what the person has allowed. | The person owns the fix and the decision. |
| Confirm every API on the target branch before flagging its use or its absence. | See the version traps. |

## Decisions

Ask these; each has a default.

1. **In what order do you want the checks, and does a failure stop the
   rest?** *Why:* the edge-case run is the slow one; some people want it
   last, some first because it changes the fix most. *Default:* 1, 2, 4, 3;
   keep going after a failure and report everything.
2. **What may the skill fix by itself?** *Default:* nothing in the fix;
   in the tests, only what makes an already intended assertion correct.
   Everything else is flagged.
3. **How do you create and name a temporary database, and with which
   modules?** Your own script, a template database, plain `odoo-bin`? Demo
   data? *Why:* this is the most environment-specific part. *Default:* plain
   `odoo-bin -i`, a name with a recognisable prefix, dropped at the end.
4. **How much may the edge-case run cost?** A number of cases, a time
   limit, a stop rule? *Default:* one case for each dimension the changed
   code touches, the most likely to break first, then stop.
5. **How wide is the test run before commit?** *Default:* the test file,
   then the module; dependents only when the fix touches something they use.
6. **How strict is the security check for an ordinary fix?** The short
   table only, or the full sweep every time? *Default:* the short table
   always; the full sweep when the diff touches a controller, `security/`,
   a `sudo()`, SQL or HTML rendering.
7. **Review severity: what blocks and what is a note?** *Default:* blocks:
   stable violations, a broken consumer, a security finding, a test that
   does not fail without the fix. Notes: style on new lines.

## Your material

- The commands or scripts you use for databases and test runs.
- Optional: review comments on your last few fixes: what was sent back and
  why.
- Optional: one fix where part of the diff turned out to be unnecessary.

## Test prompts (optional)

Testing the built skill is optional. The agent asks the person first whether
they want to do it, and skips this section if they do not.

Give the built skill a past fix as it was first submitted, before review. A
good run flags what review flagged, reproduces the bug before verifying the
fix, lists the edge cases it tried with results, and shows for each part of
the fix which test fails without it. Include one fix with an unnecessary
hunk, one where the test passed without the fix, and one that added a
`sudo()`.