# Molecule configuration inheritance

How a scenario's effective configuration is assembled from the shared base
config and the layered inventories. Read this before writing a scenario that
needs to override anything; the short version for the common case is in
[molecule/README.md](../../molecule/README.md).

## Configuration: base config + scenario molecule.yml

Molecule auto-discovers `.config/molecule/config.yml` at the repository root
and deep-merges each scenario's `molecule.yml` on top of it:

```text
molecule defaults  <-  .config/molecule/config.yml  <-  molecule/<scenario>/molecule.yml
                       (shared base config)             (scenario overrides, wins)
```

Merge semantics (same algorithm as Ansible's `combine(recursive=True)`):

- **Dicts merge recursively** — a scenario can override one nested key without
  restating its siblings.
- **Everything else is replaced wholesale** — lists in particular. Overriding
  `scenario.test_sequence` or `ansible.executor.args.ansible_playbook`
  replaces the *entire* list, never appends to it.
- **Environment interpolation** (`${VAR}`, `${VAR:-default}`) is applied to
  both files before parsing, so environment references may be used in either
  file.

The shared base config wires everything a scenario normally needs:

```yaml
# .config/molecule/config.yml (abridged)
ansible:
  env:
    ANSIBLE_ROLES_PATH: ../../roles          # repo roles resolve by name
  executor:
    args:
      ansible_playbook:
        - --inventory=../utils/inventory/base/     # localhost + platform group_vars
        - --inventory=../utils/inventory/linux/    # rhel10, rhel9
        - --inventory=inventory/             # scenario-local overlay (optional)
  playbooks:
    create: ../utils/provisioners/aws/create.yml
    destroy: ../utils/provisioners/aws/destroy.yml
    prepare: prepare.yml   # optional scenario-local hook, skipped when absent
    converge: converge.yml
    verify: verify.yml

scenario:
  test_sequence: [dependency, syntax, create, prepare, converge, verify, destroy]

verifier:
  name: ansible
```

Relative paths resolve from the scenario directory (`molecule/<scenario>/`),
so the same base config works for every scenario. The scenario name is derived
from the directory name — never set `scenario.name`.

### molecule.yml examples

The normal case — inherit everything:

```yaml
---
# All configuration comes from .config/molecule/config.yml.
```

Add the idempotence step (list replacement: state the full sequence):

```yaml
---
scenario:
  test_sequence:
    - dependency
    - syntax
    - create
    - prepare
    - converge
    - idempotence
    - verify
    - destroy
```

Opt in to Windows by restating the stack with the windows layer — the utils
provision exactly the hosts the loaded inventories declare. Keep the linux
line for a mixed fleet; drop it for Windows-only:

```yaml
---
ansible:
  executor:
    args:
      ansible_playbook:
        - --inventory=../utils/inventory/base/
        - --inventory=../utils/inventory/linux/
        - --inventory=../utils/inventory/windows/
        - --inventory=inventory/
```

A scenario whose hosts differ entirely from the shared ones drops the OS
layers and ships its own instead — keep `../utils/inventory/base/` (platform
group_vars and the explicit `localhost` entry) and declare hosts in the
scenario's own `inventory/`.

## Inventory: layered sources

Ansible merges the `--inventory` sources in order; **later sources win**
for conflicting vars, and hosts/groups are matched by name (layering can add
hosts and vars, but never remove a host):

| Layer | Path | Contents |
| ----- | ---- | -------- |
| 1. Base | `molecule/utils/inventory/base/` | Explicit `localhost`; `group_vars/all.yml` with platform defaults (`molecule_aws_*`). Always loaded |
| 2. OS | `molecule/utils/inventory/linux/`, `molecule/utils/inventory/windows/` | The `molecule` group's test instances with per-host image/connection identity. `linux/` is loaded by default; `windows/` is opt-in via the scenario's `molecule.yml` |
| 3. Scenario | `molecule/<scenario>/inventory/` | Optional. Extra groups/vars for this scenario's converge/verify, or overrides of earlier layers |
| 4. Runtime | `$MOLECULE_EPHEMERAL_DIRECTORY/inventory/` | Written by the create playbook: `ansible_host`, port, user, credentials for each instance. Auto-injected by molecule on every command — never listed in a stack declaration. It is parsed before the static sources, so static inventories must never define `ansible_*` connection vars for test hosts |

Notes on the shared layer:

- `localhost` is declared explicitly so it joins `all` and sees
  `group_vars/all.yml` — implicit localhost would not, and the lifecycle
  playbooks run there.
- The utils playbooks provision every host in the `molecule` group — the
  loaded inventory stack IS the declaration of what a scenario tests, so
  plays can always target `hosts: molecule`.
- Per-host vars (`molecule_aws_ami_name`, `molecule_aws_image_id`,
  `molecule_aws_ami_owner`, `molecule_aws_connection`,
  `molecule_aws_instance_type`) are read via
  `hostvars[<instance>]` by the utils playbooks; everything else in
  `group_vars/all.yml` is play-level and cannot be overridden per host.

Typical scenario-local overlays:

```yaml
# Add a var to existing shared hosts (matched by name, nothing re-provisioned)
---
all:
  children:
    molecule:
      hosts:
        rhel10:
          my_scenario_expected_value: "x"
```

```yaml
# Map shared hosts into the groups a repo playbook targets
---
all:
  children:
    server:
      hosts:
        rhel10:
    client:
      hosts:
        rhel9:
```

The second pattern is how playbook scenarios satisfy plays like
`hosts: server` without touching the playbook: group membership is additive
across inventory sources, so `rhel10` stays in `molecule` (the utils playbooks
provision it) *and* joins `server` (the playbook targets it).
