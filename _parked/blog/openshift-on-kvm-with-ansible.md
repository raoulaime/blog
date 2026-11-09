---
title: "Deploying OpenShift on KVM with Ansible and the agent-based installer"
date: 2026-11-08
description: "A role that builds single-node and multi-node OpenShift clusters as KVM guests from two YAML lists, with the templates, the VM definition, the install watchdog and the traps that cost me hours."
image: /img/og-ocp-kvm-architecture.png
categories: ["Homelab", "OpenShift", "Automation"]
tags: ["openshift", "ansible", "kvm", "libvirt", "agent-installer", "sno"]
---

I rebuild my OpenShift clusters often. A single-node cluster for experiments, a hub and a spoke for RHACM, sometimes a 3+2 cluster. Doing that by hand means the same twenty commands and the same typos. So the whole thing is one Ansible role, driven by two lists of clusters.

This post walks through how it works, with the real code, simplified. IPs, MACs and domains are examples.

## Prerequisites

| What | Needed | Why |
|---|---|---|
| KVM host | RHEL, CentOS Stream or Fedora with libvirt, VT-x/AMD-V enabled, x86-64-v2 CPU (v3 or better for RHEL 10 guests) | Runs the cluster nodes as guests. |
| Capacity for one SNO | 10 vCPU, 48 GiB RAM, 120 GB disk on fast storage (SSD or NVMe) | A single-node cluster with a few operators. |
| Capacity for 3+2 | 3 masters at 8 vCPU / 32 GiB / 120 GB, 2 workers at 4 vCPU / 16 GiB / 120 GB | Compact control plane plus workers. |
| Libvirt storage pool | `/var/lib/libvirt/images`, thin provisioned, SELinux labels left alone | Disks and the agent ISO. |
| Network | A bridge (`br-vm`) on the machine VLAN, one static address per node, VIPs for the API and ingress on multi-node | The nodes keep fixed addresses. |
| DHCP | One reservation per MAC on the real DHCP server | The installer races the static config on first boot, the reservation is the safety net. |
| DNS | `api.<cluster>.<domain>`, `api-int` and `*.apps.<cluster>.<domain>` pointing at the node or the VIPs | The installer fails without them. Backends: dnsmasq or Technitium. |
| Outbound internet | `quay.io`, `registry.redhat.io`, `api.openshift.com`, `mirror.openshift.com`, `cloud.redhat.com` | Images and the installer. Use a mirror registry if the lab is offline. |
| Red Hat account | A pull secret and an SSH public key | Placed in the install config. |
| Control node | Ansible 2.16+ with `community.libvirt`, `community.general`, `ansible.posix`, SSH access with sudo to the KVM host | Runs everything. |

## The big picture

![Architecture: the Ansible control node renders the installer files, builds the agent ISO and creates the VMs on a KVM host; DNS, DHCP reservations and credential sinks sit around it](/img/ocp-kvm-architecture.svg)

<p class="figsrc">Editable source: <a href="/img/ocp-kvm-architecture.excalidraw" download>ocp-kvm-architecture.excalidraw</a>, opens in <a href="https://excalidraw.com">excalidraw.com</a>.</p>

The control node does everything. It renders two installer files, turns them into one ISO, defines the VMs on the KVM host, keeps DNS and DHCP in step, and at the end asks the cluster whether the nodes really have the addresses I planned.

## The goal

I want to write this:

```yaml
ocp_sno_clusters:
  - name: dev-sno
    ip: 192.168.30.41
    mac: "52:54:00:0c:af:41"
    vcpus: 12            # optional: vcpus, memory_mb, disk_gb, firmware, autostart
```

and run this:

```bash
ansible-playbook playbook_linux_ocp_cluster.yml \
  -e '{"ocp_limit": ["dev-sno"]}'
```

and get a working cluster, with DNS records, stored credentials and a verified node address at the end. A multi-node cluster is the same idea, with three masters, any number of workers and two VIPs:

```yaml
ocp_multinode_clusters:
  - name: ocp
    api_vip: 192.168.30.20
    ingress_vip: 192.168.30.29
    masters:
      - { name: ocp-mr1, ip: 192.168.30.21, mac: "52:54:00:0c:9f:21" }
      - { name: ocp-mr2, ip: 192.168.30.22, mac: "52:54:00:0c:9f:22" }
      - { name: ocp-mr3, ip: 192.168.30.23, mac: "52:54:00:0c:9f:23" }
    workers:
      - { name: ocp-wr1, ip: 192.168.30.24, mac: "52:54:00:0c:9f:24", vcpus: 4 }
```

