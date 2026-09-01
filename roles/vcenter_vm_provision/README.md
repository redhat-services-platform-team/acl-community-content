# Role: `vcenter_vm_provision`

**Tags:** pillar:cloud,lifecycle:day1,maturity:l2,platform:linux,tech:vmware
**Supported OS:** Any (delegates to localhost / vCenter API)

Clone a virtual machine from a vCenter template using
`community.vmware.vmware_guest`. Supports hardware overrides, network
configuration, and guest customization (hostname, DNS).

## Requirements

- `community.vmware` collection (>= 5.0.0)
- `pyvmomi` Python library on the execution host
- VMware Tools installed on the source template (required for guest
  customization and IP address reporting)
- vCenter credentials with permissions to clone templates in the target
  datacenter/cluster

## Role Variables

All variables are prefixed `vcenter_vm_provision_` per GPA.

### Connection

| Variable | Default | Description |
|----------|---------|-------------|
| `vcenter_vm_provision_hostname` | `""` | vCenter hostname or IP |
| `vcenter_vm_provision_username` | `""` | vCenter username |
| `vcenter_vm_provision_password` | `""` | vCenter password |
| `vcenter_vm_provision_validate_certs` | `true` | Validate TLS certificates |
| `vcenter_vm_provision_port` | `443` | vCenter API port |

### Placement

| Variable | Default | Description |
|----------|---------|-------------|
| `vcenter_vm_provision_datacenter` | `""` | Target datacenter (required) |
| `vcenter_vm_provision_cluster` | `""` | Target cluster (required) |
| `vcenter_vm_provision_datastore` | `""` | Target datastore (optional; uses template default) |
| `vcenter_vm_provision_folder` | `""` | VM folder path (optional) |

### Template and VM Identity

| Variable | Default | Description |
|----------|---------|-------------|
| `vcenter_vm_provision_template` | `""` | Source template name (required) |
| `vcenter_vm_provision_name` | `""` | New VM name (required) |
| `vcenter_vm_provision_state` | `poweredon` | Desired state: `poweredon`, `poweredoff`, `present` |

### Hardware Overrides

Set to `0` or omit to keep the template defaults.

| Variable | Default | Description |
|----------|---------|-------------|
| `vcenter_vm_provision_cpu` | `0` | Number of vCPUs |
| `vcenter_vm_provision_memory_mb` | `0` | Memory in MB |
| `vcenter_vm_provision_disks` | `[]` | List of disk overrides (see example) |
| `vcenter_vm_provision_networks` | `[]` | List of network adapter configs (see example) |

### Guest Customization

| Variable | Default | Description |
|----------|---------|-------------|
| `vcenter_vm_provision_customization` | see defaults | Inline customization (hostname, domain, dns_servers, dns_suffix) |
| `vcenter_vm_provision_customization_spec` | `""` | Existing vCenter customization spec name (overrides inline) |

### Post-Provisioning

| Variable | Default | Description |
|----------|---------|-------------|
| `vcenter_vm_provision_wait_for_ip` | `true` | Wait for VMware Tools to report an IP |
| `vcenter_vm_provision_wait_for_ip_timeout` | `600` | Timeout in seconds |

## Dependencies

None.

## Example Playbook

```yaml
- name: Provision a RHEL 9 VM from template
  hosts: localhost
  connection: local
  gather_facts: false
  roles:
    - role: vcenter_vm_provision
      vcenter_vm_provision_hostname: vcenter.example.com
      vcenter_vm_provision_username: administrator@vsphere.local
      vcenter_vm_provision_password: "{{ vault_vcenter_password }}"
      vcenter_vm_provision_validate_certs: false
      vcenter_vm_provision_datacenter: DC01
      vcenter_vm_provision_cluster: Cluster01
      vcenter_vm_provision_datastore: Datastore01
      vcenter_vm_provision_template: rhel9-golden
      vcenter_vm_provision_name: my-rhel9-vm
      vcenter_vm_provision_cpu: 4
      vcenter_vm_provision_memory_mb: 8192
      vcenter_vm_provision_networks:
        - name: "Production VLAN"
          device_type: vmxnet3
          ip: "10.100.1.50"
          netmask: "255.255.255.0"
          gateway: "10.100.1.1"
      vcenter_vm_provision_customization:
        hostname: my-rhel9-vm
        domain: example.com
        dns_servers:
          - "10.100.1.10"
          - "10.100.1.11"
```

## See Also

For more comprehensive VMware automation — including snapshot management,
OVF export/deploy, content library operations, cluster settings, and ESXi
maintenance — see the
[cloud.vmware_ops](https://github.com/redhat-cop/cloud.vmware_ops)
validated Ansible collection from Red Hat CoP.

## License

Apache-2.0

## Author

Ansible Services Platform Team, Red Hat
