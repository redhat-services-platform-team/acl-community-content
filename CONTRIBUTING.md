# Contributing to ACL Community Content

Thank you for contributing. This is a **public** repository. Merged content is
Apache-2.0 licensed and must meet the same bar as
[acl-ready-to-use-content](https://github.com/redhat-services-platform-team/acl-ready-to-use-content).

Read [docs/specifications.md](docs/specifications.md) before opening a PR.

## Developer Certificate of Origin

Every commit must be signed off (`git commit -s`) to certify the
[Developer Certificate of Origin](https://developercertificate.org/):

```
Signed-off-by: Your Name <you@example.com>
```

Use the name and email associated with your GitHub account.

## What belongs here

Community playbooks and roles that follow GPA, pass ansible-lint `production`,
and have been tested on AAP 2.6+ (or include a Molecule scenario a maintainer
can run). Broader than sales starter-pack content; still not a dumping ground
for unlinted or untested YAML.

## Pull request checklist

- [ ] Playbook under `playbooks/` named `linux-<use-case>.yml` or `win-<use-case>.yml`
- [ ] Tasks live in `roles/<role>/` (thin playbook: hosts, become, facts, `roles:`)
- [ ] Catalog header before `---` (`Playbook:`, `Description:`, `Tags:` including
      `pillar:`, `lifecycle:`, `maturity:`, `Author:`, `Version:`, `Date:`, `License: Apache-2.0`)
- [ ] Role `README.md` with GPA sections and `**Tags:**` line
- [ ] Public vars `{rolename}_*`, internals `__{rolename}_*`
- [ ] `pre-commit run --all-files` (or CI) is green
- [ ] Tested on AAP 2.6+ **or** Molecule scenario documented in the PR
- [ ] No secrets, customer names, or internal-only URLs
- [ ] Commits include `Signed-off-by`

## Review

SPT maintainers (`americas-spt`) review and merge. One approval is required on
`main`. Maintainers may ask for lint or test evidence before merge.

Questions: open a GitHub issue (not for security — see [SECURITY.md](SECURITY.md)).
