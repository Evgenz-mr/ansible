# Ansible through Teleport + jump host

This document contains the complete client-side setup for running Ansible on macOS while using Teleport and an SSH jump host only as transport.

## Architecture

```text
macOS / iMac
  ansible-playbook, roles, templates, downloads
        |
        | OpenSSH using tsh-generated config
        v
Teleport proxy
        |
        v
dev-ansible-jumphost.main
        |
        | SSH
        v
managed hosts
```

## 1. Teleport login

```bash
tsh login \
  --proxy=<teleport-proxy>:443 \
  --user=<teleport-user> \
  --auth=<auth-method>
```

Check the session:

```bash
tsh status
```

## 2. Generate OpenSSH config

```bash
mkdir -p ~/.ssh
tsh config > ~/.ssh/teleport_config
chmod 600 ~/.ssh/teleport_config
```

The generated config contains Teleport certificate paths and `tsh proxy ssh` as a `ProxyCommand`.

## 3. Test the Teleport node

```bash
ssh -F ~/.ssh/teleport_config \
  <teleport-user>@dev-ansible-jumphost.main
```

## 4. Test a target through the jump host

```bash
ssh \
  -F ~/.ssh/teleport_config \
  -J <teleport-user>@dev-ansible-jumphost.main \
  -i ~/.ssh/id_ed25519_ansible \
  ansible_admin@<target-ip>
```

If this works, Ansible can use exactly the same SSH path.

## 5. Inventory options

Recommended inventory variables:

```yaml
---
all:
  vars:
    ansible_connection: ssh
    ansible_user: ansible_admin
    ansible_ssh_private_key_file: "/Users/<mac-user>/.ssh/id_ed25519_ansible"
    ansible_ssh_common_args: >-
      -F /Users/<mac-user>/.ssh/teleport_config
      -o ProxyJump=<teleport-user>@dev-ansible-jumphost.main
      -o StrictHostKeyChecking=no
      -o ServerAliveInterval=30
      -o ServerAliveCountMax=6
      -o ConnectTimeout=20
```

The important Ansible option is `ansible_ssh_common_args`. It is passed to OpenSSH for every managed host.

Equivalent one-off CLI test:

```bash
ansible -i inventory-dev.yml vps_in_1 \
  -m ping -vvv \
  --ssh-common-args='-F /Users/<mac-user>/.ssh/teleport_config -o ProxyJump=<teleport-user>@dev-ansible-jumphost.main -o StrictHostKeyChecking=no'
```

Normally the options should stay in inventory/group_vars rather than being typed for every run.

## 6. ansible.cfg

A useful local configuration is:

```ini
[defaults]
inventory = ./inventory-dev.yml
host_key_checking = False
forks = 10
timeout = 30
retry_files_enabled = False

[ssh_connection]
pipelining = True
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
control_path = %(directory)s/%%h-%%p-%%r
```

Do not put `ProxyJump` in both `ansible.cfg` and inventory. Keep environment-specific routing in inventory/group_vars.

## 7. Test order

```bash
# Teleport session
tsh status

# Jump host
ssh -F ~/.ssh/teleport_config <teleport-user>@dev-ansible-jumphost.main

# Target through jump host
ssh -F ~/.ssh/teleport_config \
  -J <teleport-user>@dev-ansible-jumphost.main \
  -i ~/.ssh/id_ed25519_ansible \
  ansible_admin@<target-ip>

# Ansible transport
ansible -i inventory-dev.yml vps_in_1 -m ping -vvv

# One-host playbook test
ansible-playbook -i inventory-dev.yml playbook.yml --limit vps_in_1

# Group run
ansible-playbook -i inventory-dev.yml playbook.yml --limit vps_in
```

## 8. Internet access is independent from SSH access

Teleport and the jump host only solve management-plane connectivity. Package tasks execute on the target host itself.

For example:

```yaml
- name: Update apt cache
  ansible.builtin.apt:
    update_cache: true
  become: true
```

requires outbound connectivity from the managed host to its configured Ubuntu/Debian repositories.

If SSH works but `apt update` fails, test on the affected host:

```bash
ip route
getent ahosts archive.ubuntu.com
curl -4 -I --connect-timeout 10 https://archive.ubuntu.com/ubuntu/
```

Fix route/NAT/firewall/proxy access on the target; do not change Teleport routing for this failure.

## Security

Never commit real passwords, private keys, Teleport certificates, Keycloak client secrets, join tokens or recovery codes. Use Ansible Vault or an external secrets manager.
