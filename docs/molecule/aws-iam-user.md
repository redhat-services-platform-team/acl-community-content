# Scoped AWS IAM user for molecule

The molecule scenarios only need a narrow slice of EC2, but sandbox accounts
usually hand out broadly privileged credentials. `hack/create-molecule-iam-user.yml`
mints a dedicated IAM user (`molecule-provisioner`) whose policy allows exactly
what `molecule/utils/provisioners/aws/` does — and nothing else. Running the
tests with it limits the blast radius of a leaked key or a misbehaving run to
the molecule resources themselves.

## Creating and using the user

Run once with your privileged sandbox credentials in the environment:

```bash
ansible-playbook hack/create-molecule-iam-user.yml
source hack/molecule-iam.env   # gitignored, mode 0600
molecule test
```

The playbook is idempotent for the user and policy. Each run deletes the
user's existing access keys and mints a fresh one, so `hack/molecule-iam.env`
always holds the user's only valid key — there is never a stale key to clean
up.

The policy is pinned to one region, captured from `AWS_REGION` at create time
(default `us-east-2`). If you switch regions, rerun the playbook so the policy
follows.

Tear everything down (user, access keys, managed policy, local env file):

```bash
ansible-playbook hack/destroy-molecule-iam-user.yml
```

## What the user can and cannot do

Everything below is additionally pinned to the single region.

| Capability | Scope |
| --- | --- |
| Read EC2 state | `ec2:Describe*` and `ec2:GetPasswordData` (Windows Administrator password) |
| Launch instances | Only when tagged `molecule_scenario=molecule-*` at launch, instance type in a ≤4 vCPU allowlist (`t3.micro`–`t3.xlarge`), AMI from an allowed publisher (Red Hat, Amazon) |
| Terminate instances | Only instances tagged `molecule_scenario=molecule-*` |
| Key pairs | Import/delete `molecule-*` names only |
| Throwaway network (VPC, subnet, IGW, route table, security group) | Create/attach/associate region-wide; **delete only when tagged `molecule_scenario=molecule-*`** |
| Tagging | Tag-on-create only, plus re-tagging of resources it already owns — it cannot adopt or re-tag foreign resources |
| IAM | None at all — in particular no `iam:PassRole`, so it cannot launch instances with an instance profile |

There is no cap on the *number* of instances: IAM cannot count, so the
per-launch vCPU allowlist plus the account's EC2 on-demand vCPU service quota
are the effective fleet-size bound.

`create.yml` stamps `molecule_scenario=molecule-<scenario>` on every resource
it creates, which is what makes the tag-scoped permissions line up.

## Customizing

Playbook vars, overridable with `-e`:

| Variable | Default | Purpose |
| --- | --- | --- |
| `molecule_iam_user_name` | `molecule-provisioner` | IAM user name |
| `molecule_iam_policy_name` | `molecule-aws-provisioner` | Managed policy name |
| `molecule_iam_region` | `$AWS_REGION` or `us-east-2` | Region the policy is pinned to |
| `molecule_iam_ami_owners` | Red Hat, Amazon | AMI publishers `RunInstances` accepts |
| `molecule_iam_instance_types` | `t3.micro`–`t3.xlarge` | Launchable instance types (keep ≤4 vCPU) |
| `molecule_iam_credentials_file` | `hack/molecule-iam.env` | Where the access key lands |

Pass the same overrides to the destroy playbook if you customized the names.

## Troubleshooting

- **`UnauthorizedOperation` deleting a security group / VPC during destroy** —
  the resource predates the `molecule_scenario` tagging (or was made by other
  tooling) and the tag-scoped delete refuses it. The scoped user cannot adopt
  untagged resources by design; clean the strays up once with privileged
  credentials, after which create/destroy cycles are self-contained.
- **`UnauthorizedOperation` on `RunInstances`** — usually a host var stepping
  outside the allowlists: a `molecule_aws_instance_type` not in
  `molecule_iam_instance_types`, or a `molecule_aws_ami_owner` not in
  `molecule_iam_ami_owners`. Extend the list and rerun the create playbook.
- **Everything denied** — region mismatch: the policy is pinned to the region
  captured at create time; rerun the create playbook with the current
  `AWS_REGION`.
