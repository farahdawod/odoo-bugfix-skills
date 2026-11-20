# Basis: Forward-port

A skill built from this basis takes a forward-port that the bot could not
complete cleanly, on one newer branch, and ends with the same fix working on
that branch, plus a final summary of what had to be adapted.

The bot handles forward-ports that apply cleanly and pass their tests. This
skill is for the others: a conflict, a port that applies but fails its tests,
or a port that applies and passes but no longer fixes anything.

Out of scope: the original fix and its test (stages 2 and 3). A port that
needs a new design is a new fix and goes back to those stages.

Read first: `shared/odoo-context.md` (repositories, version traps, reading
branches that cannot be fetched), `shared/stable-rules.md`.

## Facts

### What the bot does

- After the fix is merged on its target branch, the bot forward-ports it to
  each newer supported branch, one branch at a time, and opens one pull
  request per branch.
- The ports form a chain: the port to a branch is built from the port to the
  branch before it. A resolution made on one branch is what the next port
  starts from, and a mistake made there travels down the chain.
- When a port does not go through, the bot leaves its forward-port branch in
  some state: conflict markers committed, a failing test run, or a note on
  the pull request. What exactly it leaves depends on the bot. Find out from
  the branch and its history; do not ask the person what can be read.
- A fix with a community half and an enterprise half is ported in both
  repositories. The two ports to a given version live on branches of the
  same name and are resolved and tested together.

### Three ways a port goes wrong

1. **Conflict.** The patch does not apply: the lines it changes, or the
   lines around them, are different on the newer branch.
2. **Applies, fails.** The patch applies, but a test fails: a renamed
   method, a changed signature, a field that moved, a test base class or a
   tag default that changed (see the version traps).
3. **Applies, passes, fixes nothing.** The patch lands in code that no
   longer runs on the newer branch: a method kept for compatibility but no
   longer called, a view replaced by another one, a file that is no longer
   in the manifest, logic moved to another module. Tests can still pass if
   they only exercise the old path. This is the most dangerous case because
   nothing signals it.

### Reading a port

Read in this order, before touching anything:

1. **The original fix.** Its commit message (the root cause and the
   invariant), its diff on the original branch, and its test. The goal on
   the newer branch is to remove the same root cause, not to make the
   original diff apply.
2. **What the bot left.** The forward-port branch: which files carry
   conflict markers, which test failed and how.
3. **Why the newer branch differs.** For each conflicting or failing place,
   the commits between the two branches that changed it (`git log` on the
   file or `git log -L` on the function, from the original branch to the
   newer one). That commit tells you whether the code was renamed, moved,
   refactored, or whether the bug was already fixed another way.

### Four outcomes

| Outcome | What the port becomes |
| --- | --- |
| Same change, adapted | The fix with names, locations or APIs updated to the newer branch. The usual case. |
| The bug is already gone | A refactor removed it, or another fix covered it. The code change is dropped; what happens to the test is a decision. Name the commit that removed the bug. |
| The bug is there in another shape | The fix must be designed again for the newer code. That is a new fix: stop, propose the new shape, and apply the fix and test bases to it. |
| The code moved to another module | The fix goes where the code went. Check that module's dependencies allow everything the fix references. |

### Resolving a conflict

- Resolve by intent, hunk by hunk: what the original hunk did, what the
  newer code looks like at that place, what the equivalent change is there.
  Never take one side of a conflict whole.
- When a file was moved, split or renamed on the newer branch, the
  cherry-pick may show a modify/delete conflict, or apply to a file that is
  no longer loaded. Find where the content went and port there.
- A clean merge is not a correct one. Look for semantic conflicts in the
  parts that applied without complaint:
  - a call to a method that was renamed or whose arguments changed;
  - a field renamed or moved, referenced in Python, in a view, in a domain,
    or in JavaScript;
  - an xpath or a view anchor that the newer parent view no longer has;
  - a test that moved to another file, a base class that changed, a tag
    default that changed so the ported test no longer runs;
  - an API from the original branch that does not exist on the newer one,
    or exists with different behaviour (see the version traps).
