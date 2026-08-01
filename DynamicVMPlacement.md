
# Agentless Dynamic VM Placement for ACM Spokes with IPAM support

## Project Blueprint: Minimal IaaS Compute & Network Allocation Engine for ACM Spokes

## 1. Project Overview & Intent
This document details a headless, lightweight, agentless Cloud Infrastructure-as-a-Service (IaaS) control plane designed for a small number of compute nodes running Red Hat Enterprise Linux (RHEL) 9.4 with KVM hypervisor.

The primary objective is to use OpenNebula strictly as a declarative resource scheduler and IP Address Management (IPAM) engine. Virtual machines (VMs) are placed on hypervisors based on topology rules, but they are instantiated in a dormant (HOLD) state without backing storage images. The actual OS injection, lifecycle management, and provisioning are decoupled and handed off to Red Hat Advanced Cluster Management for Kubernetes (ACM) via Redfish VirtualMedia.

------------------------------

## 2. Architecture & Components

```
+---------------------------------------------------------------------------------+
|                         BASTION NODE (Control Machine)                          |
|  [Podman Container: opennebula-frontend]                                        |
|   - Runs OpenNebula Core (`oned` Scheduler and SQLite State DB)                 |
|   - Exposes REST API & CLI tools to Ansible                                     |
|   - Holds SSH Private Key to push domain definitions to KVM nodes               |
+---------------------------------------------------------------------------------+
                                      |
                (Establishes agentless SSH sessions over network)
                                      v
+---------------------------------------------------------------------------------+
|                      RHEL 9.4 COMPUTE NODES / HYPERVISORS                       |
|  [Host Layer]                                                                   |
|   - Native Linux Bridge (`br0`) mapped to a flat physical network pool          |
|   - Native `libvirtd.service` tracking hypervisor capacity                      |
|                                                                                 |
|  [Podman Container: sushy-emulator]                                             |
|   - Runs in host network mode (`--network host` on Port 8000)                   |
|   - Mounts host's libvirt socket (SELinux-relabeled via `:Z`)                   |
|   - Exposes dynamically discovered local VMs as Redfish Rest targets for ACM    |
+---------------------------------------------------------------------------------+
```

## Component Details

* Bastion Control Machine: Hosts the containerized OpenNebula scheduler. It tracks cluster capacity (vCPU, RAM, Datastore Disk space) and executes matchmaking calculations.
* KVM Hypervisors (RHEL 9.4): Completely agentless from an IaaS perspective. They run native libvirt to execute workloads, a flat network bridge (br0) for node connectivity, and a containerized sushy-emulator to interface with ACM.
* Sushy-Emulator (Redfish API Layer): A dynamic gateway container per host that translates ACM’s incoming redfish-virtualmedia+http commands into native local libvirt socket operations.

------------------------------
## 3. Automation & Deployment Lifecycles

### Spoke Configuration & Topology

Ansible reads spoke configuration from two sources:

**From `ansible/vars/acm-spoke.yml`:**
- `num_spokes`: Total number of ACM spoke clusters to create
- `spoke_vm_config`: Dictionary mapping VM roles to their specifications
  - `master`: Control plane VM specifications
    - `count`: Number of master VMs per spoke (typically 3 for HA)
    - `vcpu`: vCPU cores allocated per master VM
    - `memory`: RAM in MB allocated per master VM
    - `disk_size`: Disk size in MB for each master VM
  - `infra`: Infrastructure VM specifications
    - `count`: Number of infra VMs per spoke
    - `vcpu`: vCPU cores allocated per infra VM
    - `memory`: RAM in MB allocated per infra VM
    - `disk_size`: Disk size in MB for each infra VM
  - `worker`: Worker node VM specifications
    - `count`: Number of worker VMs per spoke
    - `vcpu`: vCPU cores allocated per worker VM
    - `memory`: RAM in MB allocated per worker VM
    - `disk_size`: Disk size in MB for each worker VM
- Network topology and anti-affinity rules per role

