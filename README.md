# Agent management ansible playbooks

Ansible playbooks for managing migration-planner-agent on Fedora CoreOS VMs.

## Playbooks

- **deploy-agent.yml** - Deploys a container image to FCOS VMs running the agent as a Podman quadlet service. Creates an OCI archive from the source image, uploads it to the VM, swaps the image in containers-storage, and restarts the service.
- **reset-agent-db.yml** - Resets the agent database by stopping the service, removing the DuckDB file, and restarting the service.

## Prerequisites

**Control node** (where Ansible runs):

- Ansible 2.15+
- `skopeo` (ships with `podman`)
- `sshpass` (needed for password-based SSH)

**Target VM** (Fedora CoreOS):

- User `core` with lingering enabled
- `skopeo` available
- `planner-agent` quadlet already configured at
  `~core/.config/containers/systemd/planner-agent.container`

## Quick start

1. Copy the example inventory and edit it with your VM address:

   ```bash
   cp inventory.example.yml inventory.yml
   # edit ansible_host to point to your VM
   ```

2. Run a playbook (you will be prompted for the SSH password):

   ```bash
   # Deploy from a remote registry
   ansible-playbook deploy-agent.yml -i inventory.yml \
     -e image_source=quay.io/myorg/migration-planner-agent:latest

   # Deploy from local podman storage (transport is inferred automatically)
   ansible-playbook deploy-agent.yml -i inventory.yml \
     -e image_source=localhost/migration-planner-agent:latest

   # Reset the agent database
   ansible-playbook reset-agent-db.yml -i inventory.yml
   ```

## Variables (deploy-agent.yml)

### Required

| Variable | Passed via | Description |
|---|---|---|
| `image_source` | `-e` | Image reference to deploy, e.g. `quay.io/org/migration-planner-agent:latest` |

### Optional (have defaults)

| Variable | Default | Description |
|---|---|---|
| `image_transport` | auto | Skopeo transport for the source image. Inferred automatically: `containers-storage` when `image_source` starts with `localhost/`, `docker` otherwise. Can be overridden explicitly. |
| `target_image` | `localhost/migration-planner-agent:latest` | Image name written into containers-storage on the VM. |
| `service_name` | `planner-agent` | Name of the quadlet systemd unit to restart. |
| `archive_filename` | `migration-planner-agent.tar` | Filename used for the temporary OCI archive. |

### Prompted at runtime

| Variable | Description |
|---|---|
| `ansible_password` | SSH password for the `core` user on the target VM. |

## Inventory

The playbook targets hosts in the `planner_vms` group. See
`inventory.example.yml` for the expected structure:

```yaml
all:
  children:
    planner_vms:
      hosts:
        fcos-vm-01:
          ansible_host: 192.168.122.10
      vars:
        ansible_user: core
        ansible_ssh_pass: "{{ ansible_password }}"
```

## Variables (reset-agent-db.yml)

| Variable | Default | Description |
|---|---|---|
| `service_name` | `planner-agent` | Name of the quadlet systemd unit. |
| `db_path` | `/var/lib/data/agent.duckdb` | Path to the DuckDB database file to remove. |

## What the playbooks do

### deploy-agent.yml

1. **Creates an OCI archive** on the control node via `skopeo copy`
2. **Uploads** the archive to the VM
3. **Stops** the `planner-agent` quadlet service
4. **Loads** the new image into the user's containers-storage via `skopeo copy`
5. **Starts** the `planner-agent` service
6. **Cleans up** the temporary archive from both the VM and the control node

### reset-agent-db.yml

1. **Stops** the `planner-agent` quadlet service
2. **Removes** the DuckDB database file
3. **Starts** the `planner-agent` service
