# netactuate-ansible-bgp-bird2

Ansible playbook for deploying anycast BGP infrastructure on NetActuate's global edge network
using BIRD2. Provisions Ubuntu 24.04 LTS compute nodes across multiple Points of Presence
(PoPs), configures BGP peering sessions with NetActuate's upstream routers, and binds your
anycast IP prefixes — giving you globally routed anycast addresses that respond from the
nearest PoP to each client.

## What This Deploys

For each node defined in your inventory:

1. A NetActuate compute VM running Ubuntu 24.04 LTS
2. SSH key access and base package installation
3. BIRD2 BGP daemon with IPv4 and IPv6 peering sessions (redundant by default)
4. Anycast IP addresses bound to the loopback interface via `rc.local`
5. Sysctl tuning for IPv4/IPv6 forwarding

End state: your anycast prefixes are announced from every PoP, traffic is routed to the
nearest node, and `birdc show protocols` shows all sessions as Established.

## Prerequisites

### NetActuate Account Requirements

| Requirement | Where to Get It |
|-------------|----------------|
| API key | [portal.netactuate.com/account/api](https://portal.netactuate.com/account/api) — whitelist your IP under "Manage API ACL(s)" |
| BGP group ID | Pre-configured by NetActuate — contact support@netactuate.com |
| Contract ID | Visible on the portal API page |
| ASN | Customer-owned or NetActuate-assigned (provided with BGP group setup) |
| IPv4 prefix | Assigned by NetActuate (typically /24) |
| IPv6 prefix | Assigned by NetActuate (typically /48) |

### Control Node Setup

Your local machine needs Python 3, Ansible, the NetActuate API library, and the NetActuate
Ansible collection.

#### macOS

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install ansible
pip install git+https://github.com/netactuate/naapi.git@vapi2
ansible-galaxy collection install git+https://github.com/netactuate/ansible-collection-compute.git,vapi2
```

#### Linux

```bash
sudo apt install python3-venv python3-pip
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install ansible
pip install git+https://github.com/netactuate/naapi.git@vapi2
ansible-galaxy collection install git+https://github.com/netactuate/ansible-collection-compute.git,vapi2
```

#### Windows (WSL2)

Install WSL2 with Ubuntu 24.04, then follow the Linux instructions above.

#### AI-Assisted (Claude Code / Cursor / Copilot)

If you are using an AI coding assistant, provide it with this prompt — replace the
placeholder values with your actual settings from NetActuate:

```
Set up this NetActuate anycast deployment for me. Here are my settings:

- API Key: <YOUR_API_KEY>
- Contract ID: <YOUR_CONTRACT_ID>
- BGP Group ID: <YOUR_BGP_GROUP_ID>
- Origin ASN: <YOUR_ASN>
- IPv4 Prefix: <YOUR_IPV4_PREFIX>/<CIDR> (bind to <FIRST_USABLE_IP>)
- IPv6 Prefix: <YOUR_IPV6_PREFIX>/<CIDR> (bind to <FIRST_USABLE_IP>)
- Node hostname pattern: <PREFIX>-POP-edge1.<DOMAIN>
- Billing plan: VR8x4x50
- Locations: LAX,SJC,FRA,LHR,HKG,TYO,SIN,DFW,CHI,SYD,GRU,MIA,LGA,OTP,IAD3

Please:
1. Set up the Python venv and install all dependencies
2. Update group_vars/all with my settings
3. Add my SSH public key to keys.pub
4. Build the hosts inventory for all locations
5. Run createnode.yaml to provision all nodes
6. Run bgp.yaml to configure BGP and BIRD on all nodes
7. Show me how to verify BGP sessions
```

## Configuration

### Step 1: Add your SSH public key

Copy your public key into `keys.pub`:

```bash
cat ~/.ssh/id_ed25519.pub > keys.pub
# or: cat ~/.ssh/id_rsa.pub > keys.pub
```

### Step 2: Edit group_vars/all

Copy the example and fill in your values:

```bash
cp group_vars/all.example group_vars/all
```

| Variable | Type | Description | Example |
|----------|------|-------------|---------|
| `auth_token` | string | NetActuate API key | `"abc123def456"` |
| `ssh_public_key` | string | Path to public key file | `"keys.pub"` |
| `operating_system` | string | OS image name | `"Ubuntu 24.04 LTS (20240423)"` |
| `bgp_group` | string | BGP group ID | `"12345"` |
| `contract_id` | string | Billing contract ID | `"67890"` |
| `bgp_networks.IPv4[].net` | string | IPv4 prefix to announce | `"192.0.2.0/24"` |
| `bgp_networks.IPv4[].ips` | list | IPv4 anycast addresses to bind | `["192.0.2.1"]` |
| `bgp_networks.IPv4[].origin` | string | Origin ASN | `"65000"` |
| `bgp_networks.IPv6[].net` | string | IPv6 prefix to announce | `"2001:db8::/48"` |
| `bgp_networks.IPv6[].ips` | list | IPv6 anycast addresses to bind | `["2001:db8::1"]` |
| `bgp_networks.IPv6[].origin` | string | Origin ASN | `"65000"` |

### Step 3: Edit the hosts inventory

Each node is one line under `[nodes]` with a hostname, location code, BGP flag, and plan:

```ini
[master]
localhost ansible_connection=local ansible_python_interpreter=.venv/bin/python3

[nodes]
myapp-LAX-edge1.example.com location=LAX bgp_enabled=True plan='VR8x4x50'
myapp-FRA-edge1.example.com location=FRA bgp_enabled=True plan='VR8x4x50'
myapp-SIN-edge1.example.com location=SIN bgp_enabled=True plan='VR8x4x50'
```

## Deployment

Activate your venv first:

```bash
source .venv/bin/activate
```

### Provision nodes

```bash
ansible-playbook createnode.yaml
```

This creates VMs, waits for SSH, and runs basic Ubuntu setup. Re-run if any nodes time out
during provisioning — it is idempotent and will skip already-provisioned nodes.

### Configure BGP

```bash
ansible-playbook bgp.yaml
```

This fetches BGP peering details from the NetActuate API, installs BIRD2, generates config,
binds anycast IPs, and tunes sysctl settings.

### Limit to a single node

```bash
ansible-playbook createnode.yaml -l myapp-LAX-edge1.example.com
ansible-playbook bgp.yaml -l myapp-LAX-edge1.example.com
```

## Validation

SSH to any node and check BGP status:

```bash
ssh ubuntu@<node-ip>
sudo -i
```

### Check BGP sessions

```bash
birdc show protocols
```

All BGP sessions should show state **Established**. You will see multiple sessions — typically
two IPv4 peers and two IPv6 peers (redundant peering).

### Check announced routes

```bash
birdc show route
```

You should see your IPv4 and IPv6 prefixes listed as static routes being exported to BGP
peers.

### Test anycast from external locations

Use [ping.pe](https://ping.pe) to test reachability:

- `https://ping.pe/<YOUR_ANYCAST_IPV4>`
- `https://ping.pe/<YOUR_ANYCAST_IPV6>`

Responses should come from different PoPs depending on the source location.

## Teardown

```bash
ansible-playbook deletenode.yaml
```

This terminates all nodes listed in the hosts inventory and cancels billing. BGP sessions are
automatically cleaned up when the server is deleted.

## PoP Location Codes

See the full list of available locations at
[portal.netactuate.com/account/api](https://portal.netactuate.com/account/api) or
[netactuate.com/edge-locations](https://www.netactuate.com/edge-locations/).

## VM Plans

Available plans and their specifications can be found at
[portal.netactuate.com/account/api](https://portal.netactuate.com/account/api).

## Related Resources

- [NetActuate API Documentation](https://www.netactuate.com/docs/)
- [NetActuate Ansible Collection](https://github.com/netactuate/ansible-collection-compute)
- [BIRD2 Documentation](https://bird.network.cz/?get_doc&v=20)
- [NetActuate Portal](https://portal.netactuate.com)

## Need Help?

- NetActuate support: support@netactuate.com
- BGP group setup and prefix assignments: contact NetActuate support
- API access and billing: [portal.netactuate.com](https://portal.netactuate.com)