**From `ansible/vars/all.yml` (group all:vars):**
- `controlplane_network`: The flat network subnet range used to:
  - Define the IP addresses for the hypervisors' br0 interface
  - Configure the IPAM pool for all spoke VMs (ensuring hypervisors and VMs share the same flat network)

Each spoke is identified by a numeric index and creates distinct anti-affinity groups for each role (e.g., `spoke-0-masters-anti`, `spoke-0-infra-anti`, `spoke-0-workers-anti`). Spokes are placed across available hypervisors in the `[hv]` group defined in `ansible/inventory/cloud##.local`.

### IaaS Resource & IPAM Scheduling Loop

   1. Ansible reads the spoke configuration vars from `ansible/vars/acm-spoke.yml` and the hypervisor inventory from the [hv] group.
   2. For each spoke (identified by index), Ansible builds or queries explicit Anti-Affinity Groups in OpenNebula for each VM role (masters, infra, workers).
   3. Ansible constructs raw text templates containing resource boundaries (vCPU, Memory, Disk footprint size) for each role, tagging them with the SPOKE_GROUP and ROLE_AFFINITY_GROUP keys. It specifies connectivity against OpenNebula's managed flat network pool.
   4. Ansible triggers template instantiation with the --hold flag for each required VM across all spokes.
   5. OpenNebula claims IP addresses and MACs from the network layer, evaluates available hypervisor capacity, respects anti-affinity rules, opens an SSH pipeline to the chosen KVM host, and defines unbooted (HOLD) domains via virsh define.

## Redfish/ACM Handoff Loop

   1. Sushy-Emulator on the target hypervisor monitors /var/run/libvirt/libvirt-sock:Z. The instant OpenNebula defines the dormant domain (named under the structural format one-<ID>), sushy reflects it in its Redfish API directory path: /redfish/v1/Systems/one-<ID>.
   2. Ansible executes a structured JSON query (onevm list --json), processes the output variables, and formats a mapping of the VM name, host location, assigned IP/MAC addresses, and the specific Redfish URI string.
   3. **[Out of Scope - ACM-Load-Deploy Project]** The ACM-Load-Deploy project consumes the OpenNebula resources by reading the VM inventory and topology data. It uses the Redfish endpoints and VM placement information to generate ACM ClusterInstance Custom Resource (CR) manifests.
   4. **[Out of Scope - ACM-Load-Deploy Project]** The ACM-Load-Deploy project applies the generated ClusterInstance CRs to the ACM hub cluster. ACM reads the CRs, targets the sushy endpoints on the designated KVM nodes, attaches the OpenShift discovery payloads to the VirtualMedia mounts, and changes the machine power states to trigger network installations.

------------------------------
## 4. Technical Reference Configurations

### Sample Spoke Configuration (ansible/vars/acm-spoke.yml)

```yaml
# Number of ACM spoke clusters to create
num_spokes: 2

# VM configuration per spoke
spoke_vm_config:
  master:
    count: 3
    vcpu: 4
    memory: 16384
    disk_size: 122880
  infra:
    count: 1
    vcpu: 4
    memory: 16384
    disk_size: 122880
  worker:
    count: 2
    vcpu: 4
    memory: 16384
    disk_size: 122880

# Network pool name in OpenNebula
network_pool: "acm-flat-net"

# VM arch and boot order
os_arch: "x86_64"
boot_order: "network,hd"
```

With this configuration:
- Total spokes: 2
- VMs per spoke: 6 (3 masters + 1 infra + 2 workers)
- Total VMs to create: 12

### OpenNebula Raw VM Payload Structure

Example for a master node in spoke-0:

```
CPU    = "4.0"
VCPU   = "4"
MEMORY = "16384"
OS     = [ ARCH = "x86_64", BOOT = "network,hd" ]
DISK   = [ TYPE = "file", SIZE = "122880", TARGET = "vda", DRIVER = "raw" ]
NIC    = [ NETWORK = "acm-flat-net" ]
ROLE_AFFINITY_GROUP = "spoke-0-masters-anti"
SPOKE_GROUP         = "spoke-0"
```

## Sushy-Emulator Container Run Command (RHEL 9.4)

