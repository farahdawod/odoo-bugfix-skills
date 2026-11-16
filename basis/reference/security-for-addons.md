# Security for addon code

Security material for someone who works only on Odoo addons, used by the
pre-commit check. It combines the Odoo-specific rules with the parts of a
general security audit that apply to addon code. Everything about memory
safety, binaries, cloud deployment, desktop and mobile apps, supply chain,
messaging protocols and identity providers was left out: an addon fix does
not touch them. Add a section back if your area does (for example AI
features or device integrations).

Access-control file names and several APIs here are master's. See the version
traps in `shared/odoo-context.md`.

## How to judge a hit

A pattern match is not a finding. A finding names:

- a lower-trust actor: a public visitor, a portal user, an internal user
  without the group, a user of another company;
- what they send or do;
- the control that should have stopped them;
- what they get: data they may not read, a write they may not make, code
  that runs as someone else.

Without a concrete actor and result, it is a hardening note at most. With
them, it blocks the commit.

## What a bug fix most often gets wrong

A fix rarely introduces SQL injection. It usually does one of these, because
each of them makes an error go away:

| The fix adds | The question to ask |
| --- | --- |
| A `sudo()` to silence an `AccessError` | Was the access legitimately denied? What can the caller now choose (record, field, values, domain)? |
| A wider domain or a removed condition | Which records does the user now see or write that they did not before? |
| A public method, or a `_` removed | It is now callable over RPC with any arguments on any records. |
| A change to a route's `auth`, `methods` or `csrf` | Who can now reach it, and does it write? |
| A looser access row or record rule | For which groups, which operations, which records? |
| `Markup(...)` or an HTML field rendered raw to fix escaped output | Is every dynamic part escaped before it is marked safe? |
| A value from the request passed straight to `write`, `create`, `search` or `browse` | Which keys, ids and operators can the caller choose? |
| `with_user`, `with_company`, a context key that disables a check | Who decides the user, the company or the key? |

The right fix for an access error is the minimum access, in the narrowest
scope, with a comment saying why. A silent widening is a finding.

## Sweep the diff

Sweep the changed lines and the methods they sit in. Judge each hit against
its section.

| Sweep for | Section |
| --- | --- |
| `sudo(`, `with_user(`, `with_company(` | Do not over-sudo |
| `cr.execute`, `SQL(`, string formatting near SQL | Use the ORM; parameterize SQL |
| A domain built from request, RPC or stored values | Domain injection |
| Public (non-`_`) model methods | Default to private methods |
| `@route(` | Routes: auth, POST, CSRF |
| `Markup(`, `markup(`, `innerHTML`, `insertAdjacentHTML`, `t-raw` | Prevent XSS |
| Password, token, secret fields; `related=` across models | Field-level access |
| `open(` | Files |
| `eval(`, `exec(`, `safe_eval` | Evaluation |
| `pickle` | Never pickle |
| `getattr(`, `setattr(` on records | Dynamic attribute access |
| `==` on a secret or token | Timing |
| A mutable default argument | Mutable defaults |
| A URL from user input that the server fetches or redirects to | Requests and redirects |

## Access control

- A permission row grants exactly the operations it lists. A restriction
  that covers only read leaves write governed by the permission rows alone:
  `write` checks write access, never read, so a user can alter records the
  read restriction hides. Restrict every operation that needs it.
- How record rules combine, on branches that use `ir.rule`. Global rules
  (no `groups`) are combined with `AND`: every one of them applies to every
  user. Group rules are combined with `OR` among themselves: a user gets the
  union of what the rules of their groups allow. The result is then combined
  with `AND` with the global rules. (On master, the same split exists
  between restriction rows without a group and permission rows with one.)
- Two errors a bug fix makes with that:
  - adding a group rule to restrict users who already pass another group
    rule. It restricts nothing: group rules only widen each other. A user in
    both groups keeps the broader access. A restriction that must hold for
    everyone is a global rule;
  - forgetting `groups` on a rule meant for one role. It becomes global and
    is applied to every user, which locks unrelated roles out of records
    they could reach before.
- Flag create, write or delete granted to everyone, to portal or to public.
  Look twice at read on models with personal or business-sensitive data.