Everything that differs between environments is data (the network, the DNS backend, where credentials go), not a copy of the role.

## Code structure and variables

```text
playbook_linux_ocp_cluster.yml
roles/role_linux_ocp_cluster/
├── README.md
├── defaults/main.yml
├── handlers/main.yml
├── meta/main.yml
├── tasks/
│   ├── main.yml            # imports the stages in order, one tag each
│   ├── 00-validate.yml     # refuse bad input before touching anything
│   ├── 01-preflight.yml    # host, pool, bridge, tools
│   ├── 02-dns.yml          # records through the chosen backend
│   ├── 03-configs.yml      # install-config and agent-config
│   ├── 04-image.yml        # one agent ISO per cluster
│   ├── 05-vms.yml          # virt-install
│   ├── 06-wait.yml         # install with a watchdog
│   ├── 07-credentials.yml  # kubeconfig and admin password to the sinks
│   ├── 08-decommission.yml # only with ocp_decommission_confirm
│   ├── 09-verify.yml       # nodes kept their addresses
│   ├── 10-unifi.yml        # DHCP reservations
│   ├── dns/                # dnsmasq.yml, technitium.yml, technitium_cluster.yml
│   └── sink/               # fetch.yml, vault.yml, vault_cluster.yml
└── templates/
    ├── install-config.yaml.j2
    ├── agent-config.yaml.j2
    ├── dnsmasq.conf.j2
    └── wait-watchdog.sh.j2
```

```yaml
# group_vars/kvm.yml
ocp_version: "4.20"
ocp_base_domain: example.com
ocp_machine_cidr: 192.168.30.0/24
ocp_gateway: 192.168.30.1
ocp_dns_server: 192.168.30.6
ocp_dns_backend: technitium
ocp_credentials_sinks: [vault]
```

| Variable | Default | Meaning |
|---|---|---|
| `ocp_version` | none | OpenShift release to install. |
| `ocp_base_domain` | none | Base domain of the clusters. |
| `ocp_node_bridge`, `ocp_node_iface` | `br-vm`, `enp1s0` | Bridge on the host and NIC name inside the node. |
| `ocp_machine_cidr`, `ocp_gateway`, `ocp_dns_server` | none | Node network. |
| `ocp_default_vcpus`, `ocp_default_memory_mb`, `ocp_default_disk_gb` | `8`, `32768`, `120` | Used when a node does not set its own. |
| `ocp_default_cluster_network_cidr`, `ocp_default_service_network_cidr` | `10.128.0.0/14`, `172.30.0.0/16` | Pod and service networks. |
| `ocp_image_pool`, `ocp_install_root` | `/var/lib/libvirt/images`, `/var/lib/ocp` | Disks and ISOs, installer working files. |
| `ocp_wait_bootstrap_seconds`, `ocp_wait_install_seconds` | `5400` | Time allowed before the watchdog gives up. |
| `ocp_os_variant`, `ocp_vm_firmware`, `ocp_vm_autostart` | `rhel9.6`, `bios`, `false` | VM settings. |
| `ocp_dns_backend` | `dnsmasq` | Where the DNS records are written. |
| `ocp_credentials_sinks` | `[]` | Where `kubeconfig` and the admin password go. |
| `ocp_limit` | `[]` | Build only the named clusters. |
| `ocp_decommission_confirm` | `false` | Must be true to destroy a cluster. |
| `ocp_sno_clusters`, `ocp_multinode_clusters` | none | The clusters themselves, as shown above. |

## Why the agent-based installer

The agent-based installer fits KVM well. You give it two files, it produces one bootable ISO, and every node boots from that ISO. No bootstrap machine, no load balancer for a single node, no API calls to a hypervisor. The role only uses what the installer documents: an `install-config.yaml` and an `agent-config.yaml`. It adds nothing to the cluster itself.

The role is a list of stages, each one a tag you can run alone:

![The nine stages of the role, with decommission kept apart](/img/ocp-stages.svg)

```text
validate -> preflight -> dns -> configs -> image -> vms -> wait -> credentials -> verify
```

