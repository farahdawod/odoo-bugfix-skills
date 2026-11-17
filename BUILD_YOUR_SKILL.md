# Build a skill from a basis

This file is addressed to the agent. The person you are working with wants a
skill for one stage of Odoo bug fixing, or one skill covering several
stages, built around how they work.

## Your job

The basis is raw material, not a draft of the skill. Do not turn it into a
skill directly. Unless the person chooses the defaults, interview them first,
then write a skill that encodes their workflow, using the basis for the Odoo
knowledge and the team rules.

A skill that the person did not shape is a skill they will not use. That is
the failure this repository exists to avoid. An interview so long that they
stop answering is the other failure. Aim for about ten turns for a build of
all four bases, fewer for one.

## Before the interview

Read, in this order:

1. `README.md`
2. Everything in `shared/`
3. Every basis the person named, and any file each one links under
   `basis/reference/`. If the scope question later adds a stage, read its
   basis before continuing.

Then look at what you can find out without asking: which repositories and
branches are checked out, which remotes they use and whether you can fetch
from them, what tools you can run (shell, database), how Odoo is started
here, whether there is an existing skill or instruction file of theirs. Do
not spend interview questions on things you can check yourself.

If this is not the first skill built in this conversation, also collect the
answers already given in it. See "Building several skills in one
conversation".

## First two questions

1. **Scope.** Which stages go into this skill, and which agent and
   environment will run it? If the person already named the bases, confirm
   and move on.
2. **Interview or defaults.** Ask whether they want to be asked about their
   workflow, or to build with the defaults. Show the defaults briefly: one
   short line per decision, grouped by stage, each shared decision once (see
   the decision map). Tell them they can change any default in their reply.

**If they choose the defaults:**

- Build the skill with every default, plus any changes they gave in their
  reply.
- Ask only what cannot be defaulted or detected, usually their commands and
  paths when you could not find them.
- Skip the walkthrough and the other interview rules.
- Then offer testing, which is still optional. Say that testing on their own
  tickets is how a skill built from defaults gets fitted to them, and ask for
  the material for it if they agree.

**If they choose the interview:** follow the interview rules.

## Building several skills in one conversation

The person may build one skill, then another, in the same conversation. Do
not ask again what they already answered.

1. **Before the first question of a new build, collect every answer given
   earlier in this conversation:** the agent and environment, the choice
   between interview and defaults, the environment and commands, the stop
   points, the final summary, the test-run width, the recurring corrections,
   and the decisions of every stage already covered. An answer counts
   whether it came from a question, from the walkthrough, or from a
   correction during testing.
2. **Reuse them without asking.** An answer to a shared decision carries
   over to every later build. An answer to a stage decision carries over to
   every later build that contains that stage. Do not repeat a walkthrough
   for a stage that was already walked through.
3. **Show what carries over, once, at the start of the new build.** Combine
   the scope question and the second question with one short list: "From
   earlier in this conversation: ..." with one line per answer. The person
   changes any of them in their reply; silence keeps them.
4. **Ask only what is new:** the decisions of stages not covered yet, and
   any shared decision that no earlier build needed. Use the decision map
   as usual for those.
5. **Change only what the person changes.** A change made for this build
   applies to this build. Ask whether it should also apply to the skills
   already built, and update those only if they say so.
6. **Mind answers tied to one stage.** A stop point or a final summary item
   given for one stage is not automatically one for another. Carry it to a
   new stage only when it clearly applies there too, and list it so the
   person can see it.

## Interview rules

1. **One topic per turn.** A turn asks one question, or one group from the
   decision map. Give the default with every question, so that "default" is
   a valid answer and the person can change only what differs.
2. **Walkthrough before the Decisions list:** "Take the last ticket where
   you did this. What did you do first? Then what?" Follow it step by step.
   Most decisions get answered here without being asked. A description from
   memory is enough: past tickets and fixes are optional material, so do not
   ask the person to provide them and do not make the interview depend on
   them. Use them if they are offered.
3. **Then the decisions, in the order of the decision map:** the shared
   decisions first, one per turn, then each stage's remaining decisions as
   one list per stage. Skip anything the walkthrough answered, and anything
   answered in an earlier build in this conversation.
4. **Do not accept vague answers.**
   - "It depends": ask what it depends on, and for one case on each side.
   - "Always" or "never": ask for the exception.
   - "The usual way": ask them to spell it out as if for a new colleague.
5. **Ask what goes wrong today, once for the whole skill:** "When an agent
   or a new colleague does this, what do you keep correcting?" These
   corrections are the most valuable input you will get. Ask for at least
   three. This also answers every "which review comments do you get most"
   decision. In a later build, ask only about the stages that are new.
6. **Fixed rules are not negotiable here.** If an answer conflicts with a
   Fixed rule, say so and give the rule's reason. Fixed rules are team
   policy; the person can propose a change to this repository but the skill
   does not silently drop one.
7. **Do not ask about Facts.** They are not preferences.
8. **Stop** when every decision is answered or explicitly deferred, and you
   can narrate their workflow back from input to output and they agree with
   it.

## Decision map

Decisions are referred to by basis and number: `I` investigation, `F` write
the fix, `T` regression test, `P` pre-commit check. `F2` is decision 2 of
`basis/2-write-the-fix.md`.

