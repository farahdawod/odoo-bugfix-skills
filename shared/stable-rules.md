# Changes in a stable version

Fixed rules. Every stage of a bug fix in a released branch works inside them.

The principle: a stable branch runs in production for thousands of databases
that update by pulling code, often without a module update, with custom and
third-party modules installed on top. A fix must not break any of them. Only
bug fixes and localizations go to stable; improvements and features go to
master.

| Rule | Why |
| --- | --- |
| Keep the change minimal and strictly about the bug. If the risk is high or the value low, it belongs in master. | Every changed line is a line that can break someone's database or someone's override. |
| No improvements and no cosmetic changes (formatting, PEP 8, renaming, restyling). | They add risk and conflicts on forward-port with no benefit to the user who reported the bug. |
| Match the surrounding code's style, even where it breaks the current guidelines. | The existing file style outranks the guidelines in stable. |
| Never change the signature of a public model method (no `_` prefix). Avoid changing private signatures too. | Public methods are API for integrators. Private ones are overridden by extension modules, which break on a new signature. |
| No data-model change: no stored column added, removed, or changed to an incompatible type. Compatible tweaks (`ondelete`, `size`) are allowed when needed. A non-stored computed field may be added if really necessary. | A database that pulls the code without updating has no such column and crashes. |
| Keep XML ids of existing data. Do not delete data records that user data may reference, unless essential and the records were `noupdate` from the start. | Customer records, customizations and other modules point at them. |
| Change XML (views, menus, default data) only when inevitable, and then the Python code must not depend on the change. | A database that has not been updated still has the old XML and must keep working. |
| A fix may need a module update only if users who do not run it are safe. A critical security fix must work with a pull and a restart alone. | Not everyone updates modules after pulling. |
| Do not modify existing translatable source terms, even to fix a typo. Adding or removing terms is fine. Do not ship `.pot` updates. | Translations are keyed on the exact source string; changing it drops every existing translation. Correct the term in master, or add an English-to-English translation in stable. |

When a correct fix cannot be made inside these rules, that is a finding to
report, not a rule to bend: say what the proper fix would be, why stable does
not allow it, and what the smaller stable-safe alternative is.
