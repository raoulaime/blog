---
title: "A two-node Technitium DNS cluster with Ansible, and the traps on the way"
date: 2026-10-23
description: "A single resolver cut the whole lab's DNS at every reboot. Building a primary and secondary cluster with Ansible taught me where rootless containers and DNS clash."
image: /img/og-dns-cluster.png
categories: ["Homelab", "Cloud Infrastructure"]
tags: ["dns", "technitium", "podman", "letsencrypt", "ansible"]
---

My lab ran on one DNS server, so every reboot of that machine cut name resolution for everything: the clusters, the CI runners, my own laptop. I replaced it with a **Technitium** cluster of one primary and one secondary, with clients and DHCP handing out both addresses. An Ansible role builds it, and the same role later became the place where every other automation creates its DNS records.

## Prerequisites

| What | Needed | Why |
|---|---|---|
| Two Linux hosts | RHEL or Fedora, 1 vCPU, 1 GiB RAM, 10 GB disk each, different machines | One primary, one secondary. A VM next to the primary defeats the point. |
| Network | Same VLAN or routed, static addresses (for example `192.168.30.6` and `192.168.30.7`) | DHCP hands both addresses to clients as DNS servers. |
| Ports between nodes | 53/tcp+udp, 5380/tcp (web and API), 53443/tcp (cluster) | Replication and the API calls. |
| Rootless Podman | Podman 4.4+ with Quadlet, `net.ipv4.ip_unprivileged_port_start=53` | The container runs as a normal user and still binds port 53. |
| Public DNS zone | A domain served by Cloudflare and an API token that can edit its DNS | The Let's Encrypt DNS-01 challenge. |
| Outbound internet | `acme-v02.api.letsencrypt.org`, the Cloudflare API, `1.1.1.1:53` | Certificate issue and renewal. |
| Control node | Ansible 2.16+ with `ansible.posix` and `containers.podman` | Runs the role. |
| Secrets | Technitium admin password, Cloudflare token (Ansible Vault or HashiCorp Vault) | Never in the inventory in clear. |

## What it looks like

