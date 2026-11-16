# Odoo bug-fix skill basis

Raw material for building your own agent skills for bug fixing in Odoo's
standard code.

This repository does not contain skills. It contains a basis for each stage of
a bug fix. You hand a basis to your agent, the agent interviews you about how
you actually work, and it writes a skill that matches your workflow. Two people
starting from the same basis should end up with different skills.

## How to use it

1. Pick the stage you want a skill for (see the table), or several stages
   for one combined skill, for example the fix and the regression test
   together.
2. Open a session with your agent in this repository and say:
   "Read BUILD_YOUR_SKILL.md and build my skill from basis/<file>."
3. Answer the interview. Bring one or two tickets you fixed recently; the
   agent will ask for them.
4. Optionally, try the skill on a past ticket, correct it, repeat. The
   agent asks whether you want to.

Your finished skill lives wherever your agent loads skills from, not here.

## What is in a basis

Every basis has the same five parts, and they are not equal:

| Part | What it is | Who owns it |
| --- | --- | --- |
| Facts | How Odoo behaves. True whatever your workflow is. | The repository |
| Fixed rules | Team and Odoo policy, each with its reason. | The repository |
| Decisions | Open questions about how you work, each with a default. | You |
| Your material | Commands, corrections you keep making. Past tickets and fixes are optional. | You |
| Test prompts | Optional. How to check the skill once it is built. | You |

The agent copies the facts it needs, keeps the fixed rules, and asks you the
decisions. The decisions are the point: a skill built without them is just a
generic skill.

## Contents

| File | Stage |
| --- | --- |
| [basis/1-investigation.md](basis/1-investigation.md) | Read the traceback, find the root cause, optionally find the introducing commit |
| [basis/2-write-the-fix.md](basis/2-write-the-fix.md) | Smallest change in the right module, within stable rules |
| [basis/3-regression-test.md](basis/3-regression-test.md) | Reuse or write a test, place it, tag it |
| [basis/4-pre-commit-check.md](basis/4-pre-commit-check.md) | Rules, security, edge cases on a temporary database, tests fail without the fix |
| [basis/reference/security-for-addons.md](basis/reference/security-for-addons.md) | Security material for addon code, used by stage 4 |
| [shared/odoo-context.md](shared/odoo-context.md) | Repositories, branches, version traps, how modules extend each other |
| [shared/stable-rules.md](shared/stable-rules.md) | What may and may not change in a released branch |
| [shared/code-rules.md](shared/code-rules.md) | House code rules that matter when writing a fix |
| [BUILD_YOUR_SKILL.md](BUILD_YOUR_SKILL.md) | Instructions the agent follows to interview you and write the skill |
| [basis/_template.md](basis/_template.md) | Template for adding a basis |
