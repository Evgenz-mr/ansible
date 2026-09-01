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

Use the Teleport proxy, username and authentication method assigned to your environment:

```bash
tsh login --proxy=<teleport-proxy>:443 --user=<teleport-user> --auth=<auth-method>
```

Verify that the jump host is accessible:

```bash
tsh ssh <teleport-user>@dev-ansible-jumphost
```

### 2. Generate an OpenSSH configuration from Teleport

```bash
mkdir -p ~/.ssh
tsh config > ~/.ssh/teleport_config
```

The generated file contains the Teleport SSH certificate paths and a `ProxyCommand` using `tsh proxy ssh`.

Check it if necessary:

```bash
cat ~/.ssh/teleport_config
```

Teleport node names in this configuration use the cluster suffix. For example:

```text
dev-ansible-jumphost.main
```

### 3. Test OpenSSH access to the jump host

```bash
ssh -F ~/.ssh/teleport_config \
  <teleport-user>@dev-ansible-jumphost.main
```

### 4. Test the complete SSH path to a target server

```bash
ssh \
  -F ~/.ssh/teleport_config \
  -J <teleport-user>@dev-ansible-jumphost.main \
  -i ~/.ssh/id_ed25519_ansible \
  ansible_admin@<target-ip>
```

If this command succeeds, the complete path is working:

```text
Mac -> Teleport -> jump host -> target server
```

### 5. Configure the Ansible inventory

Example:

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

  children:
    vps_in:
      hosts:
        vps_in_1:
          ansible_host: <target-ip-1>
        vps_in_2:
          ansible_host: <target-ip-2>
      vars:
        ansible_become: true
        ansible_become_method: sudo
```

Do not commit passwords, private keys, Teleport certificates, real credentials or other secrets to the repository. Prefer Ansible Vault or an external secret store for secrets required by inventories or playbooks.

### 6. Test Ansible before running a playbook

Test one host first:

```bash
ansible -i inventory-dev.yml vps_in_1 -m ping -vvv
```

Expected result:

```text
SUCCESS
"ping": "pong"
```

Then run a playbook against one host:

```bash
ansible-playbook -i inventory-dev.yml playbook.yml --limit vps_in_1
```

After validation, run it against the required group:

```bash
ansible-playbook -i inventory-dev.yml playbook.yml --limit vps_in
```

### How file transfer works

The jump host does not become the Ansible control node. Ansible still runs locally on the Mac. Files referenced by modules such as `copy` and `template` originate on the Mac and are transferred to the target through the SSH connection established via Teleport and the jump host.

The target servers themselves still need network access for tasks that download content remotely. For example, this task runs `apt` on the target, not on the Mac:

```yaml
- name: Update apt cache
  ansible.builtin.apt:
    update_cache: true
  become: true
```

Therefore, if a playbook performs package installation or `apt update`, the relevant target host needs access to its configured package repositories (or an explicitly configured package proxy/mirror). A working Teleport/SSH path does not provide outbound Internet access to the target.

### Troubleshooting

If Ansible reports an SSH error such as:

```text
Connection closed by UNKNOWN port 65535
```

first test the complete SSH command from step 4. A common cause is using `ProxyJump=dev-ansible-jumphost` without loading the Teleport-generated SSH configuration or without the Teleport cluster suffix (for example `.main`).

If SSH and `ansible -m ping` work but a playbook fails with:

```text
Failed to update apt cache after 5 retries
```

check outbound connectivity from the affected target server. For example:

```bash
curl -4 -I --connect-timeout 10 https://archive.ubuntu.com/ubuntu/
ip route
```

If the connection times out, fix the target server's outbound route/NAT/firewall/proxy configuration rather than the Teleport or Ansible SSH settings.

### Summary

The working design is:

```text
Ansible/playbooks/files on Mac
          |
          v
Teleport SSH transport
          |
          v
Ansible jump host
          |
          v
Target hosts
```

Teleport and the jump host solve management-plane SSH access. Package repositories and other remote resources used by playbook tasks remain data-plane dependencies of the target hosts themselves.