`decommission` is separate and never runs unless asked for.

## Stage 1: validate before touching anything

Mistakes in a cluster table are cheap to catch early and expensive to catch after a 40-minute install. The first stage asserts the shape of the data:

```yaml
- name: Require the fields of every single-node cluster
  ansible.builtin.assert:
    that:
      - item.name is defined
      - item.name is match('^[a-z0-9]([-a-z0-9]*[a-z0-9])?$')
      - item.ip is defined
      - item.mac is defined
    fail_msg: "single-node cluster {{ item.name | default('?') }}: needs a DNS-label name, an ip and a mac."
    quiet: true
  loop: "{{ ocp_sno_clusters }}"
  loop_control:
    label: "{{ item.name | default('?') }}"
```

The multi-node version also requires exactly three masters. The playbook itself refuses to run without a `--limit`, because this role creates and destroys VMs.

## Stage 2: the two installer files

`install-config.yaml` is a template. The only branch is the platform: a single node uses `none`, a multi-node cluster uses `baremetal` with VIPs.

```yaml
apiVersion: v1
baseDomain: {{ item.base_domain }}
metadata:
  name: {{ item.name }}
controlPlane:
  name: master
  replicas: {{ item.masters }}
compute:
- name: worker
  replicas: {{ item.workers }}
networking:
  networkType: OVNKubernetes
  clusterNetwork:
  - cidr: {{ item.cluster_network_cidr }}
    hostPrefix: {{ item.cluster_network_host_prefix }}
  serviceNetwork:
  - {{ item.service_network_cidr }}
  machineNetwork:
  - cidr: {{ item.machine_cidr }}
platform:
{% if item.platform == 'baremetal' %}
  baremetal:
    apiVIPs:
    - {{ item.api_vip }}
    ingressVIPs:
    - {{ item.ingress_vip }}
{% else %}
  none: {}
{% endif %}
pullSecret: '{{ ocp_pull_secret }}'
sshKey: '{{ ocp_ssh_pubkey }}'
```

`agent-config.yaml` carries the **static network configuration of each node**, in NMState format. This is what lets a node come up with a fixed address instead of waiting for DHCP:

```yaml
apiVersion: v1beta1
kind: AgentConfig
metadata:
  name: {{ item.name }}
rendezvousIP: {{ item.rendezvous_ip }}
hosts:
{% for node in item.nodes %}
  - hostname: {{ node.name }}
    role: {{ node.role }}
    interfaces:
      - name: {{ item.iface }}
        macAddress: {{ node.mac }}
    networkConfig:
      interfaces:
        - name: {{ item.iface }}
          type: ethernet
          state: up
          ipv4:
            enabled: true
            address:
              - ip: {{ node.ip }}
                prefix-length: {{ item.machine_cidr.split('/')[1] }}
            dhcp: false
      dns-resolver:
        config:
          server: [{{ item.dns_server }}]
          search: [{{ item.base_domain }}]
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: {{ item.gateway }}
            next-hop-interface: {{ item.iface }}
{% endfor %}
```

A single node is its own rendezvous host. For a multi-node cluster the first master is.

Both files are rendered with `force: false` and a backup copy is taken, because **`openshift-install` consumes them**. Once the ISO exists, the originals are gone from the install directory. Rendering again would silently produce files that no longer match the ISO.

```yaml
- name: Render install-config.yaml for each cluster
  become: true
  ansible.builtin.template:
    src: install-config.yaml.j2
    dest: "{{ item.install_dir }}/cluster/install-config.yaml"
    mode: "0600"
    force: false
  loop: "{{ _ocp_to_render }}"
  loop_control:
    label: "{{ item.name }}"
  no_log: true
```

Note `no_log: true`: the pull secret is in that file.

## Stage 3: one ISO per cluster

```yaml
- name: Generate each cluster's agent ISO (downloads the RHCOS image on first use)
  become: true
  ansible.builtin.command:
    cmd: /usr/local/bin/openshift-install agent create image --dir cluster --log-level info
    chdir: "{{ item.install_dir }}"
    creates: "{{ item.install_dir }}/cluster/agent.x86_64.iso"
  loop: "{{ _ocp }}"
  loop_control:
    label: "{{ item.name }}"
```

`creates:` makes the stage idempotent. If a previous run died halfway and left a partial installer state with no ISO, a small task before it deletes `.openshift_install_state.json` so the next attempt starts clean.