![Clients and DHCP use both nodes; the primary and secondary replicate through a catalog zone; Let's Encrypt certificates come from a DNS-01 challenge on Cloudflare; a weekly job renews them](/img/dns-cluster.svg)

<p class="figsrc">Editable source: <a href="/img/dns-cluster.excalidraw" download>dns-cluster.excalidraw</a>, opens in <a href="https://excalidraw.com">excalidraw.com</a>.</p>

## Code structure and variables

```text
role_linux_technitium_cluster/
├── defaults/main.yml        # every variable below, with its default
├── meta/main.yml
└── tasks/
    ├── main.yml             # order: prereqs, container, cluster, certificate, records
    ├── prereqs.yml          # sysctl, firewalld ports, linger, resolved stub
    ├── cluster.yml          # init on the primary, join on the secondary
    ├── certificate.yml      # lego + Cloudflare DNS-01, renewal timer
    ├── records.yml          # A records from the inventory, reverse zones
    ├── record.yml           # one record (included in a loop)
    ├── cname.yml
    └── sso.yml              # optional OIDC login
```

The inventory only says which node plays which role:

```yaml
# group_vars/dns.yml
technitium_cluster_domain: dns-cluster.example.com
technitium_acme_email: ops@example.com
technitium_cloudflare_zone: example.com
technitium_records_zone: home.example.com

# host_vars/dns-01.yml
technitium_cluster_role: primary
technitium_node_ips: [192.168.30.6]

# host_vars/dns-02.yml
technitium_cluster_role: secondary
technitium_primary_url: https://dns-01.dns-cluster.example.com:53443
technitium_primary_ip: 192.168.30.6
```

| Variable | Default | Meaning |
|---|---|---|
| `technitium_cluster_role` | `secondary` | `primary` initialises the cluster, `secondary` joins it. |
| `technitium_cluster_domain` | none | Name under which the nodes see each other. |
| `technitium_api_url` | `http://127.0.0.1:5380` | API of the local node. |
| `technitium_cluster_port` | `53443` | HTTPS port used between nodes. |
| `technitium_node_ips` | the default IPv4 | Addresses published for this node. |
| `technitium_primary_url`, `technitium_primary_ip` | empty | Where a secondary finds the primary. |
| `technitium_firewalld_ports` | `53/tcp, 53/udp, 5380/tcp, 53443/tcp` | Opened by the prereqs. |
| `technitium_acme_directory` | Let's Encrypt production | Switch to staging to test. |
| `technitium_tls_renew_days` | `30` | Renew when fewer days remain. |
| `technitium_records_enabled` | `true` | Create A and PTR records from the inventory. |
| `technitium_records_zone`, `technitium_records_ttl` | none, `3600` | Zone and TTL of those records. |
| `technitium_records_extra`, `technitium_cnames` | `[]` | Records that are not hosts. |
| `technitium_sso_enabled` | `false` | OIDC login to the web console. |

## Building the cluster through the API

Technitium has a clustering API, so the role is a short sequence of API calls. Everything starts with a login, and every later call carries the bearer token:

```yaml
- name: Log in to the API of the node
  ansible.builtin.uri:
    url: "{{ technitium_api_url }}/api/user/login"
    method: POST
    body_format: form-urlencoded
    body:
      user: "{{ technitium_admin_user }}"
      pass: "{{ technitium_admin_password }}"
    return_content: true
  register: _technitium_login
  no_log: true
  changed_when: false
```

The role first reads the cluster state, so that running it twice does nothing the second time. On the primary node it creates the cluster:

```yaml
- name: Create the cluster on the Primary node
  ansible.builtin.uri:
    url: "{{ technitium_api_url }}/api/admin/cluster/init"
    method: POST
    headers:
      Authorization: "Bearer {{ _technitium_login.json.token }}"
    body_format: form-urlencoded
    body:
      clusterDomain: "{{ technitium_cluster_domain }}"
      primaryNodeIpAddresses: "{{ technitium_node_ips | join(',') }}"
  when:
    - technitium_cluster_role == 'primary'
    - not _technitium_state.json.response.clusterInitialized
```

On the secondary it joins. This call replaces the settings of the node by those of the cluster, which is why the condition on `clusterInitialized` matters:

```yaml
# initJoin replaces the settings of this node by those of the cluster: run it on a new node only
- name: Join the cluster as a Secondary node
  ansible.builtin.uri:
    url: "{{ technitium_api_url }}/api/admin/cluster/initJoin"
    method: POST
    headers:
      Authorization: "Bearer {{ _technitium_login.json.token }}"
    body_format: form-urlencoded
    body:
      secondaryNodeIpAddresses: "{{ technitium_node_ips | join(',') }}"
      primaryNodeUrl: "{{ technitium_primary_url }}"
      primaryNodeIpAddress: "{{ technitium_primary_ip }}"
      primaryNodeUsername: "{{ technitium_admin_user }}"
      primaryNodePassword: "{{ technitium_admin_password }}"
  when:
    - technitium_cluster_role == 'secondary'
    - not _technitium_state.json.response.clusterInitialized
```

The last task of the role is the one I trust most: it asks the node for its state and asserts that it is initialized and in the right cluster domain.

## Lessons from building it

**Use host networking.** With published ports under rootless Podman, every peer appears to come from the container network's gateway address. NOTIFY and zone transfers are then refused and nothing replicates. Running both nodes with `Network=host` fixed it. Publishing port 53 on all addresses also hijacked queries meant for the container network's own resolver and broke name resolution between containers.

**A container cannot reach its own host's address.** My blackbox probes for the primary therefore go through `host.containers.internal`.

**Pre-existing zones are not clustered automatically.** Zones created before the cluster are not copied to the secondary until they are members of the catalog zone. The role now attaches every primary zone:

```yaml
- name: Make them members of the catalog
  ansible.builtin.uri:
    url: "{{ technitium_api_url }}/api/zones/options/set"
    method: POST
    headers:
      Authorization: "Bearer {{ _technitium_login.json.token }}"
    body_format: form-urlencoded
    body:
      zone: "{{ item.name }}"
      catalog: "cluster-catalog.{{ technitium_cluster_domain }}"
  loop: >-
    {{ _technitium_zones.json.response.zones
       | selectattr('type', 'equalto', 'Primary')
       | rejectattr('name', 'equalto', technitium_cluster_domain) | list }}
  loop_control:
    label: "{{ item.name }}"
```

**The cluster needs a real certificate.** Nodes validate each other over HTTPS, so a self-signed certificate fails. The role requests a Let's Encrypt certificate with the DNS-01 challenge through Cloudflare, so no inbound port is open. Renewal is a weekly job in Ansible Automation Platform that mails me only on error.

## One DNS for every automation

Once the cluster existed, the same role gained a records task: it creates the A and PTR records of every host in the inventory, and other roles (the OpenShift one, for example, with its `api`, `api-int` and `*.apps` records) call it instead of editing a zone by hand. Deleting a record does the reverse zone too, so nothing is left behind after a rebuild.

## Testing failover

I blocked the primary's DNS port for 90 seconds from another host. Clients were answered by the secondary in about one second. What I have not tested is a full reboot of the primary with every client running, so I treat that as unproven.

## Takeaway

Rootless containers and DNS both make assumptions about source addresses and ports. If a clustered service sees the wrong peer address, look at the network mode first. And give DNS an API-driven owner early: every other automation will need it.