sudo setsebool -P container_manage_cgroup true

podman run -d \
  --name sushy-redfish-emulator \
  --restart always \
  --network host \
  --privileged \
  -v /var/run/libvirt/libvirt-sock:/var/run/libvirt/libvirt-sock:Z \
  -e SUSHY_EMULATOR_LIBVIRT_URI="qemu:///system" \
  -e SUSHY_EMULATOR_LISTEN_PORT="8000" \
  -e SUSHY_EMULATOR_VM_FEATURES="vmedia" \
  quay.io/metal3-io/sushy-tools:latest

------------------------------
## 5. Phased Implementation Plan

### Phase 1: Bare-Metal Host Bootstrapping

* Configure a consistent Linux bridge network setup (`br0`) on all RHEL 9.4 KVM nodes in the `[hv]` inventory group.
* Enable the required SELinux container boolean on all compute hosts (container_manage_cgroup).
* Run the automated playbook to mount and launch the containerized sushy-redfish-emulator image on every hypervisor in the `[hv]` inventory group.

### Phase 2: Core Control Plane Instantiation

* Spin up the opennebula-frontend Podman container on the Bastion node using host networking.
* Verify connectivity by registering all hypervisors from the [hv] inventory group into the OpenNebula core host pool via the OpenNebula CLI (onehost create).
* Establish the global IPAM framework by creating the Virtual Network resource mapping your exact flat subnet range.

### Phase 3: Ansible Orchestration & VM Allocation Development

### hv-acm-setup.yml Playbook

A new orchestration playbook that drives hypervisor setup for ACM spoke VM allocation:

```yaml
---
- name: Setup hypervisors for ACM spoke VM hosting
  hosts: hv
  vars_files:
  - vars/lab.yml
  - vars/hv.yml
  - vars/acm-spoke.yml
  roles:
  - hv-install
  - hv-network
  - hv-libvirt
  - role: hv-setup-disk2
    when: hostvars[inventory_hostname].disk2_enable | default(false) | bool
  - hv-acm-coredns
```

Usage: `ansible-playbook -i ansible/inventory/cloud##.local ansible/hv-acm-setup.yml`

### hv-acm-coredns Role

A new role that dynamically generates and syncs DNS configuration based on OpenNebula VM allocations:

**Role Tasks (`ansible/roles/hv-acm-coredns/tasks/main.yml`):**
1. Query OpenNebula for all allocated VMs: `onevm list --json`
2. Parse JSON output to extract:
   - VM ID and name (one-<ID>)
   - Assigned host location
   - Assigned IP address and MAC
   - SPOKE_GROUP and ROLE_AFFINITY_GROUP tags
3. Generate Corefile.hosts template dynamically using Jinja2
4. Deploy generated Corefile.hosts to all hypervisors via synchronize

**Generated DNS Entries Structure:**
- Format: `<spoke_ip> <spoke_hostname>`
- Spoke hostname convention: `spoke-<index>-<role>-<instance>.{{ base_dns_name }}`
  - Example: `spoke-0-master-0.example.com`, `spoke-0-worker-1.example.com`
- Wildcard app routing: `*.apps.spoke-<index>.{{ base_dns_name }}` pointing to first master/infra node per spoke

**Templates:**
- `Corefile.j2`: Simple configuration that imports the dynamically generated Corefile.hosts
- `Corefile.hosts.j2`: Generated from OpenNebula VM data with all spoke VMs and their DNS entries

**Key Features:**
- Queries OpenNebula instead of relying on static inventory
- Always executes (no conditional flag) since DNS is required for VM communication
- Syncs generated configuration to all [hv] group hypervisors for consistency
- Derives VM naming from spoke index and role, not pre-defined patterns

### Ansible Orchestration Tasks