- Multi-company models need a company restriction. Sensitive models need
  record-level restrictions, not only group permissions.
- Access control is enforced on create, read, write and delete. It is not
  enforced on method calls. A method that acts on `self` without checking
  acts on whatever records the caller passed.
- The same state change is often reachable several ways (the button, a
  direct `write` on the state field, an import, a batch action, a portal
  route). The effective rule is the weakest of them. If the fix adds a check
  on one path, look for the others.
- A value in the request must not be able to override what the permission
  model decided: the company, the owner, the partner of a portal user.
- Batch, export and import paths must enforce per-record access, not only
  access to the action.

## Field-level access

- A field's `groups` removes it from views and `fields_get` and raises on
  explicit read or write. Use it for sensitive fields; hiding in the view is
  not protection.
- Passwords and API tokens: system group only, or no access at all. A token
  field on a model with a permissive access row is readable by any user
  through `search_read`.
- `related` fields are computed as superuser by default. A related field
  reaching sensitive data through a `Many2one` the user can write lets them
  point it at any record and read the value. Set `related_sudo=False` on
  sensitive chains.
- Searching, sorting or grouping on a field the user cannot read can reveal
  its values through which records match or in what order. A custom search
  method or a `sudo()` search can reopen what the field's `groups` closed.

## Default to private methods

- Any public method is callable over RPC with attacker-chosen arguments on
  attacker-chosen records. Prefix with `_` unless the method must be called
  from outside, and then validate its inputs.
- Privacy alone is not a control: a `_` method fed untrusted data by a
  public one is still dangerous.
- The RPC layer blocks `_`-prefixed names and, on branches that have it,
  methods decorated `@api.private`. Check before flagging a public name as
  exposed.

## Use the ORM; parameterize SQL

- Raw SQL bypasses access control, translations, field invalidation and
  `active` handling. Use `search`, `_read_group` and field access.
- When SQL is necessary, never build it with `+`, `%` or `.format`. Pass
  values as parameters, or use the SQL wrapper the branch provides. Table
  and column names cannot be parameters: use the identifier helper, and
  never feed it raw user input.
- Flush before SQL that reads what the transaction wrote; invalidate after
  SQL that writes.

## Domain injection

- Never concatenate a user-provided domain onto a security domain. A user
  can inject an `'|'` and widen it. Combine with the branch's domain tool.

## Do not over-sudo

- For each `sudo()`, confirm the caller does not control: which model,
  record or field is read; which model or values are created; which record
  or values are written; which model or domain is searched.
- In a controller, never `record.sudo().write(post)` with raw parameters.
  Whitelist the keys.
- Under `sudo()`, x2many commands inside the values run as superuser on the
  comodel. A sudo write with raw values carries the privilege into other
  models. Whitelist command lists too.
- `sudo()` also drops record rules, including the company ones. A sudo read
  can return another company's records.
- Narrow the scope: sudo the one read that needs it, then go back to the
  user's environment for the rest.
- `with_user` and `with_company` must be chosen by the code, not by the
  caller.

## Routes: auth, POST, CSRF

- Match `auth` to exposure. A `public` or `none` route must not expose
  internal data or perform privileged writes.
- A route that writes uses `methods=['POST']`. On an HTTP-type route keep
  CSRF on, except for dedicated webhooks. State-changing logic on a GET
  route can be triggered by a link or a hidden form in the victim's session.
- Templates that POST include the CSRF token input.
- A token in the URL (`access_token`) must be bound to the one record it was
  issued for, compared safely, and checked on every route that serves the
  record, including downloads and sub-pages.
- Different answers for "does not exist" and "no access" let a visitor
  enumerate records. Treat that as a finding only when existence itself is
  confidential.

## Prevent XSS

- Reflected and stored script both run in the victim's session. Stored is
  the common one in Odoo: a low-privilege user saves a value, an admin opens
  the record.
- Server and QWeb: render with `t-out`, which escapes. The raw-HTML vector
  is a `Markup` value reaching `t-out`. Build HTML by wrapping literals in
  `Markup` and formatting user content into them, which escapes it.
  `Markup(f"<p>{x}</p>")` defeats escaping; use
  `Markup("<p>{x}</p>").format(x=...)`.
