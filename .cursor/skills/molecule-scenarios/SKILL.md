---
name: molecule-scenarios
description: Add a molecule test scenario for a playbook or role in this repo using the shared base config and layered inventories. Use when asked to add, scaffold, or write molecule tests/scenarios.
---

# Molecule scenarios

This repo has self-contained molecule infrastructure: a shared base config
(`.config/molecule/config.yml`), shared inventory layers
(`molecule/utils/inventory/`: `base/` = localhost + platform vars, `linux/` =
`rhel10`/`rhel9` loaded by default, `windows/` = `win2022` opt-in), and
lifecycle playbooks
(`molecule/utils/provisioners/aws/`) provisioning EC2 instances (RHEL over SSH, Windows
Server over WinRM; create waits for guest connectivity at the end). A scenario
only supplies the test itself. Detailed docs:
`docs/molecule/adding-scenarios.md` and
`docs/molecule/configuration-inheritance.md`; a minimally scoped IAM user for
running the tests is documented in `docs/molecule/aws-iam-user.md`.

## Naming (mandatory)

- Playbook test: `playbook-<playbook-name-without-file-extension>`
- Role test: `role-<name-of-role>`

## Steps

1. Create `molecule/<scenario-name>/` with a `molecule.yml` containing only
   `---` (everything is inherited; the scenario name comes from the directory
   name — never set `scenario.name`).
2. Linux content needs no stack declaration — the default is `base/` +
   `linux/`. Windows content opts in by restating the full args list in
   `molecule.yml` with `--inventory=../utils/inventory/windows/` added after
   the linux line (drop the linux line for Windows-only; the list replaces
   the base config's wholesale — see `molecule/default/molecule.yml`).
3. Write `converge.yml`:
   - Playbook: `ansible.builtin.import_playbook: ../../playbooks/<name>.yml`
   - Role: a play on `hosts: molecule` with `become: true`, applying
     `roles: [role: <role_name>]` (repo roles resolve by name via
     `ANSIBLE_ROLES_PATH` from the base config). Role vars go under `vars:`.
4. Playbook scenarios: read the playbook's `hosts:` lines and map the shared
   instances into those groups with `inventory/hosts.yml`
   (`all.children.<group>.hosts.rhel10:` etc.). Group membership is additive —
   hosts stay in `molecule` for the lifecycle.
5. Write `verify.yml` asserting the content's real outcomes (services running,
   ports listening, files/config present) — not that Ansible modules work.
6. If the test harness needs prerequisites the stock AMI lacks (e.g. `acl`
   for `become_user` as an unprivileged user on RHEL 10), add a scenario
   `prepare.yml` — the base config already points `prepare:` at it, and
   molecule skips the step as "Missing playbook" when the file is absent.
   Harness plumbing only: runtime dependencies of the content under test
   belong in the role/playbook itself, or the test masks a gap.
7. Validate without provisioning:
   `molecule syntax -s <scenario-name>` and `ansible-lint molecule/<scenario-name>/`.
8. Only run `molecule test -s <scenario-name>` when platform credentials are
   available (AWS creds); it provisions real instances and
   destroys them at the end.

## Windows scenarios

Worked examples: `molecule/playbook-win-*` (five of them, one per `win-*`
playbook). Beyond the inventory opt-in in step 2:

- The `win-*` playbooks all target `hosts: all_windows` — map `win2022` into
  that group in the scenario's `inventory/hosts.yml`.
- Every `win_*` role defaults to an empty package/service list, so a converge
  on defaults is a no-op. Supply the role vars in the scenario's inventory.
- NO `become: true` — WinRM connects as Administrator already, and `become` on
  Windows is a different mechanism (`runas`).
- Verify from inside the guest (e.g. `win_uri` against `http://localhost/`) so
  assertions don't depend on the security group. Useful info modules:
  `ansible.windows.win_service_info`, `win_feature_info`, `win_stat`,
  `win_reg_stat`, `win_powershell`, `chocolatey.chocolatey.win_chocolatey_facts`.
- Content that serves traffic needs the port opened on the test security group:
  set `molecule_aws_extra_ports` (group-wide, default `[]`) in the scenario's
  `inventory/group_vars/all.yml` — `create.yml` reads it on `localhost`.

## Rules

- Do NOT edit `molecule/utils/inventory/` (shared) or `.config/molecule/config.yml`
  for scenario-specific needs — use the scenario's own `inventory/` and
  `molecule.yml`. Generic, defaulted extension points (e.g.
  `molecule_aws_extra_ports`) are the exception, not scenario-specific values.
- Do NOT copy base-config keys into a scenario `molecule.yml`; override only
  what differs. Merged lists are replaced wholesale (e.g. overriding
  `scenario.test_sequence` must state the full sequence).
- Per-host image vars (`molecule_aws_image_id` > `molecule_aws_ami_name`,
  `molecule_aws_ami_owner`, `molecule_aws_connection`,
  `molecule_aws_instance_type`) live on host entries; other `molecule_aws_*`
  values are group-wide in `molecule/utils/inventory/base/group_vars/all.yml`
  and are NOT per-host.
- A scenario with entirely different hosts restates the
  `ansible.executor.args.ansible_playbook` list without the OS layers — keep
  `../utils/inventory/base/` (platform group_vars and the explicit
  `localhost`) and declare hosts in the scenario's own `inventory/` (see
  configuration-inheritance.md).
