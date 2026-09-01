# Adding molecule scenarios

Step-by-step guides for adding a scenario that tests a playbook or a role.
Background on how the base config and inventories compose is in
[configuration-inheritance.md](configuration-inheritance.md).

## Prerequisites

Required once per environment — the shared base config does not auto-install
deps (see [molecule/README.md](../../molecule/README.md#setup)):

```bash
pip install -r molecule/requirements.txt
ansible-galaxy collection install -r molecule/requirements.yml
# Collections the content under test needs (ansible.windows, community.windows,
# chocolatey.chocolatey, community.postgresql, ...). Install to the galaxy
# default path, NOT the repo's collections/ tree: molecule generates its own
# ansible.cfg and a converge playbook's playbook_dir is the scenario directory,
# so <repo>/collections would not be discovered.
ansible-galaxy collection install -r collections/requirements.yml
# AWS credentials + AWS_REGION in the environment; a minimally scoped IAM
# user for the tests can be minted per aws-iam-user.md
```

The `dependency` step runs all three of these on every molecule run, so this
is only needed to pre-install or to lint outside molecule.

## Naming convention

- Playbook test: `playbook-<playbook-name-without-file-extension>`
  (`playbooks/linux-yum-repo-client.yml` → `playbook-linux-yum-repo-client`)
- Role test: `role-<name-of-role>`
  (`roles/yum_repo_client` → `role-yum-repo-client`)

## Anatomy of a scenario

```text
molecule/<scenario>/
├── molecule.yml    # usually empty — inherits the base config
├── converge.yml    # applies the content under test
├── verify.yml      # asserts the outcomes
├── prepare.yml     # optional: test-harness prerequisites (skipped when absent)
└── inventory/      # optional: group mappings / extra vars for this scenario
    └── hosts.yml
```

Every scenario automatically gets: the shared Linux instances
(`utils/inventory/linux/` = `rhel10`/`rhel9`), create/destroy lifecycle
(create waits for guest connectivity), an optional scenario-local
`prepare.yml` hook (skipped when absent), and repo role resolution by name. Windows
(`utils/inventory/windows/` = `win2022`, WinRM) is opt-in: declare the inventory
stack in the scenario's `molecule.yml` with the windows layer — the utils
provision exactly the hosts the loaded inventories declare (see
[configuration-inheritance.md](configuration-inheritance.md)).

## Walkthrough: testing a playbook

Example: `playbooks/linux-yum-repo-client.yml`, whose play targets
`hosts: all_rhel`.

```bash
mkdir -p molecule/playbook-linux-yum-repo-client/inventory
printf -- '---\n' > molecule/playbook-linux-yum-repo-client/molecule.yml
```

`converge.yml` — import the playbook; paths are relative to the scenario
directory:

```yaml
---
- name: Converge
  ansible.builtin.import_playbook: ../../playbooks/linux-yum-repo-client.yml
```

`inventory/hosts.yml` — map the shared instances into the groups the playbook
targets (check its `hosts:` lines). Group membership is additive, so the hosts
remain in `molecule` for the lifecycle playbooks:

```yaml
---
all:
  children:
    all_rhel:
      hosts:
        rhel10:
        rhel9:
```

Playbook vars (survey defaults, repo-specific overrides) can be set in the
same file per host, or in `inventory/group_vars/<group>.yml`.
For this playbook, set `yum_repo_client_repositories` in group_vars.

`verify.yml` — assert what the playbook is supposed to have done, not that
Ansible modules work:

```yaml
---
- name: Verify
  hosts: all_rhel
  gather_facts: false
  tasks:
    - name: Read configured repos
      ansible.builtin.command: dnf repolist --enabled
      changed_when: false
      register: repolist

    - name: Assert a configured repo is enabled
      ansible.builtin.assert:
        that:
          - "'company-baseos' in repolist.stdout"
```

Run it:

```bash
molecule test -s playbook-linux-yum-repo-client
```

## Walkthrough: testing a role

Example: `roles/yum_repo_client`.

```bash
mkdir molecule/role-yum-repo-client
printf -- '---\n' > molecule/role-yum-repo-client/molecule.yml
```

`converge.yml` — apply the role by name (`ANSIBLE_ROLES_PATH` from the base
config resolves the repo's `roles/`); role vars go under `vars:`:

```yaml
---
- name: Converge
  hosts: molecule
  become: true
  vars:
    yum_repo_client_repositories:
      - name: acl_test_baseos
        description: ACL molecule BaseOS file repo
        file: acl_yum_repo_client_test
        baseurl: file:///var/tmp/yum_repo_client_test/baseos
        enabled: true
        gpgcheck: false
  roles:
    - role: yum_repo_client
```

`verify.yml` — assert the role's contract:

```yaml
---
- name: Verify
  hosts: molecule
  become: true
  gather_facts: false
  tasks:
    - name: Stat managed repo file
      ansible.builtin.stat:
        path: /etc/yum.repos.d/acl_yum_repo_client_test.repo
      register: repo_file

    - name: Assert repo file exists
      ansible.builtin.assert:
        that:
          - repo_file.stat.exists
```

Run it:

```bash
molecule test -s role-yum-repo-client
```

## Iterating

`molecule test` destroys instances at the end (and on failure via cleanup).
While developing, run the phases separately so instances persist between
edits:

```bash
molecule create -s <scenario>     # provision once (create + prepare)
molecule converge -s <scenario>   # re-run after each edit
molecule verify -s <scenario>     # re-run assertions
molecule destroy -s <scenario>    # tear down when done
```

Cheap checks that need no credentials or instances:

```bash
molecule syntax -s <scenario>     # config + playbook syntax through molecule
ansible-lint molecule/<scenario>/
```

If a scenario needs different molecule settings (extra test steps, different
hosts), override only those keys in its `molecule.yml` — see the examples in
[configuration-inheritance.md](configuration-inheritance.md). Never copy the
whole base config into a scenario.
