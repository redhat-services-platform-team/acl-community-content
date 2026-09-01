---
name: gpa-automation-content
description: Creates Ansible automation (roles, playbooks, collections, inventories) aligned with Red Hat COP Good Practices for Ansible (GPA). Use when writing or reviewing Ansible roles, playbooks, collections, inventories, redhat-cop automation standards, or GPA-aligned automation content.
---

# GPA automation content

Authoritative source: [redhat-cop/automation-good-practices](https://github.com/redhat-cop/automation-good-practices) (published at [redhat-cop.github.io/automation-good-practices](https://redhat-cop.github.io/automation-good-practices/)).

Treat GPA as **good** (not blind) practices — apply what fits; document conscious exceptions with rationale.

## Workflow

### 1. Pick the structure (landscape → type → function → component)

- **Landscape**: full stack deployed together (often AAP workflow or playbook-of-playbooks).
- **Type**: one playbook per host type; each host has exactly one type.
- **Function**: reusable role (e.g. base OS, app server).
- **Component**: `tasks/<component>.yml` inside a function role; promote to its own role only when a task file is too large.

Put logic in **roles**, not playbooks. Playbooks list roles (or only `import_role`/`include_role` in `tasks` — never mix `roles:` and `tasks:` role imports).

### 2. Design roles for function, not implementation

- Interface describes **what** (e.g. NTP configured), not **how** (chrony vs ntpd) unless providers diverge enough for separate roles.
- Variables: `{rolename}_*` for public API; `__{rolename}_*` for internal; prefix tags with role name; no dashes in role names.
- Multi-backend: `{rolename}_provider` with detection and `{rolename}_provider_os_default`.
- Prefer a collection entry role (e.g. `mycollection.run` with `actions:`) over exposing many `include_role` chains to consumers.

### 3. Coding style (minimum bar)

- `snake_case` identifiers; name tasks/plays/blocks; imperative task names ("Ensure …").
- YAML: 2-space indent; list items indented under key; FQCN modules (`ansible.builtin.*`); YAML-style args (not `debug: msg=…`).
- Booleans: `true` / `false`; files `.yml`; lines ≤ ~82 chars (`>-` folding, break `when:` into lists).
- Jinja: `{{ var }}`; bracket notation `item['key']`; filters over huge inline Jinja for data transforms.
- Idempotent tasks; `| bool` on bare `when:` variables; avoid `meta: end_play` (prefer `end_host`).

### 4. Validate before handoff

```bash
ansible-playbook --syntax-check path/to/playbook.yml
ansible-lint path/to/role_or_playbook
```

Use the project's `.ansible-lint` config when present.

## Review checklist

- [ ] Playbook is thin; roles hold behavior
- [ ] Type/function/component placement is intentional
- [ ] Role variables and tags prefixed; semver tags for releases
- [ ] Named tasks, FQCN, idempotency, check mode where relevant
- [ ] No secrets in examples; realistic but generic names

## Output format

Brief structure note (landscape/type/function), then files with paths.

## Additional resources

Section index and rules by topic: [reference-gpa-map.md](reference-gpa-map.md)
