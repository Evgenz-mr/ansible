# Teleport + Keycloak deployment with Ansible

This branch contains an example deployment for a Teleport auth/proxy node, Teleport SSH nodes and Keycloak as an OIDC identity provider.

The examples are intentionally free of real hostnames, IP addresses and secrets. Replace placeholders and keep secrets in Ansible Vault.

## Components

```text
Users
  |
  | HTTPS / OIDC
  v
Teleport Proxy/Auth  <------>  Keycloak
  |
  | Teleport SSH
  v
Teleport nodes / jump hosts
  |
  | SSH
  v
managed infrastructure
```

## Files

```text
playbooks/teleport-keycloak.yml
roles/teleport/
roles/teleport_node/
roles/keycloak/
group_vars/teleport.yml
group_vars/keycloak.yml
examples/secure-access-inventory.yml
requirements.yml
```

## 1. Install required Ansible collections on the Mac

```bash
ansible-galaxy collection install -r requirements.yml
```

## 2. Prepare inventory

Copy the example and replace only environment-specific values:

```bash
cp examples/secure-access-inventory.yml inventory-secure.yml
```

The Mac remains the Ansible control node. If the target infrastructure is reachable only through the existing Teleport jump host, keep the following options in `all.vars`:

```yaml
ansible_ssh_common_args: >-
  -F /Users/<mac-user>/.ssh/teleport_config
  -o ProxyJump=<teleport-user>@dev-ansible-jumphost.main
  -o StrictHostKeyChecking=no
  -o ServerAliveInterval=30
  -o ServerAliveCountMax=6
  -o ConnectTimeout=20
```

## 3. Store secrets with Ansible Vault

Create a vault file:

```bash
ansible-vault create group_vars/vault.yml
```

Example structure:

```yaml
vault_teleport_join_token: "REPLACE_ME"
vault_teleport_oidc_client_secret: "REPLACE_ME"
vault_keycloak_admin_password: "REPLACE_ME"
vault_keycloak_db_password: "REPLACE_ME"
```

Reference the vault file from your environment-specific playbook/inventory or add it with `--extra-vars @group_vars/vault.yml` according to your local workflow.

Never commit the decrypted file.

## 4. Configure DNS and TLS before production use

Create DNS records for the public names used by the configuration, for example:

```text
teleport.example.com -> Teleport proxy IP
keycloak.example.com -> reverse proxy / Keycloak IP
```

Teleport and Keycloak should be exposed through valid TLS certificates. The included Keycloak container listens on local HTTP and is intended to sit behind a TLS reverse proxy/load balancer. Do not expose the plain HTTP management endpoint directly to untrusted networks.

Required connectivity normally includes:

```text
Teleport proxy: TCP/443
Teleport auth:  TCP/3025 only where required internally
Keycloak:        HTTPS/443 through reverse proxy
Teleport nodes: outbound connectivity to the Teleport proxy/auth endpoint
```

Restrict ports to the smallest required source ranges.

## 5. Configure variables

Review:

```bash
$EDITOR group_vars/teleport.yml
$EDITOR group_vars/keycloak.yml
```

Important values:

```yaml
teleport_cluster_name: "example.internal"
teleport_public_addr: "teleport.example.com:443"
teleport_oidc_issuer_url: "https://keycloak.example.com/realms/teleport"
teleport_oidc_redirect_url: "https://teleport.example.com:443/v1/webapi/oidc/callback"
```

and:

```yaml
keycloak_hostname: "keycloak.example.com"
keycloak_realm: "teleport"
keycloak_teleport_client_id: "teleport"
keycloak_teleport_redirect_uri: "https://teleport.example.com:443/v1/webapi/oidc/callback"
```

Pin Teleport and Keycloak versions to versions approved for your environment before deployment.

## 6. Check SSH connectivity first

```bash
ansible -i inventory-secure.yml all -m ping -vvv
```

Do not start package installation until every intended host is reachable.

## 7. Deploy

```bash
ansible-playbook \
  -i inventory-secure.yml \
  playbooks/teleport-keycloak.yml \
  --ask-vault-pass
```

For initial rollout, limit each stage separately:

```bash
ansible-playbook -i inventory-secure.yml playbooks/teleport-keycloak.yml --limit keycloak --ask-vault-pass

ansible-playbook -i inventory-secure.yml playbooks/teleport-keycloak.yml --limit teleport --ask-vault-pass

ansible-playbook -i inventory-secure.yml playbooks/teleport-keycloak.yml --limit teleport_nodes --ask-vault-pass
```

## 8. What the Keycloak role creates

The Keycloak role:

1. installs Podman;
2. creates an isolated Podman network;
3. starts PostgreSQL;
4. starts Keycloak;
5. waits for the Keycloak API;
6. creates the `teleport` realm if it does not exist;
7. creates a confidential OIDC client for Teleport;
8. creates the `teleport-users` group.

The initial administrator should then create or federate users and place approved users in `teleport-users`.

## 9. What the Teleport role creates

The Teleport role:

1. installs Teleport;
2. configures the auth service;
3. configures the proxy service;
4. creates an OIDC connector pointing to Keycloak;
5. maps the Keycloak `teleport-users` group to the Teleport `access` role;
6. enables the Teleport service.

The node role installs Teleport on SSH nodes and joins them to the cluster using a join token stored in Vault.

For a production environment, prefer short-lived/provision-token workflows or cloud/IAM join methods where available instead of long-lived static tokens.

## 10. Verify Keycloak OIDC

After deployment, check the issuer endpoint through HTTPS:

```bash
curl -fsS https://keycloak.example.com/realms/teleport/.well-known/openid-configuration | jq .issuer
```

Expected issuer:

```text
https://keycloak.example.com/realms/teleport
```

Check the Teleport connector on the Teleport auth node:

```bash
sudo tctl get oidc/keycloak
```

Then authenticate from the workstation:

```bash
tsh login --proxy=teleport.example.com:443 --auth=keycloak
```

## 11. Generate the SSH config used by Ansible

```bash
tsh config > ~/.ssh/teleport_config
chmod 600 ~/.ssh/teleport_config
```

Check access:

```bash
ssh -F ~/.ssh/teleport_config <teleport-user>@dev-ansible-jumphost.main
```

Then check the complete path to a managed host:

```bash
ssh \
  -F ~/.ssh/teleport_config \
  -J <teleport-user>@dev-ansible-jumphost.main \
  -i ~/.ssh/id_ed25519_ansible \
  ansible_admin@<target-ip>
```

Finally:

```bash
ansible -i inventory-secure.yml <target-name> -m ping -vvv
```

## 12. Package downloads and Internet access

Teleport does not proxy arbitrary package downloads for managed hosts. Tasks such as:

```yaml
ansible.builtin.apt:
  update_cache: true
```

run on the target. The target therefore needs access to its configured repository, NAT, an HTTP(S) proxy, or an internal mirror.

A successful SSH path does not imply that `apt update` can reach the Internet.

## 13. Security checklist

- use TLS for Teleport and Keycloak;
- keep Keycloak HTTP behind a trusted reverse proxy;
- restrict Teleport auth port exposure;
- keep join tokens and OIDC secrets in Vault/secret manager;
- use short-lived Teleport certificates for operator access;
- enforce MFA in the identity provider/Teleport policy;
- use least-privilege Teleport roles;
- audit `tctl` changes and Teleport access events;
- rotate bootstrap credentials after initial setup;
- back up Keycloak PostgreSQL and Teleport state according to your HA design.