## Stage 4: the VMs

I do not use `community.libvirt.virt_install` here, on purpose. In community.libvirt 2.0.0 its `events` option is broken (the dict is passed to `virt-install` as a Python repr), and the RHCOS install **reboots the VM**, so I need `on_reboot=restart`. A plain `virt-install` command is more reliable. Listing, destroying and undefining still use `community.libvirt.virt`.

```yaml
- name: List the VMs already defined on this hypervisor
  become: true
  community.libvirt.virt:
    command: list_vms
  register: _ocp_defined_vms

- name: Define and start each node VM that does not exist yet
  become: true
  ansible.builtin.command:
    cmd: >
      virt-install --connect qemu:///system
      --name {{ item.1.name }}
      --memory {{ item.1.memory_mb | default(item.0.memory_mb) }}
      --vcpus {{ item.1.vcpus | default(item.0.vcpus) }}
      --cpu host-passthrough
      --disk path={{ ocp_image_pool }}/{{ item.1.name }}.qcow2,size={{ item.1.disk_gb | default(item.0.disk_gb) }},bus=virtio,format=qcow2,cache=none
      --network bridge={{ item.0.bridge }},model=virtio,mac={{ item.1.mac }}
      --cdrom {{ item.0.install_dir }}/cluster/agent.x86_64.iso
      --os-variant {{ ocp_os_variant }}
      --boot {{ 'uefi,' if item.0.firmware == 'efi' else '' }}hd,cdrom
      --events on_reboot=restart,on_poweroff=destroy
      --graphics vnc
      --noautoconsole
  loop: "{{ _ocp | subelements('nodes') }}"
  loop_control:
    label: "{{ item.0.name }}/{{ item.1.name }}"
  when: item.1.name not in _ocp_defined_vms.list_vms
  changed_when: true
```

Two details matter:

- **`subelements('nodes')`** turns "clusters with nodes" into one loop over every node, while keeping the cluster in `item.0`. That is how a node can override its cluster's `vcpus` with a simple `default()`.
- **`when: ... not in list_vms`** keys idempotence on the libvirt domain. Re-running the play does not touch a VM that exists.

`--boot hd,cdrom` is the quiet hero. On first boot the disk is empty, so the VM boots the ISO. After the install writes RHCOS to disk, the same VM boots from disk. No manual media change.

## Stage 5: waiting, with a watchdog

This is the part I would not have predicted. During the install, a node writes RHCOS to its disk and then **powers off instead of rebooting**, depending on how the VM is defined. If nobody restarts it, the install waits until it times out. My fix is a small supervised wait script, rendered per cluster:

```bash
#!/bin/bash
# Usage: wait-watchdog.sh bootstrap-complete|install-complete
set -uo pipefail
cd "{{ item.install_dir }}"
vms=({% for node in item.nodes %}{{ node.name }} {% endfor %})
virsh_c="virsh --connect qemu:///system"
logfile=$(mktemp)
/usr/local/bin/openshift-install agent wait-for "$1" --dir cluster --log-level info >"$logfile" 2>&1 &
pid=$!
trap 'kill "$pid" 2>/dev/null' EXIT
trap 'exit 143' TERM
while kill -0 "$pid" 2>/dev/null; do
  for vm in "${vms[@]}"; do
    state=$($virsh_c domstate "$vm" 2>/dev/null | tr -d '\r' || true)
    if [ "$state" = "shut off" ]; then
      device=$($virsh_c domblklist "$vm" --details 2>/dev/null \
        | awk '$1 == "cdrom" && $4 != "-" {print $3; exit}' || true)
      [ -n "$device" ] && $virsh_c change-media "$vm" "$device" --eject --config || true
      $virsh_c start "$vm" || true
    fi
  done
  sleep 10
done
wait "$pid"
result=$?
cat "$logfile"
rm -f "$logfile"
exit "$result"
```

It runs `openshift-install agent wait-for` in the background and, every ten seconds, looks for a node that is "shut off". If it finds one, it ejects the installer ISO and starts it. The `trap` makes sure that killing the script, which the decommission does, does not leave the installer running.

Ansible launches it asynchronously and polls, so the play survives a long install and several clusters install in parallel:

