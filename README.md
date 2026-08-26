# Ansible Content Library — Community Content

Public corpus for community Ansible contributions. The
[Red Hat Services Platform Team](https://github.com/redhat-services-platform-team)
maintains merge rights. This is **not** the sales-motion
[acl-ready-to-use-content](https://github.com/redhat-services-platform-team/acl-ready-to-use-content)
repository; the **engineering bar is the same**.

This repository follows the
[Red Hat COP Good Practices for Ansible (GPA)](https://redhat-cop.github.io/automation-good-practices/).

Landing here is **not** Automation Hub validated or certified. See
[docs/specifications.md](docs/specifications.md) § Hub promotion.

## Principles

1. **GPA** — role variables `{rolename}_*` / `__{rolename}_*`; thin playbooks; role READMEs.
2. **ansible-lint `production`** — CI and pre-commit must pass.
3. **AAP 2.6+** — contributors attest that content was tested on supported AAP, or include a Molecule scenario.
4. **Catalog headers** — every playbook has the maturity-catalog comment block used by the library UI.
5. **No secrets** — gitleaks in CI; never commit vaulted inventory or tokens.

## Layout

```
acl-community-content/
├── playbooks/                 # Flat linux-*.yml / win-*.yml (no subdirectories)
├── roles/                     # Role logic (GPA layout + README)
├── collections/requirements.yml
├── execution_environments/    # Optional custom EE
├── controller_config/         # AAP Configuration as Code (grows with content)
├── inventory/                 # Example inventory only
├── docs/specifications.md
└── ansible.cfg                # roles_path = roles
```

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). Open a pull request; do not push to `main`.

```bash
python3.12 -m venv .venv
source .venv/bin/activate
pip install -r requirements-dev.txt
pre-commit install
pre-commit run --all-files
```

## License

Apache License 2.0. See [LICENSE](LICENSE).
