# Build a skill from a basis

This file is addressed to the agent. The person you are working with wants a
skill for one stage of Odoo bug fixing, or one skill covering several
stages, built around how they work.

## Your job

The basis is raw material, not a draft of the skill. Do not turn it into a
skill directly. Interview the person first, then write a skill that encodes
their workflow, using the basis for the Odoo knowledge and the team rules.

A skill that the person did not shape is a skill they will not use. That is
the failure this repository exists to avoid.

## Before the interview

Read, in this order:

1. `README.md`
2. Everything in `shared/`
3. Every basis the person named, and any file each one links under
   `basis/reference/`. If the scope question later adds a stage, read its
   basis before continuing.

Then look at what you can find out without asking: which repositories and
branches are checked out, what tools you can run (shell, database),
whether there is an existing skill or instruction file of theirs. Do not spend
interview questions on things you can check yourself.

## Interview rules

1. Ask one question at a time and wait for the answer.
2. Start with scope: one skill for this stage, or this stage combined with
   others? Which agent and environment will run it?
3. Then ask for a walkthrough before asking anything from the Decisions list:
   "Take the last ticket where you did this. What did you do first? Then
   what?" Follow it step by step. Most decisions get answered here without
   being asked. A description from memory is enough: past tickets and fixes
   are optional material, so do not ask the person to provide them and do
   not make the interview depend on them. Use them if they are offered.
4. Go through the remaining Decisions in the basis. State the default with
   each question so that "default" is a valid answer.
5. Do not accept vague answers.
   - "It depends": ask what it depends on, and for one case on each side.
   - "Always" or "never": ask for the exception.
   - "The usual way": ask them to spell it out as if for a new colleague.
6. Ask what goes wrong today: "When an agent or a new colleague does this
   stage, what do you keep correcting?" These corrections are the most
   valuable input you will get. Ask for at least three.
7. Ask where the skill must stop and ask: what it must never do without
   asking, and what the final summary of a finished run should contain.
8. If an answer conflicts with a Fixed rule, say so and give the rule's
   reason. Fixed rules are team policy; the person can propose a change to
   this repository but the skill does not silently drop one.
9. Do not ask about Facts. They are not preferences.
10. Stop when every decision is answered or explicitly deferred, and you can
    narrate their workflow back from input to output and they agree with it.

## Combining bases

The person may want one skill built from several bases, for example the fix
and the regression test together so that one run produces the whole diff.
Then:

1. **Read every basis involved first.** You cannot merge questions you have
   not read.
2. **Settle the order of the stages before anything else.** Which stage runs
   when, and are they interleaved? The order of the files is only a default:
   someone who writes the test before the fix has a different skill from
   someone who writes it after. Decisions inside a basis that are about order
   (such as "test before or after the fix") are asked here, first.
3. **Ask shared decisions once.** Several bases ask the same thing: what the
   final summary contains, when to stop and ask, which review comments the
   person gets most, how they run Odoo and tests. Ask once for the whole
   skill, and follow up per stage only when the answer differs by stage.
4. **Drop decisions that combining answers.** A question about what a stage
   receives or hands on is settled when the neighbouring stage is in the
   same skill.
5. **One final summary, at the end.** What an earlier stage would have
   summarized (root cause, target branch, what the fix does not reach) is
   kept and passed to the next stage, and reported once when the run
   finishes. Ask where the person wants a checkpoint in between, for example
   to approve the root cause before any code is written. *Default:* none
   beyond the stop conditions.
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
   deferred. Get it confirmed before writing the skill.
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
   correcting.
4. Feed back. If the interview surfaced a fact, a recurring root cause, or a
   decision that the basis lacks, offer to add it to this repository.
   Preferences stay in their skill.