- No conflict marker may remain anywhere: Python, XML, JavaScript, SCSS,
  CSV, translation files. Search for `<<<<<<<`, `=======` and `>>>>>>>`.
- If the newer branch is stable, the stable rules apply to the port as they
  did to the fix. On master they are looser, but a port is still the same
  fix, not the occasion for the better one master would allow.

### Checking the port

- The ported test fails on the newer branch without the port and passes
  with it. If it passes without the port, the bug is already gone (outcome
  2) or the test no longer reaches the bug (outcome 3); find out which.
- When the port was adapted, the test checks behaviour on the newer code
  path, not only on the path the original fix touched.
- The reproduction of the original bug, when there is one, is run once on
  the newer branch. This is what catches outcome 3.
- Run the tests of every module the port touches, in both repositories
  when the fix has two halves.
- Look ahead: on the next branch in the chain, the same place may have
  changed again. Noting it saves the next resolution from starting cold.

## Fixed rules

| Rule | Why |
| --- | --- |
| Read the original fix, its root cause and its test before resolving anything. | The port must remove the same root cause; a conflict resolved without knowing it is a guess. |
| Resolve each conflict by intent; never take one side whole. | Taking a side drops either the fix or the newer branch's own changes. |
| A port carries the same fix. Improvements, even ones the newer branch would allow, go in a separate change. | A reviewer checks a port for equivalence with the original. Anything extra is unreviewable in that frame and travels down the chain. |
| Stable rules apply when the newer branch is stable. | A stable branch runs in production whatever the origin of the change. |
| Confirm every API the port uses on the newer branch. | The original diff was written against an older branch; see the version traps. |
| The ported test fails on the newer branch without the port and passes with it. | Otherwise the port may fix nothing, and nothing would show it. |
| No conflict marker remains in any file. | A marker in XML, CSV or a translation file can pass review and break at install. |
| Community and enterprise ports to the same version are resolved and tested together, on branches of the same name. | They are one change; one half alone can break the other. |
| A port that needs a new design stops and goes back to the fix and test stages. | It is a new fix, and needs the investigation and checks of one. |
| Keep the bot's commit message and its reference to the original. | It is how the chain is followed from one branch to the next. |

## Decisions

Ask these; each has a default.

1. **What do you give the skill to start?** The forward-port branch name,
   the pull request, the original fix and the newer branch? *Why:* decides
   the first thing it reads, and whether it can reach GitHub at all from
   where it runs. *Default:* the forward-port branch name; the skill finds
   the original fix from its history.
2. **When the bug is already gone on the newer branch, what happens to the
   test?** Keep it as a guard, drop it, ask each time? *Why:* a test with no
   code change is a small diff that still protects the behaviour, but some
   reviewers reject it. *Default:* keep the test if it still runs the flow
   of the original bug, and say so in the final summary.
3. **One branch per run, or the whole chain?** *Why:* resolving the chain
   in one go is faster but piles up unreviewed resolutions. *Default:* one
   branch per run, with a note on what the next branch will hit.
4. **How much of the suite runs on the newer branch?** *Default:* the
   ported tests, then the tests of every module the port touches.
5. **What do you want in the final summary?** *Default:* in the
   conversation: the outcome (adapted, already gone, new shape, moved), each
   conflict and how it was resolved with the commit that caused it, APIs
   that changed, tests run and their result, any failed fetch with the date
   of the code read, what the next branch will hit.
6. **When must it stop and ask, and what must it never do without
   asking?** *Default:* stop when the fix needs a new design, when the bug
   is already gone, when the two halves disagree; never push, and never
   force-push, without asking.

## Your material

- The commands you use to check out a newer branch and create a database
  for it.
- Optional: two or three past forward-ports that conflicted, with how they
  were resolved.

## Test prompts (optional)

Testing the built skill is optional. The agent asks the person first whether
they want to do it, and skips this section if they do not.

Give the built skill a past forward-port in the state the bot left it. A good
run reaches the same outcome as the merged resolution, resolves each
conflict the same way or an equivalent one, and shows the ported test
failing without the port. Include one port that conflicted and one that
applied cleanly but failed its tests.