* Read spoke configuration from `ansible/vars/acm-spoke.yml` to determine spoke count and VM topology (master/infra/worker counts per role).
* Target hypervisors using the [hv] inventory group from `ansible/inventory/cloud##.local`.
* Construct the Ansible automation tasks in `prepare-acm-vms.yml` that declare OpenNebula anti-affinity groups per role per spoke and process conditional VM templates using --hold.
* Iterate across all spokes and VM roles to generate the required VM instantiation sequence based on `num_spokes` and `spoke_vm_config`.
* Build the Jinja2 text parsing framework within Ansible to filter the JSON data output from OpenNebula.
* Map out the schema that correlates database auto-increment IDs with Redfish endpoints (/redfish/v1/Systems/one-{{ ID }}).

### Phase 4: Integration Validation & Lifecycle End-to-End Test

Execute the following manual steps to validate the implementation:

#### Step 1: Execute the Ansible allocation sequence

```bash
# Prepare and allocate VMs across hypervisors (in HOLD state, not provisioned)
ansible-playbook -i ansible/inventory/cloud##.local ansible/prepare-acm-vms.yml
```

#### Step 2: Verify VM allocation and anti-affinity

```bash
# Query OpenNebula to list all created VMs with their placement
onevm list --json | jq ‘.VM[] | {id, name, host, user_template}’

# Verify anti-affinity compliance: master VMs should be on different hosts
onevm list --json | jq ‘.VM[] | select(.user_template.ROLE_AFFINITY_GROUP | contains("masters")) | {id, name, host}’
```

#### Step 3: Verify sushy-emulator discovery on hypervisors

For each hypervisor in the `[hv]` group, execute:

```bash
# Check sushy container is running
ssh root@<hypervisor_ip> ‘podman ps | grep sushy’

# Query sushy REST API to verify one-<ID> VMs are discoverable
curl -s http://<hypervisor_ip>:8000/redfish/v1/Systems/ | jq ‘.Members[] | .["@odata.id"]’

# Verify specific VM is listed (example: one-123)
curl -s http://<hypervisor_ip>:8000/redfish/v1/Systems/one-123 | jq ‘.’
```

#### Step 4: Verify VMs are in HOLD state on libvirt

```bash
# SSH to hypervisor and check VM state
ssh root@<hypervisor_ip> ‘virsh list --all | grep one-’

# Verify a specific VM definition exists but is not running
ssh root@<hypervisor_ip> ‘virsh dumpxml one-123 | head -20’
```

#### Step 5: Verify output data for ACM-Load-Deploy handoff

```bash
# Output the complete VM inventory for consumption by ACM-Load-Deploy project
onevm list --json > ansible/generated/opennebula-vm-inventory.json

# Verify the JSON output contains all required fields for handoff
jq '.VM[] | {id, name, host, user_template: {SPOKE_GROUP, ROLE_AFFINITY_GROUP}, template: {NIC}}' \
  ansible/generated/opennebula-vm-inventory.json

# Document the spoke topology and Redfish endpoints for the ACM-Load-Deploy project
echo "=== Redfish Endpoints ===" && \
  jq -r '.VM[] | "\(.user_template.SPOKE_GROUP): \(.host) -> http://\(.host):8000/redfish/v1/Systems/one-\(.id)"' \
  ansible/generated/opennebula-vm-inventory.json
```

#### Step 6: ACM-Load-Deploy project integration (out of scope here)

The ACM-Load-Deploy project will:
- Consume the OpenNebula VM inventory (`opennebula-vm-inventory.json`)
- Read the Redfish endpoint mappings and VM placement information
- Generate ACM ClusterInstance Custom Resource (CR) manifests
- Apply the CRs to the ACM hub cluster to begin cluster provisioning

------------------------------
## 6. Guidance for AI Agent Execution
When writing code or configurations to implement this plan, ensure the following constraints are met:

   1. Never write loops that expect an OpenNebula agent/daemon on compute nodes. Compute nodes remain agentless over SSH.
   2. Never configure an OpenNebula Image registry. VMs must use raw unprovisioned allocations (TYPE = "file") and local empty disks.
   3. Always use the :Z suffix on libvirt socket volume mounts to prevent RHEL 9.4 SELinux AVC denials inside the Podman run instructions.
   4. Ensure sushy endpoints utilize the accurate OpenNebula standard identifier prefix (one- concatenated with the raw system VM database ID string) to preserve resource alignment.

