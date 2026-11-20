# Basis: Write a regression test

A skill built from this basis takes a fix and its root cause and produces a
test that fails without the fix and passes with it: preferably by extending
a test that already exists, otherwise a new test placed and tagged the way
its file expects.

Out of scope: proving that the test fails without each part of the fix
(stage 4 does that).

Read first: the version traps in `shared/odoo-context.md`. Test defaults are
among the things that changed between versions.

## Facts

### Choosing the test type

Pick the lightest type that can show the bug.

| The bug is in | Test type | Notes |
| --- | --- | --- |
| Model logic: a compute, a constraint, a business method | `TransactionCase` | Fastest and most stable. The first choice whenever the bug can be shown by calling Python. |
| What the form does: an onchange, a default, a field that is invisible, readonly or required | `TransactionCase` using the server-side `Form` helper | `Form` plays the view: it runs onchanges and refuses to set a field the view makes readonly or invisible. It shows view bugs without a browser. |
| A controller, a route, access through the portal or website | `HttpCase` | Requests go through real dispatch, with a real session and user. |
| An interaction that only exists in the browser across several screens | Tour, started from an `HttpCase` | Not favoured: tours add a lot of overhead and usually fail because of race conditions. Write one only when there is no other option. |
| Pure client logic in one component or utility | JavaScript unit test | The framework differs by branch. |

A bug that reproduces through the UI can often still be shown with a
`TransactionCase`: reproduce the same `create` or `write` the client sends.
Prefer that when it tests the same root cause.

### Look for an existing test first

Before writing a new test, look for one that can carry the regression.

Where to look:

- `tests/` of the module that was fixed: grep for the method, the field, the
  model, and words from the flow.
- `tests/` of modules that depend on it, when the scenario needs them
  installed.
- The module's common test classes (the files that define `setUpClass` with
  shared records). They tell you what data already exists.

Three outcomes, in order of preference:

1. **Extend an existing test.** An existing test already walks the same
   flow, and the regression is one more assertion or one more step at the
   point where the bug showed. Good when the addition does not change what
   that test is about.
2. **New method in an existing class.** The class already builds the
   records the scenario needs. The new method reuses them.
3. **New class or new file.** Only when nothing fits. A new file must be
   named `test_*.py` and imported in `tests/__init__.py`, or it never runs
   and nothing complains.

A test lives where its scenario can be built. If the scenario needs an
enterprise module, the test goes in enterprise even when the fix is in
community.

### Placing a new test

- Look at the order of the test methods already in the file. If they are
  alphabetical, insert the new one in alphabetical position. If they are
  grouped by feature, put it with its group. If there is no order, put it
  next to the most closely related test, or at the end.
- Name it for the behaviour: `test_<flow>_<condition>`. Follow the naming
  already used in the file.
- Describe the scenario the way the file does. If the tests in the file use
  docstrings, the new test has a docstring; if they use comments, a comment;
  if they use neither, neither.
- No ticket reference in the test code. The ticket is referenced in the
  commit message.

### Tagging

- Whether a test runs at install or after all modules are loaded is decided
  by its tags, and the default changed between versions. Read what the class
  and its neighbours do on the target branch.
- A method added to an existing class inherits the class's tags. Do not
  retag the class.
- A new class sets its tags explicitly, with exactly one of `at_install` and
  `post_install`. Both makes the class run twice; neither makes it never
  run; in both cases the only signal is a log warning.
- `post_install` is the normal choice: the registry is fully loaded, which
  is what production looks like. `at_install` is for a test that must run
  before other modules load, and needs a comment saying why.
- An `HttpCase` stays `post_install`.

### Ways a test silently does not run

- The file is not named `test_*.py`, or is not imported in
  `tests/__init__.py`.
- The class is imported into the test file but defined elsewhere.
- The subclass defines no test of its own and inherits its `test_*` methods.
- The module is not installed in the run.
- The tags exclude it.

A new or moved test should be seen in the log at least once.

### Writing it

- Assert behaviour, not implementation: the value the user would see, the
  error they would get, the record that exists afterwards. Not the number of
  calls or the name of a private method.
- Build the smallest scenario that shows the bug. Reuse the records of the
  class; create only what is missing.
- Several records, or any operation on several records, in one call: one
  `create()` with a list of vals, one `write()` or `unlink()` on the
  recordset. A loop of single calls is slower and gets sent back.
