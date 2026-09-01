# Encrypted transport between IN and OUT nodes

This branch uses a standard WireGuard overlay to encrypt traffic between selected `vps_in` and `vps_out` nodes.

The goal is transport confidentiality and integrity between your own hosts. This is not a DPI-bypass or traffic-disguise recipe; it is a conventional encrypted overlay that can carry application traffic between trusted nodes.

## Architecture

```text
vps_in_1  10.90.0.11/24
     |      WireGuard / UDP
     +----------------------------+
                                  |
                                  v
                         vps_out_1 10.90.0.21/24
```

Applications can then bind/connect to overlay addresses instead of public addresses where appropriate.

## Required variables

Each host needs:

```yaml
wg_address: 10.90.0.11/24
wg_private_key: "..."
wg_peer_public_key: "..."
wg_peer_endpoint: "203.0.113.20:51820"
wg_peer_allowed_ips: "10.90.0.21/32"
wg_listen_port: 51820
wg_persistent_keepalive: 25
```

Keep private keys in Ansible Vault.

Example vault creation:

```bash
ansible-vault create group_vars/wireguard-vault.yml
```

Example contents:

```yaml
vault_vps_in_1_wg_private_key: "REPLACE_ME"
vault_vps_out_1_wg_private_key: "REPLACE_ME"
```

Reference those values from host_vars/group_vars rather than committing raw keys.

## Generate keys

Generate a pair on a trusted workstation or target host:

```bash
umask 077
wg genkey | tee wg-private.key | wg pubkey > wg-public.key
```

Do this separately for each WireGuard peer.

## Firewall

Allow the configured UDP WireGuard port only between expected peer addresses where possible. Example conceptually:

```text
vps_in public IP  -> vps_out UDP/51820
vps_out public IP -> vps_in  UDP/51820
```

Do not expose unrelated administration ports just because the overlay is enabled.

## Deploy

First validate Ansible connectivity:

```bash
ansible -i inventory-secure.yml vps_in:vps_out -m ping
```

Then deploy:

```bash
ansible-playbook \
  -i inventory-secure.yml \
  playbooks/secure-in-out.yml \
  --ask-vault-pass
```

## Verify

On the nodes:

```bash
sudo wg show
ip -br addr show wg0
ip route
```

From IN to OUT overlay address:

```bash
ping -c 3 10.90.0.21
```

From OUT to IN overlay address:

```bash
ping -c 3 10.90.0.11
```

To test a service over the encrypted path, bind the service to its WireGuard address or point the client to the peer WireGuard address, then verify with the service's normal health check.

## Routing application traffic

Do not automatically route `0.0.0.0/0` through the tunnel unless that is explicitly part of the network design. Prefer narrow `AllowedIPs` entries for only the subnets/services that should use the encrypted overlay.

For example:

```text
AllowedIPs = 10.90.0.21/32
```

or for an internal subnet behind the peer:

```text
AllowedIPs = 10.90.0.0/24, 172.20.10.0/24
```

If forwarding a subnet behind a host, separately configure IP forwarding and firewall/NAT rules according to that network design.

## Interaction with Teleport

Teleport management traffic and the WireGuard application overlay solve different problems:

```text
Operator / Ansible management:
Mac -> Teleport -> jump host -> server

Encrypted service transport:
vps_in -> WireGuard -> vps_out
```

You can keep SSH administration on Teleport while using WireGuard only for service-to-service traffic.
