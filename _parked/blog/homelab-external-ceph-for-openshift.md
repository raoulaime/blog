---
title: "External Ceph for OpenShift: ODF in external mode, built with Ansible"
date: 2026-11-02
description: "Red Hat Ceph Storage on a single node with cephadm, consumed by OpenShift Data Foundation in external mode, with the role's key tasks and the three things that nearly stopped me."
image: /img/og-ceph-odf.png
categories: ["Homelab", "Cloud Infrastructure"]
tags: ["ceph", "odf", "openshift", "storage", "cephadm", "ansible"]
---

I wanted to practice backup and restore (OADP and CSI snapshots) against real storage, not a throwaway local volume. So I built a single-node **Red Hat Ceph Storage 9** cluster on a VM with cephadm, three thin data disks, and connected an OpenShift cluster to it with **ODF in external mode**. An Ansible role builds the Ceph side from nothing.

## Prerequisites

| What | Needed | Why |
|---|---|---|
| Ceph VM (or three) | RHEL 10 or CentOS Stream 10, 4 vCPU, 16 GiB RAM, 40 GB system disk | A 3-OSD node with `osd_memory_target` at 2 GiB sits near 10 GiB resident. |
| Data disks | Three empty virtual disks, 100 GB each, thin provisioned (`vdb`, `vdc`, `vdd`) | cephadm refuses disks that carry a filesystem or LVM signature. |
| CPU | x86-64-v3 or better for RHEL 10 guests | RHEL 10 does not boot on older CPUs. |
| Network | Same routed network as the OpenShift nodes, a static IP, an A record | The CSI drivers and the S3 clients reach it directly. |
| Ports | 22 (SSH), 3300 and 6789 (monitor), 6800-7300 (OSDs), 8080 or 443 (RGW), 8443 (dashboard), 9283 (metrics) | Open on firewalld and on any firewall between the networks. |
| Time | chrony synchronised | Monitors refuse to form quorum with a skewed clock. |
| Subscriptions | A Red Hat subscription with Ceph Storage 9 (or the community edition) and a `registry.redhat.io` service account | Source of the tools repository and the container image. |
| Public DNS zone and token | Only for the HTTPS endpoint: a Cloudflare zone and an API token | Let's Encrypt DNS-01 for the RGW and dashboard certificates. |
| OpenShift side | OpenShift Data Foundation operator installed, 4.16 or newer | Needed to create the external StorageSystem. |
| Control node | Ansible 2.16+ with `ansible.posix` and `community.general` | Runs the role over SSH. |

## The architecture

