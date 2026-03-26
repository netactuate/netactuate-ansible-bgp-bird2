# netactuate-ansible-bgp-bird2 — AI Provisioning Context

This playbook deploys anycast BGP infrastructure on NetActuate's global edge using BIRD2.
Success means: every node has BIRD2 running, all BGP sessions are Established, and your
anycast prefixes are reachable from anywhere in the world via the nearest PoP.

## Required Inputs

You need these values from the customer before you can do anything:

| Input | Source | Example |
|-------|--------|---------|
| API key | portal.netactuate.com/account/api | `"abc123..."` |
| Contract ID | Portal API page | `"12345"` |
| BGP group ID | NetActuate support | `"67890"` |
| ASN | NetActuate or customer-owned | `"65000"` |
| IPv4 prefix | Assigned with BGP group | `"192.0.2.0/24"` |
| IPv6 prefix | Assigned with BGP group | `"2001:db8::/48"` |
| Anycast IPv4 | First usable in prefix | `"192.0.2.1"` |
| Anycast IPv6 | First usable in prefix | `"2001:db8::1"` |
| SSH public key | Customer's local machine | Content of `~/.ssh/id_ed25519.pub` |
| Locations | Customer choice | `LAX, FRA, SIN, LHR, TYO` |
| Plan | Customer choice | `"VR8x4x50"` |
| Hostname pattern | Customer choice | `"myapp-POP-edge1.example.com"` |

## Files to Edit

### 1. `keys.pub`
Paste the customer's SSH public key (entire content of their `~/.ssh/id_ed25519.pub` or `~/.ssh/id_rsa.pub`).

### 2. `group_vars/all`
Copy from `group_vars/all.example` and set:
```yaml
auth_token: "<API_KEY>"
bgp_group: "<BGP_GROUP_ID>"
contract_id: "<CONTRACT_ID>"

bgp_networks:
    IPv4:
    - { net: "<IPV4_PREFIX>/<CIDR>", ips: [ "<ANYCAST_IPV4>" ], origin: "<ASN>" }
    IPv6:
    - { net: "<IPV6_PREFIX>/<CIDR>", ips: [ "<ANYCAST_IPV6>" ], origin: "<ASN>" }
```

Do not change `operating_system`, `ssh_public_key`, or `ansible_ssh_extra_args`.

### 3. `hosts`
Add one line per node under `[nodes]`:
```ini
myapp-LAX-edge1.example.com location=LAX bgp_enabled=True plan='VR8x4x50'
myapp-FRA-edge1.example.com location=FRA bgp_enabled=True plan='VR8x4x50'
myapp-SIN-edge1.example.com location=SIN bgp_enabled=True plan='VR8x4x50'
```

Replace `myapp` with customer's hostname prefix, `example.com` with their domain.
Replace the plan if they requested something different.

## Commands to Run

```bash
# 1. Set up environment (one time)
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install ansible
pip install git+https://github.com/netactuate/naapi.git@vapi2
ansible-galaxy collection install git+https://github.com/netactuate/ansible-collection-compute.git,vapi2

# 2. Provision VMs
ansible-playbook createnode.yaml
# Re-run if any nodes fail with SSH timeout — it is idempotent

# 3. Configure BGP + BIRD2
ansible-playbook bgp.yaml
```

## How to Validate

SSH to each node and check:

```bash
ssh ubuntu@<node-ip>
sudo birdc show protocols
```

**Expected:** All BGP protocol lines show `Established`. You should see 2 IPv4 and 2 IPv6
sessions (redundant peering).

```bash
sudo birdc show route
```

**Expected:** Your IPv4 and IPv6 prefixes appear as static routes exported to BGP.

External test: `https://ping.pe/<ANYCAST_IPV4>` should show responses from multiple PoPs.

## How to Tear Down

```bash
ansible-playbook deletenode.yaml
```

Removes all VMs and cancels billing. BGP sessions are cleaned up automatically.

## Common Errors and Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `UNREACHABLE` during createnode | VM still booting | Wait 60s and re-run `createnode.yaml` |
| `Failed to connect to the host via ssh` | Cloud-init not finished | Re-run — playbook waits for SSH |
| `netactuate.compute.node` not found | Collection not installed | Run `ansible-galaxy collection install ...` |
| `naapi` import error | naapi not installed in venv | Run `pip install git+...naapi.git@vapi2` |
| BGP session stays at `Connect` or `Active` | Peering not yet converged | Wait 2–5 minutes; check `birdc show protocols all` for error details |
| `No server found in the API` on delete | host_vars out of sync | Re-run `createnode.yaml` first to refresh host_vars |

## Full Variable Reference

| Variable | Location | Type | Description |
|----------|----------|------|-------------|
| `auth_token` | group_vars/all | string | NetActuate API key |
| `ssh_public_key` | group_vars/all | string | Path to public key file (default: `keys.pub`) |
| `operating_system` | group_vars/all | string | OS image: `Ubuntu 24.04 LTS (20240423)` |
| `bgp_group` | group_vars/all | string | BGP group ID from NetActuate |
| `contract_id` | group_vars/all | string | Billing contract ID |
| `bgp_networks` | group_vars/all | dict | IPv4/IPv6 prefixes, anycast IPs, and origin ASN |
| `location` | hosts (per-node) | string | PoP code (e.g., `LAX`, `FRA`) |
| `bgp_enabled` | hosts (per-node) | boolean | Must be `True` for BGP nodes |
| `plan` | hosts (per-node) | string | VM plan (e.g., `VR8x4x50`) |

## BIRD2 Config Explanation

The `roles/bird/templates/bird.conf` template generates a BIRD2 configuration per node:

- **router id**: Set to the node's public IPv4 address
- **protocol device**: Scans interfaces every 10 seconds
- **protocol static announcement4/announcement6**: Declares your prefixes as static routes
  to be announced via BGP
- **filter export_upstream_v4/v6**: Only exports the static announcement routes, rejects
  everything else — prevents leaking learned routes
- **protocol bgp (IPv4)**: One block per IPv4 upstream peer. Uses customer ASN as local AS,
  node's peering IPv4 as source, NetActuate's peer address and ASN as neighbor. Imports all
  routes, exports only your prefixes via the filter.
- **protocol bgp (IPv6)**: Same pattern for IPv6 peers.

Redundant peering means you get 2 IPv4 + 2 IPv6 BGP sessions per node, providing resilience
if one upstream peer goes down.

The `rc.local` template binds your anycast IPs to the loopback interface and adds blackhole
routes for the full prefixes, ensuring the node can originate the routes even without a
specific next-hop.
