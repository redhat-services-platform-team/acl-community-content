# Controller Configuration (Configuration as Code)

YAML for provisioning Ansible Automation Platform objects using
`infra.aap_configuration`. These files do **not** affect `ansible-playbook`
runs from the CLI.

This repository is **public**. AAP can sync the project over HTTPS with
**no SCM credential**.

## Layout (bootstrap)

| File | Purpose |
|------|---------|
| `site_config.yml.example` | Template for site-specific values (copy to `site_config.yml`) |

Job templates, inventories, and a `configure_aap.yml` dispatch playbook will
be added as community playbooks land. Do not invent templates for content that
does not exist yet.

## Site config

```bash
cp controller_config/site_config.yml.example controller_config/site_config.yml
```

`site_config.yml` is gitignored. Never commit tokens. Pass
`aap_token` at runtime (`-e aap_token=…`) when CaC playbooks exist.

Machine credentials for Linux and Windows targets must already exist in AAP.
The SCM credential field stays empty for this public repo.