![OpenShift in ODF external mode consumes RBD block storage and an S3 endpoint from a Ceph VM; the Ceph manager exposes metrics on port 9283 and the dashboard and RGW use a Let's Encrypt certificate](/img/ceph-odf.svg)

<p class="figsrc">Editable source: <a href="/img/ceph-odf.excalidraw" download>ceph-odf.excalidraw</a>, opens in <a href="https://excalidraw.com">excalidraw.com</a>.</p>

External mode means OpenShift does not run Ceph. It only runs the CSI drivers and talks to a cluster that already exists. That is the same mechanism you would use against a production Ceph, which is why the lab is still a useful rehearsal.

## Code structure and variables

```text
role_linux_ceph/
├── README.md
├── defaults/main.yml
├── vars/main.yml            # values computed from the group (host count, failure domain)
├── tasks/
│   ├── main.yml             # imports the files below, one tag each
│   ├── 00-validate.yml      # edition, devices, hosts
│   ├── 01-prereqs.yml       # subscription, podman, lvm2, chrony, firewalld, cephadm
│   ├── 02-bootstrap.yml     # cephadm bootstrap on the first host
│   ├── 03-hosts.yml         # add the other hosts
│   ├── 04-osd.yml           # OSD spec from ceph_osd_devices
│   ├── 05-pools.yml         # RBD pools and application tags
│   ├── 06-rgw.yml, 06-rgw-tls.yml   # S3 endpoint, optional certificate
│   ├── 07-s3.yml, 07-s3-user.yml    # users, keys, bucket and quota
│   ├── 08-verify.yml        # health, OSDs up, pools exist
│   ├── 09-dashboard.yml
│   └── 10-prometheus.yml    # mgr module, required by ODF external mode
└── templates/               # osd-spec, rgw-spec, certificate scripts
```

```yaml
# group_vars/ceph.yml
ceph_edition: redhat
ceph_osd_devices: [/dev/vdb, /dev/vdc, /dev/vdd]
ceph_osd_memory_target: 2147483648
ceph_rbd_pools: [{name: ocp-rbd}]
ceph_rgw_tls: true
ceph_rgw_tls_name: s3.example.com
ceph_prometheus_enabled: true
```

| Variable | Default | Meaning |
|---|---|---|
| `ceph_group` | `ceph` | Inventory group. One host means a single node, two or more mean the host is the failure domain. |
| `ceph_edition` | `redhat` | `redhat` (RHCS 9) or `community` (upstream). They are not upgradable into each other. |
| `ceph_rhceph_image` | `registry.redhat.io/rhceph/rhceph-9-rhel10:latest` | Image for the Red Hat edition. |
| `ceph_version` | `19.2.3` | Pinned release for the community edition. |
| `ceph_osd_devices` | empty | Disks given to Ceph on every host. |
| `ceph_osd_memory_target` | `2147483648` | Bytes per OSD. |
| `ceph_pool_size`, `ceph_pool_min_size` | automatic | 3/2 from three hosts, otherwise 2/1. |
| `ceph_rbd_pools` | list | Pools for block volumes and snapshots. |
| `ceph_rgw_port`, `ceph_rgw_count` | `8080`, `1` | S3 endpoint. |
| `ceph_rgw_tls`, `ceph_rgw_tls_port` | `false`, `443` | HTTPS with a Let's Encrypt certificate. |
| `ceph_acme_staging` | `false` | Use the staging CA while testing. |
| `ceph_prometheus_enabled`, `ceph_prometheus_port` | `true`, `9283` | Metrics endpoint ODF needs. |
| `ceph_dashboard_enabled`, `ceph_dashboard_user` | `false`, `admin` | Optional web console. |

## Building the Ceph side

The role is a list of numbered task files, so `--tags` can run one stage:

```text
00-validate  01-prereqs  02-bootstrap  03-hosts  04-osd  05-pools
06-rgw  06-rgw-tls  07-s3-user  08-verify  09-dashboard  10-prometheus
```

Bootstrap is one cephadm call, made idempotent by `creates:`. The registry login goes through a file that is deleted right after, never through the command line:

```yaml
- name: Put the login of the registry in a file
  ansible.builtin.copy:
    content: "{{ {'url': 'registry.redhat.io', 'username': ceph_registry_username, 'password': ceph_registry_password} | to_json }}"
    dest: "{{ ceph_specs_dir }}/registry.json"
    mode: "0600"
  no_log: true

- name: Bootstrap the cluster
  ansible.builtin.command:
    argv: "{{ [ceph_cephadm_bin] + _ceph_bootstrap_args }}"
    creates: /etc/ceph/ceph.client.admin.keyring

- name: Remove the login file
  ansible.builtin.file:
    path: "{{ ceph_specs_dir }}/registry.json"
    state: absent
```

A single node cannot satisfy the default of three replicas, so the pool defaults are set explicitly before the first pool exists:

```yaml
- name: Set the defaults of the pools and the memory of the OSDs
  ansible.builtin.command:
    argv: "{{ _ceph_shell + ['ceph', 'config', 'set', item.who, item.key, item.value | string] }}"
  loop:
    - { who: global, key: osd_pool_default_size, value: "{{ _ceph_pool_size }}" }
    - { who: global, key: osd_pool_default_min_size, value: "{{ _ceph_pool_min_size }}" }
    - { who: osd, key: osd_memory_target, value: "{{ ceph_osd_memory_target }}" }
```

Then the RBD pools, with replicas and the RBD application tag:

```yaml
- name: Create the RBD pools
  ansible.builtin.command:
    argv: "{{ _ceph_shell + ['ceph', 'osd', 'pool', 'create', item] }}"
  loop: "{{ ceph_rbd_pools }}"

- name: Mark the pools for RBD and initialise them
  ansible.builtin.command:
    argv: "{{ _ceph_shell + ['rbd', 'pool', 'init', item] }}"
  loop: "{{ ceph_rbd_pools }}"
```

## What it provides

- an RBD pool for block storage;
- an S3 endpoint (RGW) over HTTPS with a Let's Encrypt certificate, plus a bucket for backups with a quota;
- the dashboard, also on HTTPS.

On the OpenShift side, the StorageCluster comes up Ready with an RBD storage class, an RGW storage class and a snapshot class. I verified the loop that matters: create a PVC, write data, snapshot, restore, and confirm the restored volume has the old data.

## Traps

**The Prometheus manager module is required.** ODF external needs the Ceph `prometheus` module and port 9283. If you skip the monitoring endpoint, the StorageCluster fails with "unable to retrieve monitoring-endpoint". The role enables it and opens the port:

```yaml
- name: Enable the prometheus module
  ansible.builtin.command:
    argv: "{{ _ceph_shell + ['ceph', 'mgr', 'module', 'enable', 'prometheus'] }}"
```

**Run the exporter script inside `cephadm shell`.** The script that produces the connection data for ODF needs the Ceph Python libraries, which exist in the container and not on the host.

**EC certificate keys break the dashboard.** lego generates an EC key by default. The Ceph dashboard module refuses it ("key type unsupported"), the cluster goes HEALTH_ERR and every dashboard command returns EIO. Requesting an RSA key fixes it.

**Let cloud-init decide the hostname, then use it.** cephadm names the host by its FQDN. Use `ansible_facts['nodename']` in specs, not the short hostname.

## Takeaway

Single-node Ceph is a lab setup, not a design to copy. But external mode is the same mechanism you would use against a production cluster, and the checklist above (monitoring endpoint, certificate key type, hostname) is what I would verify first.