- Do not change shared test data in a way that leaks into the tests that
  run after. Database writes in a `TransactionCase` method are rolled back;
  these are not: assignments to class attributes, mutations of shared
  Python objects built in `setUpClass` (a vals dict, a list),
  monkeypatching without `self.patch`, and commits in an `HttpCase`. To
  vary a shared record, create a new one or change it inside the method.
- When the scenario needs a module that is not a dependency of the test's
  module (typically when the fix reads it behind a guard), the test skips
  itself when that module is absent
  (`self.env['ir.module.module']._get('<module>').state != 'installed'`),
  and the run that checks it must have the module installed.
- Do not rely on demo data being there.
- If the bug depends on who the user is, run the scenario as that kind of
  user (`with_user`). Tests otherwise run with full rights and hide access
  bugs.
- If the bug depends on the company, the currency, the language, the date
  or the timezone, set it up explicitly; freeze the time when the date
  matters.
- An expected error is asserted (`assertRaises`) and its log is muted with
  `mute_logger`, so that an expected ERROR does not reach the test log.
- No `assertQueryCount` in a business test: it breaks on any unrelated ORM
  change.
- A tour's last step leaves the client in a stable state (no pending edit,
  no request in flight), or the teardown fails at random.

### A test that proves nothing

- It passes without the fix.
- It fails without the fix for another reason: a setup error, an import
  error, a missing record.
- It asserts something the fix did not change.
- It repeats the implementation, so it would pass with any implementation
  that has the same shape.

## Fixed rules

| Rule | Why |
| --- | --- |
| Search for an existing test before writing a new one: the fixed module's tests and its common classes, then the tests of direct dependents. Say what was found. | Less new code in stable, shared setup, and the regression sits next to the tests of the same flow. |
| The test fails without the fix, for the reason the bug describes. | Otherwise it is not a regression test. |
| Write a tour only when no other test type can show the bug. | Tours add a lot of overhead and usually fail because of race conditions. |
| Follow the file: its base class, its order, its naming, its tags, and whether its tests use docstrings or comments. | A stable branch keeps existing style; a test that looks foreign gets sent back. |
| Check whether the file's tests are in alphabetical order before placing a new one. | Many test files keep their methods sorted. A test inserted out of order breaks a convention the reviewer holds and is harder to find later. |
| No ticket reference in test code. | The reference belongs in the commit message, where the history keeps it next to the change; in the code it tells the reader nothing about the behaviour. |
| A new class has explicit tags with exactly one of `at_install` and `post_install`. | Both or neither fails silently. |
| Confirm test base classes, helpers and tags on the target branch. | They changed between versions. |
| Run the new or changed test at least once and read the log to see that it ran. | Tests are skipped silently in several ways. |
| Do not weaken or delete an existing assertion to make the suite pass. If the fix changes intended behaviour that a test asserted, stop and report it. | That test may be describing the behaviour someone else relies on. |

## Decisions

Ask these; each has a default.

1. **Extend an existing test or add a method: where is your line?** Some
   reviewers want one test per bug, some prefer one more assertion in a test
   that already walks the flow. *Why:* this is the first choice the skill
   makes. *Default:* extend when the existing test already reaches the buggy
   step and the addition is a few lines; otherwise a new method in the
   existing class.
2. **Do you write the test before the fix or after?** *Why:* it changes the
   order of the whole workflow and whether the skill is given a fix or a
   root cause. *Default:* after the fix, then checked against the code
   without it.
3. **File without an order: where does the new test go?** *Default:* next
   to the most related test.
4. **How do you run a single test locally?** *Default:* the `--test-tags`
   form in `shared/odoo-context.md`.
5. **How much of the suite do you run before you are satisfied?** Only the
   new test, the file, the module, the dependents? *Default:* the new test
   here; wider runs in stage 4.
6. **When a fix cannot reasonably be tested, what then?** *Default:* stop
   and say why, with the manual steps that verify it.

## Your material

- The command you use to run one test.
- Optional: two or three regression tests of yours that were merged, and
  one that review asked you to rewrite.

## Test prompts (optional)

Testing the built skill is optional. The agent asks the person first whether
they want to do it, and skips this section if they do not.

Give the built skill a past fix (the diff and the root cause) without its
test. A good run finds the same existing test or class the merged test used,
chooses the same test type or a lighter one, places and tags it as the file
expects, and the test fails when the fix is reverted. Include one case where
the right answer was extending an existing test, and one where the file was
alphabetically ordered.