```yaml
- name: Start waiting for every bootstrap to complete
  become: true
  ansible.builtin.command: "{{ item.install_dir }}/wait-watchdog.sh bootstrap-complete"
  async: "{{ ocp_wait_bootstrap_seconds }}"     # 5400
  poll: 0
  register: _ocp_bootstrap_jobs
  loop: "{{ _ocp }}"

- name: Wait for every bootstrap to complete
  become: true
  ansible.builtin.async_status:
    jid: "{{ item.ansible_job_id }}"
  register: _ocp_bootstrap_result
  until: _ocp_bootstrap_result.finished
  retries: "{{ (ocp_wait_bootstrap_seconds / 30) | int }}"
  delay: 30
  loop: "{{ _ocp_bootstrap_jobs.results }}"
```

Between the two waits, a task ejects the ISO from every node so that a later reboot cannot boot the installer again. Then the same pattern runs for `install-complete`.

## Stage 6: prove the nodes kept their addresses

The last stage is the one that earned its place the hard way. After the install reboot, RHCOS can **fall back to DHCP** and ignore the static network configuration. The cluster still installs, so nothing fails. You just find out later that a node has a different address than your DNS says.

So the role asks the cluster and compares with the table:

```yaml
- name: Read the node addresses of each cluster
  become: true
  ansible.builtin.command: >-
    /usr/local/bin/oc --kubeconfig {{ item.install_dir }}/cluster/auth/kubeconfig get nodes
    -o jsonpath='{range .items[*]}{.metadata.name}={.status.addresses[?(@.type=="InternalIP")].address}{"\n"}{end}'
  register: _ocp_nodes
  loop: "{{ _ocp }}"
  changed_when: false

- name: Require every node to have the address from the cluster table
  ansible.builtin.assert:
    that: _unexpected | length == 0
    fail_msg: >-
      {{ item.item.name }}: nodes not on their planned name=address (the agent installer dropped
      their static network config): {{ _unexpected | join(', ') }}
    quiet: true
  vars:
    _wanted: >-
      {{ item.item.nodes | map(attribute='name')
         | zip(item.item.nodes | map(attribute='ip')) | map('join', '=') | list }}
    _unexpected: "{{ _wanted | difference(item.stdout_lines) }}"
  loop: "{{ _ocp_nodes.results }}"
```

The fix for the cause is **not** in the cluster. The role has an optional stage that creates a fixed-IP reservation per MAC on the DHCP server (a UniFi controller in my lab) before the VMs exist. If RHCOS does fall back to DHCP, it gets the same address anyway. I deliberately did not "fix" it with a MachineConfig that pushes NMState: that fights the installer and is harder to reason about than a reservation.

## Rebuild and decommission

Because the cluster is data, rebuilding is cheap:

```bash
# Decommission, then build again
ansible-playbook playbook_linux_ocp_cluster_rebuild.yml \
  -e ocp_decommission_confirm=true \
  -e '{"ocp_limit": ["dev-sno"]}'
```

The decommission destroys the VMs, undefines them with their disks and NVRAM, deletes the install directory and removes the DNS records and stored credentials. It refuses to run without `ocp_decommission_confirm=true`, and it only touches the clusters in `ocp_limit`. A destructive stage should be hard to trigger by accident.

## Credentials and DNS are pluggable

The first admin credentials have to go somewhere. The role has "sinks":

```yaml
ocp_credentials_sinks: [fetch, vault]   # any combination, or none
```

`fetch` copies `kubeconfig` and the admin password to the control node. `vault` writes them to HashiCorp Vault. DNS works the same way: `ocp_dns_backend: dnsmasq` for a lab on one host, or `technitium` to create the records through an API. The `api.<cluster>`, `api-int` and `*.apps` names are all it needs.

## What I would tell a client

- **Treat the cluster as data.** Two lists in Git beat a wiki page of commands. A rebuild becomes a pull request.
- **Validate first, verify last.** The cheap check at the start and the honest check at the end are the two stages that save time.
- **Keep installer inputs immutable once consumed.** `force: false` and a backup are small habits that prevent confusing failures.
- **Prefer a boring fix for a boring cause.** A DHCP reservation is easier to reason about than a clever cluster-side patch.
- **Put guardrails on destructive actions**: a mandatory limit, an explicit confirmation variable, scope by cluster name.

The same pattern works on bare metal, with the hypervisor stages replaced by a BMC. The installer files and the verification do not change.
