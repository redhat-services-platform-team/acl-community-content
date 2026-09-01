# GPA section map

Canonical repo: https://github.com/redhat-cop/automation-good-practices
Site: https://redhat-cop.github.io/automation-good-practices/

## Structures (`structures/README.adoc`)

- **Zen of Ansible**: declarative over imperative; readability; convention over configuration; avoid "magic" manual steps; friction elimination.
- **Hierarchy**: landscape → type (one playbook per type) → function (role) → component (`tasks/*.yml`).
- Exceptions are OK when discussed; external collections may bend local rules.

## Playbooks (`playbooks/README.adoc`)

- Playbooks are mostly role lists; complex logic belongs in roles.
- Use **either** `roles:` **or** `tasks:` with `import_role`/`include_role`, not both.
- Tags: per-role name and/or one tag per meaningful purpose; never tags that only work in sequence; document tags.
- `debug`: set `verbosity:` so production runs stay quiet.

## Roles (`roles/README.adoc`)

- Design for **functionality** (outcome), not a single software product unless providers diverge.
- **Distribution**: semver Git tags; `0.y.z` until interface stable.
- **Naming**: `{role}_` on parameters/defaults; `__{role}_` internal; `{role}_` on modules in role; no dashes in role name.
- **Providers**: `{role}_provider`, detect running provider, `{role}_provider_os_default` for OS defaults.
- **Coupling**: loose coupling; shared bits in a `common` (or similar) role inside collections.
- **Entry role pattern**: `mycollection.run` with `actions:` list hides internal role graph from consumers.

## Collections (`collections/README.adoc`)

- Package related roles/modules; namespace-aware naming; align with Ansible Galaxy / Automation Hub versioning rules.

## Inventories (`inventories/README.adoc`)

- Keep inventory data separate from playbooks; structure for AAP/Controller; avoid embedding secrets in git.

## Plugins (`plugins/README.adoc`)

- Prefer filter/lookup plugins over heavy Jinja for structured data manipulation.

## Coding style (`coding_style/README.adoc`)

High-signal items agents often miss:

| Area | Rule |
|------|------|
| Names | Valid Python identifiers, `snake_case`, mnemonic, `object[_feature]_action` pattern |
| YAML | 2 spaces; indent list items; `>-` not `>` unless trailing newline matters |
| Tasks | FQCN; YAML dict args; named tasks; imperative names |
| Booleans | `true`/`false` only in playbooks |
| Files | `.yml` extension |
| Jinja | Space inside `{{ }}`; single quotes inside Jinja strings |
| Safety | Idempotency; check mode; `changed_when` on command/shell |
| Variables | Bracket access; lazy evaluation awareness |
| Anti-patterns | `meta: end_play`; excessive comments instead of task names |

## Linting

Align with the project's `.ansible-lint` profile. GPA examples generally follow ansible-lint defaults (e.g. ~82 char lines, FQCN).
