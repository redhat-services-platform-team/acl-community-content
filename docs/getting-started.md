# Getting Started

This repository starts empty of playbooks. Use it as the layout and gates for
new community contributions. See [CONTRIBUTING.md](../CONTRIBUTING.md) and
[specifications.md](specifications.md).

## Prerequisites

- Python 3.12 (ansible-lint is not compatible with 3.14 today)
- Ansible 2.16+ / AAP 2.6+ for runtime testing
- Git configured with `user.name` and `user.email` for DCO (`git commit -s`)

## Clone and local lint

```bash
git clone https://github.com/redhat-services-platform-team/acl-community-content.git
cd acl-community-content

python3.12 -m venv .venv
source .venv/bin/activate
pip install -r requirements-dev.txt
pre-commit install
pre-commit run --all-files
```

Install runtime collections when you add playbooks:

```bash
ansible-galaxy collection install -r collections/requirements.yml
```

## Inventory

Copy and edit the example:

```bash
cp inventory/inventory.example inventory/lab.ini
```

Run playbooks from the **repository root** so `ansible.cfg` resolves
`roles_path = roles`.

## Execution environment (optional)

```bash
ansible-builder build \
  -f execution_environments/execution-environment.yml \
  -t acl-community-content-ee
```

## AAP

This GitHub repository is **public**. Point an AAP project at the HTTPS URL
and leave the SCM credential empty. Copy
`controller_config/site_config.yml.example` when CaC job templates exist for
a playbook you want to launch.
