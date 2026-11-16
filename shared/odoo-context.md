# Odoo context shared by every basis

Facts that every stage relies on. Nothing here is a preference.

## Repositories and branches

- Standard code is split across repositories: community (`odoo`), enterprise,
  and any other repository on the addons path. One change can have a half in
  each. The halves always live on branches of the same name and are one
  change: read, test and review them together.
- A stable version is a released branch (`17.0`, `18.0`, intermediate
  `saas-*` branches). `master` is the development branch.
- A bug fix targets the oldest supported branch where the bug reproduces. It
  is then forward-ported to the newer branches by a bot on GitHub; it is not
  re-submitted per branch. So "which branches have this bug" is part of the
  investigation, not an afterthought.
- The list of supported versions changes over time. A skill should read it
  from wherever the team keeps it, or ask, rather than carry a hardcoded list.

## Version traps

The ORM, the view syntax and the test framework change between major
versions. The reference material in this repository was written against
master. Before using or flagging any API, attribute or file name, confirm it
on the target branch (`git grep` the framework source or a usage at that
revision). Never trust a remembered API.

Names known to differ between master and older branches:

| On master | On older branches |
| --- | --- |
| `security/ir.access.csv` (one `ir.access` model) | `security/ir.model.access.csv` plus `ir.rule` records |
| `models.Constraint`, `models.Index` attributes | `_sql_constraints` list |
| `odoo.fields.Domain` | `odoo.osv.expression` (`AND`, `OR`) |
| `self.env._(...)` | `_` imported from `odoo` |
| route `type='jsonrpc'` | `type='json'` |
| `@api.private` | not available, only the `_` prefix |
| `<list>` views | `<tree>` |
| `invisible="..."` expressions on nodes | `attrs=` and `states=` |
| `_compute_display_name` | `name_get` |
| `_read_group` returning tuples | `read_group` returning dicts |
| tests default to `post_install` | tests default to `at_install` |
| hoot JavaScript tests | QUnit |
| demo data off unless requested | demo data on unless disabled |
| `odoo.fields.Command` for x2many writes | legacy tuples `(0, 0, {...})`, `(4, id, 0)` |
| `TransactionCase` with a `setUpClass` that is rolled back | `SavepointCase` |
| `@odoo-module` ES modules (Owl 2) | `odoo.define(...)` (Owl 1, legacy) |
| `env.registry.clear_cache()` | `clear_caches()` and the older `ormcache` invalidation APIs |

The table says which names move, not in which version. Find the boundary on
the branch itself.

## How modules extend each other

The code that runs is assembled from every installed module. What a file
shows is rarely the whole behaviour.

- Models: a class with `_inherit = 'library.book'` extends the model in
  place. The method resolution order follows the module dependency graph, and
  `super()` goes to the modules loaded earlier. Any method can be overridden
  by a module that is not in the traceback, not in the diff, or not in the
  repository being read.
- Views: an inheriting view (`inherit_id`) modifies its parent's arch. The
  arch the user sees is the combination of all of them, applied by priority.
- Controllers: a subclass of a controller class overrides its routes.
- JavaScript: `patch()`, registries, and template inheritance (`t-inherit`)
  change behaviour defined elsewhere.
- Bridge modules (for example a module depending on both `sale` and `stock`)
  install automatically when their dependencies are present. Bugs that appear
  only with a combination of apps often live there.

Consequence: the set of installed modules is part of every bug report and of
every reproduction. "Cannot reproduce" usually means a different module set.

## Ways the same business flow is entered

The same model method is reached from several entry points, and they do not
behave the same:

- the form view: onchange calls, then one create or write;
- editable lists, kanban quick create, mass edit;
- import and data files;
- RPC from an integration;
- scheduled actions, automation rules, server actions;
- portal and website controllers (different user, often `sudo`);
- point of sale and other clients with their own data loading;
- another module's Python code.

`@api.onchange` runs only in the first. A bug, and a fix, that holds on one
path can fail on another.

## Running things

Typical commands. They differ by branch and by local setup; a skill should use
the person's own.

```
# fresh database with a module set
./odoo-bin -d <db> --addons-path=<paths> -i <modules> --stop-after-init

# update modules after a data or view change
./odoo-bin -d <db> --addons-path=<paths> -u <modules> --stop-after-init

# run selected tests ([-][tag][/module][:class][.method])
./odoo-bin -d <db> --addons-path=<paths> -i <modules> \
    --test-tags /<module>:<Class>.<method> --stop-after-init

# interactive shell (changes are rolled back unless committed)
./odoo-bin shell -d <db> --addons-path=<paths>

# remove a database
dropdb <db>
```

Python changes need a server restart. XML and data changes need a module
update. Records loaded with `noupdate="1"` are not changed by an update.