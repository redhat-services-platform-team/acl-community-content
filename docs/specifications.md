# ACL Community Content — Specifications

Architectural and operational decisions governing the `acl-community-content`
repository. This document is the source of truth for contributors and
maintainers.

The **engineering bar matches**
[`acl-ready-to-use-content`](https://github.com/redhat-services-platform-team/acl-ready-to-use-content)
(GPA, ansible-lint `production`, catalog headers, AAP 2.6+ testing). Scope is
broader than sales-motion starter packs; it is **not** a lower-quality inbox.

---

## 1. Repository Classification

| Decision | Value | Rationale |
|----------|-------|-----------|
| ACL category | **Community** | Public contributions; SPT maintains merge rights |
| Repository name | `acl-community-content` | Follows ACL naming: `acl-<content-type>` |
| Visibility | **Public** | Forks, Apache-2.0, DCO; AAP can sync without an SCM credential |
| Catalog scope | Use Case Maturity Catalog (any cell that is honestly classified) | Not limited to `maturity:l2` / day1–day2 the way ready-to-use is |
| Not this repo | Sales-tied starter packs | Those live in `acl-ready-to-use-content` (and ADUC repos) |

### Hub validated / certified promotion

**Out of scope.** Merging here does **not** make content Automation Hub
validated or certified. A future promotion path may be documented in this
section; it is not designed or implemented yet.

---

## 2. Repository Layout

```
acl-community-content/
├── playbooks/                 # Flat .yml files — no subdirectories
├── roles/                     # All roles hoisted to repo root
├── collections/               # requirements.yml for runtime collections
├── execution_environments/    # ansible-builder manifest (optional)
├── controller_config/         # AAP Configuration as Code (grows with content)
├── inventory/                 # Example inventory + group_vars
├── docs/                      # This spec and getting-started
└── ansible.cfg                # roles_path = roles
```

Key layout rules:

- **No subdirectories under `playbooks/`.** Each playbook is a single flat
  `.yml` file named with a platform prefix.
- **All roles live under `roles/`** and are discovered via `ansible.cfg`
  (`roles_path = roles`).
- **Scenario-specific variables** go into `roles/<role>/defaults/main.yml`,
  not into `group_vars/` outside the role.
- **Role documentation** (`README.md`) lives inside each role directory.
- **Repository-wide settings** live in `inventory/group_vars/all/`.

---

## 3. Naming Conventions

| Element | Convention | Examples |
|---------|-----------|----------|
| Playbook files | `<platform>-<use-case>.yml` | `linux-dns-primary.yml`, `win-service-management.yml` |
| Platform prefixes | `linux-` for RHEL/Linux, `win-` for Windows | — |
| Role directories | `snake_case`, no platform prefix unless Windows-only | `dns_sync`, `win_service_management` |
| AAP job template names | `ACL_<platform>_<use_case>` | `ACL_linux_example` |
| AAP credential names | `acl_<purpose>` | `acl_machine_credential` |
| AAP inventory name | `acl_lab_inventory` (default, overridable) | — |
| Role public variables | `{rolename}_*` prefix per GPA | `selinux_config_state` |
| Role internal variables | `__{rolename}_*` (double underscore) per GPA | `__web_service_rca_profile` |

---

## 4. Playbook Design

- **Thin entry points.** Playbooks contain minimal logic: host targeting,
  `become`, `gather_facts`, an optional OS assert in `pre_tasks`, and a
  `roles:` list. All task logic lives in the role.
- **Static host groups.** Playbooks use hardcoded `hosts:` values
  (`all_rhel`, `all_windows`, or a service-specific group). Operators narrow
  scope at launch time using AAP's **limit** field. No Jinja2 in `hosts:`.
- **No `import_playbook` across scenarios.** Each playbook is self-contained
  (it may call shared utility roles).

### Playbook file header (comment block)

Every file under `playbooks/` must begin with a **YAML comment header** placed
**before** the document start marker (`---`). The header is catalog metadata.
It does **not** replace role `README.md` and is **not** Ansible runtime
`tags:` on tasks.

#### Use Case Maturity Catalog

Each artifact carries **three required classification tags** (prefixed), plus
optional extras:

| Axis | Tag prefix | Required | Meaning |
|------|------------|:--------:|---------|
| **Pillar** | `pillar:` | Yes | Technology domain |
| **Lifecycle** | `lifecycle:` | Yes | Day 0 / Day 1 / Day 2 stage |
| **Maturity** | `maturity:` | Yes | Automation maturity level (L1–L4) |

Optional extras (same `# Tags:` line): `platform:`, `tech:`, `context:`.

**Required header fields**

| Field | Required | Notes |
|-------|:--------:|-------|
| Playbook title | Yes | Short human title (banner line) |
| `Description:` | Yes | 1–3 lines: purpose and notable behaviour |
| `Tags:` | Yes | Prefixed tags; must include exactly one `pillar:`, `lifecycle:`, `maturity:` |
| `Supported OS:` | When restricted | Include when a `pre_task` assert restricts OS; omit otherwise |
| `Author:` | Yes | Contributor name (maintainers may use `Ansible Services Platform Team, Red Hat`) |
| `Version:` | Yes | Semantic content version; start at `1.0.0` |
| `Date:` | Yes | `YYYY-MM-DD` — header added or last meaningful content change |
| `License:` | Yes | `Apache-2.0` unless a bundled role specifies otherwise |

**Canonical template**

```yaml
# ============================================================================
# Playbook: <Human title>
# ============================================================================
#
# Description:
#   <What this playbook does; key roles or side effects in 1–3 lines.>
#
# Tags: pillar:<pillar>,lifecycle:<stage>,maturity:<level>,platform:<linux|windows>
#
# Supported OS:
#   <e.g. RHEL 8/9 — only when a pre_task assert restricts OS>
#
# Author: <Contributor Name>
# Version: 1.0.0
# Date: YYYY-MM-DD
# License: Apache-2.0
#
# ============================================================================
---
```

**Controlled vocabulary**

Tags are lowercase, comma-separated **without** spaces. New values require a
spec update (PR).

| Prefix | Allowed values |
|--------|----------------|
| `pillar:` | `os`, `cloud`, `network`, `db`, `security`, `ops` |
| `lifecycle:` | `day0`, `day1`, `day2` |
| `maturity:` | `l1` (task focus), `l2` (process / AAP-Ansible), `l3` (event driven), `l4` (AI driven) |
| `platform:` (optional) | `linux`, `windows` |
| `tech:` (optional) | Add via spec PR when a contribution needs a new token |
| `context:` (optional) | `localhost`, `multi-play`, `rhel9-only` |

**Classification rules**

- Assign **one** `pillar:`, **one** `lifecycle:`, and **one** `maturity:` per
  playbook (primary classification). Use `Description:` for nuance.
- Tag honestly. Community content is **not** required to be `maturity:l2`.
- **Do not mirror header tags into Ansible runtime tags.** Header tags exist
  for catalog alignment. Existing runtime tags (`always`, role tags) stay as-is.

---

## 5. Role Design

- **Self-contained.** Each role includes `tasks/`, `defaults/main.yml`,
  `handlers/` (if needed), `templates/`, `files/`, and `README.md`.
- **GPA variable naming.** Follow
  [Red Hat COP Good Practices for Ansible](https://redhat-cop.github.io/automation-good-practices/):
  - **Public API:** `{rolename}_*`
  - **Internal / private:** `__{rolename}_*` (double underscore)

### Role README metadata

Use the **same catalog classification** as playbook headers, as **Markdown**
immediately under the `# Role:` title.

| Line | Required | Notes |
|------|:--------:|-------|
| `**Tags:**` | Yes | Prefixed tags; must include `pillar:`, `lifecycle:`, `maturity:` |
| `**Supported OS:**` | When restricted | Same rule as playbooks |
| Body | Yes | GPA sections (Requirements, Role Variables, Example Playbook, etc.) |

**Example**

```markdown
# Role: `example_role`

**Tags:** pillar:os,lifecycle:day1,maturity:l2,platform:linux
**Supported OS:** RHEL 8/9

Configure …
```

Playbook and role tags may differ slightly: the playbook header describes the
**entry point**; the role README describes **role behaviour**.

---

## 6. Quality gates

| Gate | Bar |
|------|-----|
| ansible-lint | Profile `production` (CI + pre-commit) |
| Secrets | gitleaks (CI + pre-commit) |
| Runtime test | Documented AAP 2.6+ test **or** a Molecule scenario in the PR |
| DCO | Every commit `Signed-off-by` (`git commit -s`) |
| License | Apache-2.0; no secrets or internal-only URLs |

Molecule with cloud provisioners is **not** required on day one. Lint is the
merge gate; AAP/Molecule evidence is a reviewer checklist item.

---

## 7. AAP Configuration as Code (CaC)

CaC YAML will grow as playbooks land. The bootstrap ships
`controller_config/site_config.yml.example` and a README only — no job
templates until there is content to launch.

| Variable | Required | Description |
|----------|:--------:|-------------|
| `aap_hostname` | Yes | AAP controller URL |
| `aap_validate_certs` | No | SSL verification; defaults to `true` |
| `aap_token` | Yes | PAT; pass at runtime via `-e` (never commit) |
| `acl_project_scm_url` | Yes | Git URL of this repository (public HTTPS by default) |
| `acl_project_scm_branch` | No | Branch to track; defaults to `main` |
| `acl_project_scm_credential` | No | **Empty for this public repo** |
| `acl_linux_credential` | Yes | Existing Machine credential for Linux hosts |
| `acl_windows_credential` | Yes | Existing Machine credential for Windows hosts |
| `acl_organization` | No | Defaults to `Default` |
| `acl_inventory_name` | No | Defaults to `acl_lab_inventory` |

Credentials are **not** created by CaC. Users pre-create machine credentials
in AAP.

---

## 8. Execution Environment

| Decision | Value |
|----------|-------|
| EE for job templates | `Default execution environment` unless a playbook needs more |
| Custom EE | `execution_environments/` manifest; optional |

Keep `execution_environments/requirements.yml` aligned with
`collections/requirements.yml`. Contributors add collections in **both**
places when a playbook needs them.

---

## 9. Collections

Runtime collections are declared in `collections/requirements.yml`. The
bootstrap list is a common baseline (`ansible.posix`, `ansible.windows`,
`community.general`). Add collections via PR when content requires them.
Keep version ranges compatible with AAP 2.6+.
