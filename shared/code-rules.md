# House code rules that matter in a fix

A digest of the house rules, limited to what a bug fix usually touches. Used
when writing the fix and again in the pre-commit check.

Two things come first:

- In a stable branch, the surrounding file's style wins over everything
  below (see `stable-rules.md`). Apply these rules to the lines you add, not
  to the lines around them.
- These rules were written against master. Some names do not exist on older
  branches (see the version traps in `odoo-context.md`).

## Python

- Imports in four groups, each sorted: stdlib, third-party, `odoo`,
  `odoo.addons.*`.
- A variable holding a record is not named `partner_id`; `_id` and `_ids`
  suffixes are for ids. `Many2one` fields end in `_id`, x2many in `_ids`.
- Method names follow the pattern of their role: `_compute_<field>`,
  `_search_<field>`, `_default_<field>`, `_onchange_<field>`,
  `_check_<name>`, `action_*`.
- New methods are private (`_` prefix) unless they must be callable from
  outside. A public name is an RPC entry point.
- Do not use `odoo.http.request` in a model. It is absent in crons, RPC,
  tests and the shell. Pass what the model needs from the controller.

## Translatable strings

Translate only static literals and pass values as arguments. Formatting
before the lookup produces a string that matches no exported term; formatting
after it bypasses placeholder validation.

```python
# good
raise UserError(self.env._("Book %s is already borrowed", book.name))
# bad: formatted before the lookup
raise UserError(self.env._("Book %s is already borrowed" % book.name))
# bad: formatted after the lookup
raise UserError(self.env._("Book %s is already borrowed") % book.name)
```

Never translate at module or class level: it runs at import with no user
language. In stable, do not reword an existing term at all.

## ORM

- `if records:` not `if len(records)`. Reading a non-relational field on
  several records raises: iterate or `mapped`. Guard a single-record
  assumption with `ensure_one()`.
- Combine domains with the branch's domain tool, never by hand-building
  `'&'` and `'|'` prefix lists.
- Context is immutable: use `with_context(...)`. A stray `default_<field>`
  key leaks into every `create` that runs underneath, including on unrelated
  models.
- `@api.depends` lists every field the compute reads. A stored compute has
  no side effects.
- `@api.onchange` is UI only. Integrity belongs in a constraint.
- `create` accepts a list of vals.
- Never `cr.commit()` or `cr.rollback()` on the request cursor.
- Catch specific exceptions over the smallest block. To recover from an
  exception inside a transaction, wrap the work in a savepoint.
- A `related` field must not pass through an x2many: the ORM silently takes
  the first record of each hop and returns wrong data.
- One inverse method shared by several fields reads siblings as `False`
  while it runs.

## Performance

- No ORM call (`search`, `search_count`, `_read_group`, `create`) inside a
  loop over records or vals. One call for the whole set, then a dict lookup.
- Conditions go in the search domain, not in `filtered()` on the result,
  unless a domain cannot express them.
- Membership tests against a `set` or with recordset arithmetic, not nested
  loops.

## XML

- Anchor view inheritance on a name, never on position. `//group[2]/field[3]`
  lands on the wrong element as soon as the parent view changes.
- A `field` locator matches the first field with that name at any depth. Use
  an xpath when the name recurs in an embedded subview.
- Do not re-add a field the parent view already renders.
- An `active` attribute on `<template>` or `<asset>` is applied at install
  only. A module update does not change it.
- `noupdate="1"` data is not touched by an update, so fixing it in the file
  fixes new databases only.
- In a record, `id` before `model`; in a field, `name` first.

## Controllers

- An override of a route is re-decorated with `@route`. It cannot change the
  parent's `type`.
- Each route has the right `auth`. A `public` route must not expose internal
  data or perform privileged writes.

## Reports

- `_get_report_values` must add `docs`, `doc_ids` and `doc_model` itself if
  the template needs them.
- `t-lang` is valid only on a node that also carries `t-call`.

## JavaScript and SCSS under static/

- Patching (`patch`) is strongly discouraged inside Odoo itself; prefer a
  proper extension point. In stable, follow what the file already does.
- Prefer a plain function, or an Owl `computed` for derived state, to a
  getter.
- A feature's files live together in one folder, not split by type.
- CSS classes are prefixed `o_<module>` and stay flat (`o_element_entry`).
  No `id` selectors.
- Every resource ships in the codebase; nothing is linked by URL.

## Comments, strings, commit messages

ASCII punctuation only: no em or en dash, no curly quotes, no ellipsis
character. They arrive by copy-paste or from a generation tool, cannot be
typed when searching, and in a translatable string create a second term.
Do not "fix" existing ones in stable.