### Shared decisions: ask each once, for the whole skill

Ask a row only when at least one of its decisions belongs to a basis in the
skill, and it was not answered in an earlier build in this conversation.

| Ask once | Collapses | Notes |
| --- | --- | --- |
| **Order and checkpoints.** In what order do the stages run, and where do you want to approve something before the skill continues? | T2 (test before or after the fix), F1 (plan before code), the checkpoint question under "Combining bases" | Ask first in a combined build. In a single-basis build, only F1 or T2 remains. |
| **Environment.** How do you create a database, start Odoo, and run one test? | I1 (how you reproduce), T4 (run one test), P3 (temporary database) | Check the machine first; ask only what you could not find. |
| **Test-run width.** How much of the suite runs before you are satisfied? | T5, P5 | When both T and P are in the skill, the new test runs in the test stage and the wider runs in the check. |
| **Stop points.** When must the skill stop and ask, and what must it never do without asking? | I8, F5, T6 | One list for the whole skill, with the stage each item belongs to. |
| **Final summary.** What goes in it, and where? | I7, F6 | One summary at the end of the run. The pre-commit report is fixed (in the conversation) and becomes its last part. |
| **Recurring corrections.** | F7, interview rule 5 | Asked once, as interview rule 5. |

### Stage decisions: one list per stage

Show each stage's remaining decisions together, with their defaults, in one
turn. The person answers only the ones they want to change; follow up only on
those.

| Stage | Decisions |
| --- | --- |
| Investigation | I2 (where to start without a traceback), I3 (how runtime values are observed), I4 (frequent root causes), I5 (introducing commit), I6 (affected branches) |
| Write the fix | F2 (how small is smallest), F3 (comments), F4 (existing wrong data) |
| Regression test | T1 (extend or add), T3 (file without an order) |
| Pre-commit check | P1 (order of checks), P2 (what it may fix itself), P4 (edge-case budget), P6 (security depth), P7 (what blocks) |

When a decision is added to, removed from or renumbered in a basis, update
this map.

## Combining bases

The person may want one skill built from several bases, for example the fix
and the regression test together so that one run produces the whole diff.
Then:

1. **Read every basis involved first.** You cannot merge questions you have
   not read.
2. **Settle the order of the stages before anything else.** This is the
   first row of the decision map. The order of the files is only a default:
   someone who writes the test before the fix has a different skill from
   someone who writes it after.
3. **Use the decision map.** Shared decisions are asked once; follow up per
   stage only when the answer differs by stage.
4. **Drop decisions that combining answers.** A question about what a stage
   receives or hands on is settled when the neighbouring stage is in the
   same skill.
5. **One final summary, at the end.** What an earlier stage would have
   summarized (root cause, target branch, what the fix does not reach) is
   kept and passed to the next stage, and reported once when the run
   finishes. Checkpoints in between are part of the first row of the map.
   *Default:* none beyond the stop conditions.
6. **Keep every fixed rule from every basis.** State a rule once when two
   bases have it. The "Out of scope" line of a basis stops applying to
   stages that are part of the skill.
7. **Merge the facts, without duplicates,** and still include only what
   their workflow uses.
8. **Mind the size.** The main file holds the order of the stages, the stop
   points and the final summary. Each stage's detail goes in its own
   reference file, read when that stage starts.
9. **Say this once if the pre-commit check is combined with the stages it
   checks:** a check run by the same skill, in the same session that wrote
   the fix, is less independent than one run separately. The choice is the
   person's.

## After the interview

1. Write a short decision summary: each decision, their answer, and anything
   deferred. Mark which answers were carried over from an earlier build. Get
   it confirmed before writing the skill. With the defaults, the list shown
   in the second question serves as this summary.
2. Write the skill.
   - Frontmatter with `name` and `description`. The description says what the
     skill does and when to use it, in the words the person uses when they
     start this kind of work.
   - Their workflow, in their order, as steps.
   - Only the facts from the basis that their workflow uses. Leave the rest
     out; a fact the skill never applies is noise.
   - The fixed rules, each with its reason. An agent that knows why a rule
     exists applies it correctly in cases the rule did not foresee.
   - Their commands and paths, not placeholders.
   - Explicit stop points and the format of the final summary.
   - Keep the main file short. Move long reference material into separate
     files and say when to read each one.
   - No dependency on another skill being installed unless the person
     confirms they have it.
   - Before using any Odoo API or attribute in the skill text, confirm it
     exists on the branches the person works on (see the version traps in
     `shared/odoo-context.md`).
3. Offer to test it. Testing is optional: ask the person whether they want
   to try the skill on past tickets now, and skip this step if they do not.
   If they do, ask for two or three past tickets (the basis's "Test prompts"
   section says which kinds), run the skill on each from the state before
   the fix, and compare with what the person actually did. Show them the
   differences and ask which are wrong. Revise and repeat until they stop
   correcting. If tickets were already given for an earlier build in this
   conversation, offer to reuse them instead of asking for new ones.
4. Feed back. If the interview surfaced a fact, a recurring root cause, or a
   decision that the basis lacks, offer to add it to this repository.
   Preferences stay in their skill.