- `t-raw` no longer exists on recent branches; an occurrence there is
  broken legacy code.
- JavaScript: the sinks are `innerHTML`, `insertAdjacentHTML` and `markup()`
  on a non-literal. Never feed them user or low-privilege strings.
- Escaping is always required when mixing data into code, even for trusted
  data. Sanitizing is for untrusted HTML only and happens after escaping.
- Second-order use: a value safe when stored becomes dangerous where it is
  used later. A name becomes a file name or a path; text enters an email
  template or a report; a stored string becomes a URL, a domain or a regex.

## Files

- Never use the builtin `open()` on a path that can be influenced. Use the
  framework's `file_open`, which confines access to the addons paths. It is
  a path confinement, not a write protection.
- Attachments: access follows the record they are attached to. Code that
  reads attachments as superuser by id serves any file to anyone who can
  guess or choose the id.

## Evaluation

- Never `eval` or `exec`. Parse data with `json.loads` or
  `ast.literal_eval`. `safe_eval` only with a constrained namespace and
  only for input from trusted privileged users.
- A public model method that returns a rich object (a key, a handle, a
  connection) is reachable from server actions and automation rules, which
  can walk its attributes. Keep such logic in a module-level function.

## Smaller items

- **Never pickle.** Loading a pickle runs code. Use JSON.
- **Dynamic attribute access.** `getattr` and `setattr` on a record with a
  name from outside expose private attributes and methods. Use
  `record[name]` and validate the name.
- **Timing.** Compare secrets in constant time (`consteq`), not `==`. Better,
  search the database for the token.
- **Randomness.** Tokens come from a cryptographic source, not from `random`
  or from ids and dates.
- **Secrets in output.** No token, password or key in a log line, an error
  message, a URL or a chatter message.
- **Mutable defaults.** `def f(x, vals=[])` keeps state across calls and can
  leak data between users.

## Requests and redirects

- A URL the user sets that the server fetches (a webhook, an image by URL,
  a feed) can be pointed at internal services. Validate the destination,
  including after redirects.
- A `redirect` parameter fed into a redirect without checking it is local
  sends a logged-in user to an attacker's page.

## Data isolation between companies and users

- A `company_id` on a record is not isolation. The rule, the domain or the
  check that enforces it is, on each read and write path.
- `check_company=True` on a relational field, enforced by
  `_check_company()`, verifies on create and write that the linked record
  belongs to a company compatible with the record's own. It is a consistency
  check, not access control: it stops a record of company A from pointing at
  a record of company B, and it does not stop anyone from reading either.
  A fix that adds a relational field to a multi-company model, or writes one
  under `sudo()` or from a controller, should keep that check in force and
  not bypass it. Confirm on the branch how the model enables the automatic
  check.
- A cache keyed too broadly serves one user's or one company's result to
  another. Every value the result depends on (user, company, language,
  website) belongs in the key of an `ormcache`.
- Follow the copies: a value restricted on its model can leak through a
  report, an export, a mail, a chatter message, a search view, a dashboard
  or a stored related field with broader access.
- Archiving or revoking access must hold on every path: a direct URL, a
  portal link, a search, a scheduled job that still acts on the record.

## Business logic

For the flow the fix touches:

- Can a step be skipped, repeated or done in the wrong order by writing the
  state field or calling the method directly?
- Check-then-act without a lock: two users, or a user and a cron, both pass
  the check and both act (double validation, double payment, stock taken
  twice).
- Numbers: negative, zero, very large, a string where a number is expected,
  a quantity in another unit, an amount in another currency.
- Time: the exact boundary, the user's timezone against the server's, a
  date in a locked period.
- Defaults: what happens when the setting is missing, the feature is off,
  or the optional module is not installed.

## Expensive work a low-trust user can trigger

A finding only when someone with little access can cost the shared server a
lot:

- a public or portal route that runs a search with no limit, or with a limit
  and domain taken from the request;
- a list of ids or a batch size taken from the request with no bound;
- a regular expression or a domain built from user text;
- a loop that sends mail or calls an external service once per item the
  user supplied.