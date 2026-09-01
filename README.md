# Ansible

## Running Ansible from macOS through Teleport and an SSH jump host

This setup keeps Ansible on the local Mac (control node). Playbooks, roles, templates, inventory files and local downloads stay on the Mac. Teleport provides access to the jump host, and the jump host is used only as an SSH transit point to the target servers.

### Connection flow

```text
Mac (Ansible control node)
        |
        | OpenSSH + Teleport configuration
        v
Teleport proxy
        |
        v
dev-ansible-jumphost.main
        |
        | SSH
        v
Target servers (ansible_admin)
```

### 1. Authenticate to Teleport

```bash
tsh login --proxy=<teleport-proxy>:443 --user=<teleport-user> --auth=<auth-method>
```

Verify that the jump host is accessible:

```bash
tsh ssh <teleport-user>@dev-ansible-jumphost
```

### 2. Generate OpenSSH configuration

```bash
mkdir -p ~/.ssh
tsh config > ~/.ssh/teleport_config
chmod 600 ~/.ssh/teleport_config
```

Teleport nodes in the generated config normally use the cluster suffix, for example:

```text
dev-ansible-jumphost.main
```

### 3. Test the jump host

```bash
ssh -F ~/.ssh/teleport_config \
  <teleport-user>@dev-ansible-jumphost.main
```

### 4. Test a target through the jump host

```bash
ssh \
  -F ~/.ssh/teleport_config \
  -J <teleport-user>@dev-ansible-jumphost.main \
  -i ~/.ssh/id_ed25519_ansible \
  ansible_admin@<target-ip>
```

### 5. Ansible SSH options

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

The critical option is `ansible_ssh_common_args`: Ansible passes these arguments to OpenSSH for connections to managed hosts.

Equivalent one-off test:

```bash
ansible -i inventory-dev.yml vps_in_1 \
  -m ping -vvv \
  --ssh-common-args='-F /Users/<mac-user>/.ssh/teleport_config -o ProxyJump=<teleport-user>@dev-ansible-jumphost.main -o StrictHostKeyChecking=no'
```

Full details: [docs/teleport-ansible-options.md](docs/teleport-ansible-options.md).

### 6. Test Ansible

```bash
ansible -i inventory-dev.yml vps_in_1 -m ping -vvv
```

Then one host:

```bash
ansible-playbook -i inventory-dev.yml playbook.yml --limit vps_in_1
```

Then the group:

```bash
ansible-playbook -i inventory-dev.yml playbook.yml --limit vps_in
```

### Internet access on targets

The jump host does not become the Ansible control node. Files used by `copy`, `template`, etc. originate on the Mac and travel through the SSH connection.

Package operations are different: `apt`, `dnf`, image pulls and other downloads execute on the managed host. Therefore target hosts still need their own route/NAT/proxy/mirror access to the required repositories.

If SSH works but a playbook fails with:

```text
Failed to update apt cache after 5 retries
```

check the target:

```bash
ip route
getent ahosts archive.ubuntu.com
curl -4 -I --connect-timeout 10 https://archive.ubuntu.com/ubuntu/
```

Fix the target's outbound networking rather than the Teleport SSH settings.

---

## feature/teleport-secure-access

This branch also contains an extended lab/reference setup for Teleport, Keycloak OIDC and encrypted IN/OUT transport.

### Teleport + Keycloak

Deployment guide:

[docs/teleport-keycloak-deployment.md](docs/teleport-keycloak-deployment.md)

Main playbook:

```bash
ansible-galaxy collection install -r requirements.yml
ansible-playbook \
  -i inventory-secure.yml \
  playbooks/teleport-keycloak.yml \
  --ask-vault-pass
```

Components:

```text
playbooks/teleport-keycloak.yml
roles/teleport/
roles/teleport_node/
roles/keycloak/
group_vars/teleport.yml
group_vars/keycloak.yml
examples/secure-access-inventory.yml
```

The Keycloak role starts PostgreSQL + Keycloak with Podman, creates the Teleport realm/client/group, and the Teleport role creates an OIDC connector mapping the Keycloak `teleport-users` group to a Teleport role.

### Encrypted IN/OUT transport

Guide:

[docs/encrypted-in-out.md](docs/encrypted-in-out.md)

Playbook:

```bash
ansible-playbook \
  -i inventory-secure.yml \
  playbooks/secure-in-out.yml \
  --ask-vault-pass
```

This uses a standard WireGuard overlay for confidentiality/integrity between selected IN and OUT nodes. It is designed as normal encrypted service transport, while Teleport remains the management-access path.

### Secrets

Do not commit:

- private SSH/WireGuard keys;
- Teleport join tokens;
- Teleport certificates;
- Keycloak administrator/database passwords;
- OIDC client secrets;
- real production inventory credentials.

Store them in Ansible Vault or an external secret manager.
