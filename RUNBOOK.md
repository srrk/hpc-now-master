# HPC Bootcamp Runbook

*Version 2.0 — 2026-04-24. Validated end-to-end against Rocky 9.7 /
OpenHPC 3.3 / Warewulf 4.6.4 / Slurm 24.11.5.*

A 4-hour live-demo runbook that builds a 1 head + 2 compute HPC
cluster on a 16 GB Windows 11 laptop using VirtualBox. By the end,
you've watched diskless compute nodes PXE-boot, run High
Performance Linpack across both of them, and saw the job light up a
Grafana dashboard in real time.

**Format conventions.** Every section is shaped the same way:

- *What this section does* — one-paragraph intent.
- *Prerequisites* — which earlier section must be green.
- *Steps* — numbered, copy-pasteable commands with expected output inline.
- *What just happened* — the conceptual takeaway.
- *Validation* — commands that prove the section landed.

Command blocks are ```bash```; expected output appears right after.
Lines starting with `#` are comment annotations; everything else is
meant to be pasted into a terminal literally.

**Audience.** This is written for enterprise IT engineers new to
HPC. Where HPC differs meaningfully from enterprise patterns, the
*what just happened* block calls it out. Standard Linux idioms are
assumed.

### Before you start

| Resource | Requirement |
|----------|-------------|
| Host RAM | 16 GB minimum (head VM 4 GB; each compute 4 GB; host headroom) |
| Host disk free on `C:` | 40 GB minimum (head VM disk + Rocky box cache + VBox VM dirs) |
| CPU | 4+ physical cores (head 2 vCPU; each compute 2 vCPU) |
| Time budget | ~50 minutes end-to-end (Section 08's image build is ~10 min, dominates) |
| Internet | Required first run (~2 GB downloads — Rocky box, OpenHPC, Grafana) |

To start over from any state, see the [Cleanup / start over](#cleanup-start-over) appendix at the end.

### Required files — check before you start

Every learner needs these six files on disk at the **repository root**
before the demo starts. They're in the bootcamp starter zip (or
`git clone`).

| File | Purpose |
|------|---------|
| `Vagrantfile` | head-VM definition (4 GB / 2 vCPU / 40 GB disk, two internal networks, NAT port forwards for 3000/9090) |
| `rockylinux-9.7-primary.json` | local metadata override pointing Vagrant at the Rocky 9.7 `.box` artifact |
| `scripts/create-compute-vm.sh` | creates the diskless compute VMs via `VBoxManage` (used in §08) |
| `scripts/grafana-hpc-cluster.json` | Grafana dashboard definition (used in §10) |
| `scripts/cleanup.sh` | one-shot teardown script (used in the Cleanup appendix) |
| `RUNBOOK.md` / `RUNBOOK.html` | this document |

Quick check from the project directory:

```bash
ls Vagrantfile rockylinux-9.7-primary.json \
   scripts/create-compute-vm.sh scripts/grafana-hpc-cluster.json scripts/cleanup.sh \
   RUNBOOK.md
```

All six should list without error.

### Section snapshots (optional but recommended)

After each section's **Validation** passes, snapshot the head VM so
you can roll back to any known-good checkpoint without redoing
earlier sections. Compute nodes don't need snapshots — they're
diskless and re-PXE from the Warewulf image is the equivalent.

```bash
# From the project directory, after e.g. Section 03 lands clean:
vagrant snapshot save s03

# To list what's saved:
vagrant snapshot list

# To roll back to the Section 03 checkpoint:
vagrant snapshot restore s03
```

Typical pattern during a walkthrough: save `s02`, `s03`, `s04`, …,
`s07` at each validation boundary. Section 08 and later involve
compute VMs too, which aren't captured — if Section 08 goes wrong,
roll back the head to `s07` and re-run the cluster-side Section 08
steps from scratch.

---

## Pinned versions

The runbook was validated end-to-end with these exact package
versions. The `dnf install` commands include the
Name-Version-Release pins inline.

If any pin fails with `No match for argument` on a future run —
OpenHPC's updates repo prunes older patch releases every few
months — drop the version suffix for that one package, re-run,
and continue. Minor-version drift is almost always harmless.

| Component | Pinned version |
|-----------|----------------|
| Rocky Linux (both head and compute) | **9.7** |
| OpenHPC release metapackage | `ohpc-release-3-1.el9` |
| Warewulf | `warewulf-ohpc-4.6.4-340.ohpc.1.3` |
| Warewulf dracut module | `warewulf-ohpc-dracut-4.6.4-340.ohpc.1.3` |
| Slurm (controller + slurmd) | `slurm-*-ohpc-24.11.5-331.ohpc.1.1` |
| OpenHPC Slurm-server meta | `ohpc-slurm-server-3.4-340.ohpc.4.1` |
| Lmod | `lmod-ohpc-8.7.64-340.ohpc.2.1` |
| GNU compiler toolchain | `gnu12-compilers-ohpc-12.2.0-300.ohpc.3.6` |
| OpenMPI | `openmpi4-gnu12-ohpc-4.1.5-300.ohpc.3.2` |
| OSU Micro-Benchmarks | `omb-gnu12-openmpi4-ohpc-6.1-300.ohpc.2.6` |
| OpenBLAS (base + serial + devel) | `0.3.29-1.el9` |
| Prometheus | `prometheus-3.11.2-1.el9` |
| Prometheus node-exporter | `node-exporter-1.11.1-1.el9` |
| Grafana | `grafana-13.0.1-1` |
| VirtualBox host | **7.2.x** |
| Vagrant | **2.4.9+** |

HPL 2.3 is built from netlib source in Section 07; the `ipxe.pxe`
binary Section 08 uses is fetched from `boot.ipxe.org`. Both are
external — see the **External artifacts** table below.

### External artifacts fetched during the build

| Artifact | URL | Used in |
|----------|-----|---------|
| OpenHPC 3 release RPM | `http://repos.openhpc.community/OpenHPC/3/EL_9/x86_64/ohpc-release-3-1.el9.x86_64.rpm` | §05, §08 |
| Full iPXE binary | `https://boot.ipxe.org/ipxe.pxe` | §08 |
| HPL 2.3 source tarball | `https://www.netlib.org/benchmark/hpl/hpl-2.3.tar.gz` | §07 |
| Grafana OSS repo | `https://rpm.grafana.com` | §10 |
| Docker Hub Rocky Linux base image | `docker://rockylinux:9` | §08 |
| Rocky primary mirror (Vagrant box via local metadata override) | `https://dl.rockylinux.org/pub/rocky/9/images/x86_64/Rocky-9-Vagrant-Vbox-9.7-20251123.2.x86_64.box` | §02 |

For fully offline demos, mirror all six into an artifact cache
before the bootcamp.

---

## Table of contents

- [Section 00 — Host preparation](#section-00-host-preparation)
- [Section 01 — What we're building](#section-01-what-were-building)
- [Section 02 — Bring up the head node](#section-02-bring-up-the-head-node)
- [Section 03 — Head as a shared resource](#section-03-head-as-a-shared-resource)
- [Section 04 — Cluster identity](#section-04-cluster-identity)
- [Section 05 — The provisioning engine](#section-05-the-provisioning-engine)
- [Section 06 — The job scheduler](#section-06-the-job-scheduler)
- [Section 07 — The science payload](#section-07-the-science-payload)
- [Section 08 — Compute nodes come alive](#section-08-compute-nodes-come-alive)
- [Section 09 — Run a real job](#section-09-run-a-real-job)
- [Section 10 — See it running](#section-10-see-it-running)
- [Section 11 — What we skipped and what's next](#section-11-what-we-skipped-and-whats-next)
- [Post-bootcamp self-study](#post-bootcamp-self-study)
- [Cleanup / start over](#cleanup-start-over)

---

## Section 00 — Host preparation

### What this section does

Windows 11's default security posture (Hyper-V, Memory Integrity,
Virtual Machine Platform) fights VirtualBox. This section is
pre-work — the host is prepared before the bootcamp starts, so
the demo opens on a machine that can actually run VMs. Learners
who arrive unprepared watch the host get fixed instead of watching
the cluster come alive.

### Prerequisites

None. Entry point.

### Steps

1. Follow [HOST-PREP.md](HOST-PREP.md) end to end. Expect 30–60
   minutes, most of which is reboots.

### What just happened

Windows 11 ships with several features that either take ownership
of the CPU's hardware virtualization extensions (Hyper-V, Virtual
Machine Platform) or block the memory allocations VirtualBox needs
(Memory Integrity under Core Isolation). Disabling them is not a
security downgrade for a dedicated lab laptop — it's restoring the
host to a state where a Type-2 hypervisor can function. Fast
Startup matters because it caches kernel state across "shutdowns"
and can silently re-enable disabled features; without turning it
off, the other changes may not stick.

### Validation

Run in a fresh PowerShell on the Windows host:

```powershell
systeminfo | Select-String "VM Monitor Mode"
VBoxManage --version
vagrant --version
```

Expected:

```
                           VM Monitor Mode Extensions: Yes
7.2.<patch>r<build>
Vagrant 2.4.9
```

Any VirtualBox 7.2.x; Vagrant 2.4.9 or later. If `systeminfo`
returns "A hypervisor has been detected" instead of the VM Monitor
Mode line, return to [HOST-PREP.md](HOST-PREP.md) Steps 1–4 with
Fast Startup confirmed off.

---

## Section 01 — What we're building

By the end of this bootcamp, a real MPI benchmark — High Performance
Linpack (HPL) — runs across two diskless compute nodes that PXE-booted
from a head node we built from nothing. Grafana shows the CPU saturation
on both compute nodes live as HPL runs. The whole stack fits on a
single 16 GB laptop.

**Prerequisites:** Section 00 complete ([HOST-PREP.md](HOST-PREP.md) validation passed).

### Architecture

```
                    ┌──────────────────────────────────────────────────┐
                    │                      head                         │
                    │          Rocky 9.7 · 4 GB · 2 vCPU                │
                    │                                                   │
                    │  NFS server · Warewulf (wwctl, wwd)               │
                    │  slurmctld · slurmdbd · MariaDB                   │
                    │  Munge · Prometheus · Grafana                     │
                    │                                                   │
                    │  10.0.0.1 (mgmt)     10.1.0.1 (data)              │
                    └────────┬───────────────────────────┬──────────────┘
                             │                           │
      ═══════════════════════╪═══════════════════════════╪═══════════════════
      hpc-mgmt  ·  10.0.0.0/24  ·  SSH · PXE · DHCP · Slurm ctrl · Munge
      ═══════════════════════╪═══════════════════════════╪═══════════════════
                             │                           │
      ───────────────────────┼───────────────────────────┼───────────────────
      hpc-data  ·  10.1.0.0/24  ·  NFS (/home /apps /scratch) · MPI
      ───────────────────────┼───────────────────────────┼───────────────────
                             │                           │
                        10.0.0.11                   10.0.0.12
                        10.1.0.11                   10.1.0.12
                             │                           │
                 ┌───────────┴───────────┐   ┌───────────┴───────────┐
                 │       compute01        │   │       compute02        │
                 │   diskless · PXE       │   │   diskless · PXE       │
                 │                        │   │                        │
                 │  NFS client · slurmd   │   │  NFS client · slurmd   │
                 │  node_exporter · Munge │   │  node_exporter · Munge │
                 └────────────────────────┘   └────────────────────────┘
```

Two fabrics, three nodes. `head` owns every service that isn't compute:
the scheduler, the provisioning stack, shared storage, observability.
`compute01` and `compute02` are diskless — they pull their root
filesystem from Warewulf on `head` over the management fabric at boot,
run whatever Slurm assigns them, and carry no state of their own.

### Terminology

**Head node / compute node.** The head is the single entry point and
control plane — it runs the scheduler, the provisioning stack, shared
storage, and observability. Compute nodes are the hands that execute
the work the scheduler hands them. Users log into the head, never into
compute nodes directly.

**Scheduler / job.** "Scheduler" in HPC means Slurm (or one of its
peers — PBS, LSF, Grid Engine). It accepts job submissions, queues
them, matches them to available resources, and records what ran. A job
is one submission — a shell script plus a set of resource requests
(N nodes, M cores, T hours).

**Fabric.** The network. HPC calls it a fabric because the network is
a high-performance substrate for computation, not a pipe to the
internet. This lab has two: `hpc-mgmt` for control traffic and
`hpc-data` for NFS and MPI between compute nodes. Production clusters
add a third — InfiniBand — when MPI latency matters.

**PXE boot / diskless / image / overlay.** PXE lets a machine with no
disk boot from a DHCP+TFTP server. "Diskless" in HPC means the compute
node has no local storage — its root filesystem lives in RAM, populated
from an "image" (Warewulf's delivered root, historically a chroot, now
an OCI container). An "overlay" is Warewulf's per-node customization
layer on top of the base image: hostname, SSH keys, Slurm config.

**MPI / rank / task.** MPI (Message Passing Interface) is the standard
for inter-process communication in parallel scientific computing. An
MPI job runs as a set of processes called "ranks" (rank 0 through
rank N-1) spread across compute nodes. A Slurm "task" is one process.
A "job" is the entire submission — all its ranks and all its resources.

**Module / module system.** HPC clusters serve many users with
conflicting software needs. A module system (Lmod is ours) lets a user
run `module load openmpi/4.1.5` and get that exact toolchain in their
shell without disturbing anyone else. It's disciplined `PATH` and
`LD_LIBRARY_PATH` manipulation with a friendly front-end.

### Enterprise IT → HPC mapping

| Enterprise IT | HPC equivalent | Key difference |
|---|---|---|
| Gold image / golden master | Warewulf container image | Same idea — one image boots many machines. HPC's image is stateless and loaded over PXE every boot, not flashed to disk. |
| Corporate NTP server | chrony pointed at pool.ntp.org | Same protocol, same daemon. HPC fleets need tighter sync because scheduler and MPI depend on clocks agreeing. |
| Active Directory / LDAP | Active Directory / AD+SSSD (production HPC at most commercial sites) or FreeIPA (academic labs) | Same directory tooling, typically bolted onto the existing enterprise identity. This lab uses local users because 4 hours. |
| Kerberos keytab | Munge key | Both are pre-shared secrets that let services trust each other. Munge is HPC's lighter-weight design — no KDC, no tickets, credentials expire in seconds. |
| Corporate PXE for workstations | Warewulf for compute nodes | PXE infrastructure you already know. Warewulf bundles DHCP, TFTP, HTTP, and an OCI-based image workflow into one CLI. |
| SCCM / Ansible Tower | Warewulf `wwctl` + overlays | Declarative node state, centrally managed. Warewulf's scope is narrower (boot image + per-node overlays) and there's no agent on compute nodes. |
| Corporate file share (CIFS/SMB) | NFS exports | Network-attached storage, different protocol. NFS is the HPC default — kernel-native on Linux and MPI workloads hate Windows-era locking semantics. |
| NAS appliance | Head serving NFS (this lab) / Lustre, GPFS, BeeGFS (production) | Same role, very different scale. Production HPC storage is a parallel filesystem across many servers, not a single NAS. |
| VLANs / broadcast domains | Internal fabrics (`hpc-mgmt`, `hpc-data`) | Same concept — layer-2 isolation. HPC calls them fabrics and often has more of them, including one reserved for low-latency MPI. |
| 10/25 GbE corporate networking | InfiniBand or RoCE for MPI | Ethernet works for MPI; InfiniBand is 5-50× lower latency. Production HPC invests heavily here; this lab uses plain ethernet. |
| Tiered storage (SAN / NAS / archive) | `/home` + `/apps` + `/scratch` | Same tiering idea, different drivers. HPC tiers by expected lifetime: `/home` backed up, `/apps` holds shared software, `/scratch` fast and disposable. |
| vCenter / SCVMM | No direct equivalent | Closest pair is Warewulf (compute node lifecycle) + Slurm (workload placement). HPC doesn't think of nodes as movable VMs. |
| VMware HA / DRS | No equivalent | HPC nodes are cattle. A failed node drops its jobs; Slurm re-queues them. No live migration. |
| Chargeback / showback | Slurm accounting (`slurmdbd` + MariaDB) | Identical motivation — track who used what for how long. Ships with the scheduler, not bolted on. |
| Service account | Munge-authenticated service-to-service trust | Both establish that a request came from a trusted process. Munge credentials are signed and expire in seconds, not months. |
| systemd service | systemd service | Same. `slurmctld.service`, `warewulfd.service`, `munge.service` — same unit files you already read. |
| Package manager (dnf / apt) | dnf + OpenHPC repo | Same tooling, one extra signed repository. OpenHPC packages install to `/opt/ohpc/`, not `/usr/`, so they coexist with system packages. |
| Patch management / OS updates | Image rebuild + reboot (compute) · dnf update (head) | For compute nodes: rebuild the Warewulf image and reboot. The head still patches conventionally. |
| Prometheus + Grafana (enterprise) | Prometheus + Grafana (HPC) | Identical. Same exporters, same dashboards, same alerting patterns. HPC adds job-aware metrics; the stack is the same. |
| RHEL / CentOS / Rocky (enterprise) | RHEL / CentOS / Rocky (HPC) | Same distros. HPC clusters typically sit one minor version behind because the stack (drivers, MPI, scheduler) needs to catch up. |
| Ansible / Puppet / Chef | Ansible / Puppet / Chef + Warewulf | Same config-management tools manage the head. Compute nodes themselves are configured by Warewulf's overlays, not by an agent. |
| Backup / DR | Mostly absent by design | `/home` and `/apps` are usually backed up; `/scratch` explicitly isn't — it's disposable. Whole-cluster DR in HPC is rare. |
| "The server" (single-purpose role) | "The node" — interchangeable by design | A compute node is provisioned, used, decommissioned, and re-provisioned with zero ceremony. Hostnames are identifiers, not identities. |
| RDP / interactive desktop | `ssh` + `sbatch` | HPC is batch-first; interactive use exists but isn't primary. |

### The promise

By the end of Section 09, HPL — a real parallel benchmark — runs across
both compute nodes and produces a GFLOPS number you can compare to
other clusters. Section 10 shows that same job live on a Grafana
dashboard. If both of those land, the bootcamp has delivered.

This is a scaffolding exercise, not a certification. You will not leave
here capable of running a production HPC cluster. You will leave with
a mental map — the concepts, the terminology, the patterns — so that
when HPC shows up in your career, you recognize what you're looking at
and know where to go deeper. Section 11 is that map, role by role.

With the map in hand, we start by bringing up the head node.

---


## Section 02 — Bring up the head node

### What this section does

Turn the Vagrantfile into a running Rocky 9.7 VM named `head` with
two vCPUs, 4 GB of RAM, 40 GB of root disk, and three network
interfaces (NAT + `hpc-mgmt` + `hpc-data`). By the end, `vagrant
ssh` drops into a shell on a Linux box you own, the root filesystem
has been grown to fill the virtual disk, and the VM is sane.

### Prerequisites

Section 00 ([HOST-PREP.md](HOST-PREP.md)) complete. You're at the
repository root (directory containing `Vagrantfile`).

### Steps

1. **Bring the VM up.**

```bash
vagrant up
```

Expected tail:

```
==> default: Machine booted and ready!
==> default: Checking for guest additions in VM...
    default: No guest additions were detected on the base box...
==> default: Setting hostname...
==> default: Configuring and enabling network interfaces...
==> default: Rsyncing folder: ...
```

Two warning classes are expected and safe to ignore:

- `You assigned a static IP ending in ".1" or ":1"` (4 copies) —
  benign on VirtualBox internal networks; no router is present.
- `No guest additions were detected` — Rocky's base box doesn't
  ship them; Vagrant falls back to `rsync` for `/vagrant` sync.

2. **Confirm the VM is running.**

```bash
vagrant status
```

Expected: `default   running (virtualbox)`.

3. **SSH into head and grow the root partition.**

```bash
vagrant ssh
```

The Vagrantfile declares a 40 GB primary disk, but that directive
only resizes the *virtual disk*. Inside the guest, Rocky's base
box leaves `/dev/sda4` at the ~9 GB base-box size. Grow it:

```bash
sudo dnf install -y cloud-utils-growpart
sudo growpart /dev/sda 4
sudo xfs_growfs /
df -h /
```

Expected (final line):

```
/dev/sda4        39G  3.1G   36G   8% /
```

Without this step, the filesystem fills up halfway through
Section 08.

### What just happened

Vagrant resolved the `rockylinux/9` box to version 9.7.0 via the
local metadata override (`rockylinux-9.7-primary.json`), pulled
the `.box` artifact from Rocky's primary mirror, created a
VirtualBox VM named `head` at 40 GB virtual disk, applied the
three firmware customizations (`--chipset ich9`, `--ioapic on`,
`--hpet on`) that EL9 needs to boot multi-vCPU, attached three
network adapters (NAT + two internal networks), and booted it.
`growpart` and `xfs_growfs` extended the root filesystem to fill
the disk.

### Validation

Inside the VM:

```bash
hostname
grep -E '^(NAME|VERSION)=' /etc/os-release
nproc
ip -br a
df -h /
```

Expected:

```
head
NAME="Rocky Linux"
VERSION="9.7 (Blue Onyx)"
2
lo               UNKNOWN        127.0.0.1/8 ::1/128
eth0             UP             10.0.2.15/24 ...
eth1             UP             10.0.0.1/24 ...
eth2             UP             10.1.0.1/24 ...
/dev/sda4        39G   3.1G  36G   8% /
```

Six things must land: hostname `head`, Rocky 9.7, 2 CPUs, three
UP interfaces with the right IPs, `/` at 39 G capacity. A `nproc`
of 1 means the Vagrantfile firmware customizations didn't apply;
a disk smaller than 39 G means `growpart` was skipped.

---

## Section 03 — Head as a shared resource

### What this section does

Turn the generic Rocky VM into something that will anchor a
cluster: cluster-wide hostname resolution, lab-only security
posture, persistent logs, the standard four-repo package foundation,
baseline admin tooling, time synchronization, and three NFS exports
wired to the data fabric. Most of this is enterprise-IT muscle
memory; two choices are HPC-specific.

### Prerequisites

Section 02 complete. You're at `[vagrant@head ~]$`.

### Steps

1. **Add cluster hosts to `/etc/hosts`.**

```bash
sudo tee -a /etc/hosts >/dev/null <<'HOSTS'

# --- cluster ---
# hpc-mgmt fabric (10.0.0.0/24)
10.0.0.1    head
10.0.0.11   compute01
10.0.0.12   compute02

# hpc-data fabric (10.1.0.0/24)
10.1.0.1    head-data
10.1.0.11   compute01-data
10.1.0.12   compute02-data
HOSTS
```

Two hostnames per node — the short name on the mgmt fabric, a
`-data` alias on the data fabric. NFS and MPI use the `-data`
names; SSH, Slurm control, and Warewulf use the short names.

Verify:

```bash
getent hosts compute01 compute01-data head-data
```

Expected:

```
10.0.0.11       compute01
10.1.0.11       compute01-data
10.1.0.1        head-data
```

Looking up `head` from head itself returns the machine's own
interface addresses — NSS's `myhostname` module pre-empts. This
is harmless: compute nodes looking up `head` via their own
`/etc/hosts` correctly get `10.0.0.1`.

2. **Lab-only posture: SELinux permissive, firewalld off, persistent journal.**

```bash
sudo sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config
sudo setenforce 0
sudo systemctl disable --now firewalld
sudo mkdir -p /var/log/journal
sudo systemd-tmpfiles --create --prefix /var/log/journal
sudo systemctl restart systemd-journald
```

Verify:

```bash
getenforce                          # Permissive
systemctl is-active firewalld       # inactive
sudo journalctl --disk-usage        # some MB used in the file system
```

A production cluster keeps SELinux enforcing and runs firewalld
with a minimal ruleset; we skip both because the cluster is
single-user on internal fabrics and time is short. Persistent
journald is general Linux hygiene — by default journald logs to
RAM (`/run/log/journal`) and discards on reboot.

3. **Enable CRB + EPEL and install baseline packages.**

```bash
sudo dnf config-manager --set-enabled crb
sudo dnf install -y epel-release
sudo dnf install -y vim tmux bind-utils net-tools
```

Rocky 9 ships BaseOS + AppStream enabled. CRB (CodeReady Builder)
carries dev packages OpenHPC dependencies need; EPEL carries
community tooling. The four-repo foundation (BaseOS / AppStream /
CRB / EPEL) is standard for any RHEL-family HPC head.

Verify:

```bash
sudo dnf repolist --enabled | awk 'NR==1 || /^(baseos|appstream|crb|epel )/'
```

Expected:

```
repo id    repo name
appstream  Rocky Linux 9 - AppStream
baseos     Rocky Linux 9 - BaseOS
crb        Rocky Linux 9 - CRB
epel       Extra Packages for Enterprise Linux 9 - x86_64
```

4. **Verify time sync.**

Chrony is pre-installed and active on the Rocky base image.

```bash
chronyc sources
chronyc tracking | head -2
```

Expected: one source marked `^*`, `Stratum 2-5`.

5. **Create NFS exports.**

```bash
sudo mkdir -p /export/home /export/apps /export/scratch
sudo chmod 1777 /export/scratch

sudo tee /etc/exports >/dev/null <<'EXPORTS'
/export/home     10.1.0.0/24(rw,sync,no_root_squash,no_subtree_check)
/export/apps     10.1.0.0/24(rw,sync,no_root_squash,no_subtree_check)
/export/scratch  10.1.0.0/24(rw,sync,no_root_squash,no_subtree_check)
EXPORTS

sudo systemctl enable --now nfs-server
sudo exportfs -ra
sudo exportfs -v
```

`/export/scratch` gets sticky-bit `1777` so every user can write
their own files without disturbing anyone else's (same as `/tmp`).
The export ACL is restricted to `10.1.0.0/24`, the `hpc-data`
fabric. This is the second HPC-specific choice in this section:
a compute node reaching NFS from its `hpc-mgmt` address
(`10.0.0.11`) will be refused, so bulk traffic *can only* flow
on the data fabric.

### What just happened

The head is now in the posture an HPC controller expects. Every
cluster node resolves on both fabrics by hostname. Lab-only
security is out of the way. Logs persist. The four-repo package
foundation and baseline tooling are in place. Time is synchronized
against the public NTP pool. Three NFS exports are bound to the
data fabric, waiting for compute-node clients.

None of this is HPC-specific except the two-fabric hostname
pattern and the fabric-restricted NFS ACL. Those two encode the
architecture of the cluster — control on one fabric, bulk data on
the other — at the OS layer, before any HPC software is installed.

### Validation

```bash
getenforce
systemctl is-active firewalld nfs-server chronyd
sudo exportfs -v
getent hosts head-data compute01
```

Expected:

```
Permissive
inactive
active
active
/export/home  	10.1.0.0/24(...)
/export/apps  	10.1.0.0/24(...)
/export/scratch
		10.1.0.0/24(...)
10.1.0.1        head-data
10.0.0.11       compute01
```

---

## Section 04 — Cluster identity

### What this section does

Install Munge (HPC's service-to-service auth broker), generate the
cluster's shared key, create two HPC users with NFS-backed homes
bind-mounted onto `/home`, give `hpcadmin` passwordless sudo, and
generate SSH keypairs for root + both users so passwordless SSH
works across the cluster the moment compute nodes come online.
This is the first section where the pattern stops being generic
Linux and starts being HPC-specific; Munge is the teaching beat.

### Prerequisites

Section 03 complete. NFS server is running, `/export/home` exported
to `10.1.0.0/24`.

### Steps

1. **Install Munge and generate the shared key.**

```bash
sudo dnf install -y munge munge-libs
sudo /usr/sbin/create-munge-key -f
sudo systemctl enable --now munge
munge -n | unmunge | head -6
```

Expected (tail):

```
STATUS:           Success (0)
ENCODE_HOST:      head (<ip>)
ENCODE_TIME:      <timestamp>
DECODE_TIME:      <timestamp>
TTL:              300
CIPHER:           aes128 (4)
```

**Why Munge.** Every HPC service — Slurm's daemons, Warewulf,
MPI process launchers — needs to prove to peers that a request
really came from a cluster member. In enterprise IT you'd reach
for Kerberos: KDC, keytabs, TGTs, renewals. HPC's trust model is
narrower: every node holds the same symmetric key in
`/etc/munge/munge.key` (mode 400, owned `munge:munge`). Any
process on any cluster node can ask its local `munged` to sign
or verify a credential — no network protocol between nodes, no
TGT expiry. Simpler than Kerberos and sufficient because cluster
nodes are equally-trusted peers. The key file will travel to
every compute node in Section 08.

2. **Create the two HPC users.**

```bash
sudo useradd -u 1100 -m -d /home/hpcadmin -s /bin/bash -c 'HPC admin' hpcadmin
sudo useradd -u 1101 -m -d /home/hpcuser  -s /bin/bash -c 'HPC user'  hpcuser
```

UIDs are pinned so they'll match the same users on compute nodes
in Section 08. Consistent UIDs across the cluster are the
prerequisite for NFS to feel sane — without them, a file
`hpcadmin` writes on head appears as an unmapped UID on a
compute node with no permission to edit.

3. **Move existing homes into `/export/home` and bind-mount `/home`.**

```bash
sudo rsync -a /home/ /export/home/
sudo tee -a /etc/fstab >/dev/null <<'FSTAB'

# Cluster home: /home is a bind of /export/home so users' files live on
# the NFS-exported path and appear identically on head and compute.
/export/home  /home  none  bind  0 0
FSTAB
sudo mount /home
findmnt /home
stat -c 'dev=%d inode=%i' /home /export/home
```

Expected:

```
TARGET SOURCE                  FSTYPE OPTIONS
/home  /dev/sda4[/export/home] xfs    rw,...
dev=<N> inode=<M>
dev=<N> inode=<M>
```

The two `stat` lines must show the same device and inode — that's
the proof of a bind mount.

4. **Passwordless sudo for `hpcadmin`.**

```bash
echo 'hpcadmin ALL=(ALL) NOPASSWD:ALL' | sudo tee /etc/sudoers.d/hpcadmin
sudo chmod 440 /etc/sudoers.d/hpcadmin
sudo visudo -c -f /etc/sudoers.d/hpcadmin
```

Expected: `/etc/sudoers.d/hpcadmin: parsed OK`.

`visudo -c` is the sanity check; a malformed drop-in can lock
you out of sudo entirely.

5. **Generate SSH keypairs for root, `hpcadmin`, and `hpcuser`.**

```bash
sudo bash -c '
set -e
for u in root hpcadmin hpcuser; do
  if [ "$u" = root ]; then home=/root; else home=/home/$u; fi
  install -d -m 700 -o $u -g $u "$home/.ssh"
  sudo -u $u ssh-keygen -t ed25519 -N "" -f "$home/.ssh/id_ed25519" -C "$u@cluster" -q
  cat "$home/.ssh/id_ed25519.pub" >> "$home/.ssh/authorized_keys"
  chown $u:$u "$home/.ssh/authorized_keys"
  chmod 600 "$home/.ssh/authorized_keys"
  cat > "$home/.ssh/config" <<SSHCFG
Host compute* head*
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null
    LogLevel ERROR
SSHCFG
  chown $u:$u "$home/.ssh/config"
  chmod 600 "$home/.ssh/config"
done
'
```

Each user gets `id_ed25519` / `id_ed25519.pub`, their own pubkey
appended to `authorized_keys`, and a `.ssh/config` that disables
host-key checking for `compute*` and `head*` hosts. Compute nodes
get re-imaged frequently, so host keys change often; on an
internal fabric this is the pragmatic choice.

Because `/home` is NFS-shared across the cluster, the users'
`authorized_keys` and `.ssh/config` appear automatically on
every compute node. Root's keys live in `/root/.ssh/` — not on
NFS — so root's public key has to be pushed into the compute
image in Section 08.

### What just happened

- **Munge** is running on head and holds the cluster's shared
  secret. The key travels to every compute node in Section 08
  so Slurm and Warewulf daemons can authenticate peer-to-peer.
- **Two HPC users exist** with UIDs 1100 and 1101 that will
  match on compute nodes. Their homes live in `/export/home`,
  which NFS serves on `hpc-data`.
- **`/home` is a bind of `/export/home`.** Users, scripts, and
  SSH configs see one canonical path on every node.
- **SSH keypairs** are in place for root, `hpcadmin`, and
  `hpcuser`; each trusts its own key.

### Validation

```bash
systemctl is-active munge
id hpcadmin
id hpcuser
findmnt /home
sudo -u hpcadmin sudo -n whoami
sudo -u hpcadmin ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o LogLevel=ERROR localhost 'whoami; hostname; pwd'
```

Expected (condensed):

```
active
uid=1100(hpcadmin) gid=1100(hpcadmin) groups=1100(hpcadmin)
uid=1101(hpcuser) gid=1101(hpcuser) groups=1101(hpcuser)
/home  /dev/sda4[/export/home] ...
root
hpcadmin
head
/home/hpcadmin
```

---

## Section 05 — The provisioning engine

### What this section does

Install Warewulf 4 via OpenHPC, configure it for our two-fabric
network, and start `warewulfd`. This is the engine that will
deliver diskless OS images to compute nodes in Section 08. DHCP
is configured in `warewulf.conf` but intentionally not started
here — no compute nodes exist, and serving DHCP to nothing is
noisy. DHCP comes on in Section 08.

### Prerequisites

Section 04 complete.

### Steps

1. **Add the OpenHPC 3 repo.**

```bash
sudo dnf install -y http://repos.openhpc.community/OpenHPC/3/EL_9/x86_64/ohpc-release-3-1.el9.x86_64.rpm
sudo dnf makecache
```

The `ohpc-release` RPM drops `OpenHPC.repo` into
`/etc/yum.repos.d/` enabling both `[OpenHPC]` (base) and
`[OpenHPC-updates]`, and installs the signing GPG key.

Verify:

```bash
sudo dnf repolist --enabled | grep -i openhpc
```

Expected:

```
OpenHPC          OpenHPC-3 - Base
OpenHPC-updates  OpenHPC-3 - Updates
```

2. **Install Warewulf 4.**

```bash
sudo dnf install -y warewulf-ohpc-4.6.4-340.ohpc.1.3
wwctl version
```

The package name is `warewulf-ohpc`, **not** `ohpc-warewulf`.
OpenHPC 3 also ships Warewulf 3 under the `ohpc-warewulf`
meta-package — that's the legacy chroot-based version. We want
Warewulf 4 (OCI-based), which is the singular-named
`warewulf-ohpc`.

Expected:

```
wwctl version:	 4.6.4-1
rpc version: apiPrefix:"rc1" apiVersion:"1" warewulfVersion:"4.6.4-1"
```

3. **Configure `/etc/warewulf/warewulf.conf`.**

```bash
sudo cp /etc/warewulf/warewulf.conf /etc/warewulf/warewulf.conf.default
sudo tee /etc/warewulf/warewulf.conf >/dev/null <<'WWCONF'
ipaddr: 10.0.0.1
netmask: 255.255.255.0
network: 10.0.0.0
warewulf:
    port: 9873
    secure: false
    update interval: 60
    autobuild overlays: true
    host overlay: true
    grubboot: false
    systemd name: warewulfd
api:
    enabled: false
    allowed subnets:
        - 127.0.0.0/8
        - ::1/128
dhcp:
    enabled: false
    template: default
    range start: 10.0.0.100
    range end: 10.0.0.200
    systemd name: dhcpd
tftp:
    enabled: true
    tftproot: /srv/tftpboot
    systemd name: tftp
    ipxe:
        00:0B: arm64-efi/snponly.efi
        "00:00": undionly.kpxe
        "00:07": ipxe-snponly-x86_64.efi
        "00:09": ipxe-snponly-x86_64.efi
nfs:
    enabled: false
    systemd name: nfs-server
ssh:
    key types:
        - ed25519
        - ecdsa
        - rsa
        - dsa
image mounts:
    - source: /etc/resolv.conf
      dest: /etc/resolv.conf
      readonly: true
WWCONF
```

Key differences from the upstream default:

- `netmask: 255.255.255.0` — /24 matching the `hpc-mgmt` network.
- `dhcp.enabled: false` — config present, daemon off until Section 08.
- `nfs.enabled: false` — we set up NFS ourselves in Section 03.

4. **Generate PXE artifacts and start warewulfd.**

```bash
sudo wwctl configure tftp
sudo systemctl enable --now warewulfd
```

`wwctl configure tftp` drops iPXE binaries (`undionly.kpxe`,
`ipxe-snponly-x86_64.efi`, and aarch64 equivalent) into
`/srv/tftpboot/warewulf/` and enables `tftp.socket`.

Verify:

```bash
systemctl is-active warewulfd
sudo ss -tulnp | awk '/:(9873|69) / || NR==1'
sudo wwctl profile list
sudo wwctl node list
systemctl is-enabled dhcpd
systemctl is-active dhcpd
```

Expected:

```
active
Netid State  ... Local Address:Port ...
udp   UNCONN 0 0    *:69   *:* users:(("in.tftpd",...))
tcp   LISTEN 0 4096 *:9873 *:* users:(("wwctl",...))
PROFILE NAME  COMMENT/DESCRIPTION
------------  -------------------
default       This profile is automatically included for each node
NODE NAME  PROFILES  NETWORK
---------  --------  -------
disabled
inactive
```

### What just happened

The head is now an HPC provisioning server. `warewulfd` listens
on `10.0.0.1:9873` for compute nodes to fetch runtime overlays.
TFTP on UDP 69 is ready to serve iPXE boot binaries. The `default`
node profile has been auto-created; any compute node defined
later inherits its settings.

DHCP is intentionally off. Turning it on now would mean head
answers DHCP on an internal network with no clients — harmless,
but muddies the story. Section 08 flips `dhcp.enabled: true`,
runs `wwctl configure --all`, and starts `dhcpd` at the same
moment the first compute node powers on.

### Validation

```bash
wwctl version
sudo wwctl profile list
systemctl is-active warewulfd
systemctl is-active dhcpd
```

Expected:

```
wwctl version:	 4.6.4-1
...
default  This profile is automatically included for each node
active
inactive
```

---

## Section 06 — The job scheduler

### What this section does

Stand up Slurm — the controller (`slurmctld`), the accounting
daemon (`slurmdbd`), and its MariaDB backing store. Register the
cluster (`hpc`) in the accounting DB. `slurm.conf` is written
with the MPI launcher, timeout, and Epilog settings Section 09
will need, so no reconfig later. By the end, `sinfo`, `squeue`,
and `sacctmgr` all work on head even though no compute nodes are
up; Slurm knows `compute[01-02]` are defined and waits for a
`slurmd` to check in.

### Prerequisites

Section 04 (Munge) and Section 05 (OpenHPC repo).

### Steps

1. **Install MariaDB + Slurm server.**

```bash
sudo dnf install -y mariadb-server ohpc-slurm-server-3.4-340.ohpc.4.1
```

`ohpc-slurm-server` pulls `slurm-ohpc`, `slurm-slurmctld-ohpc`,
`slurm-slurmdbd-ohpc`, `slurm-example-configs-ohpc`, plus
dependencies. The `slurm` system user (UID 202) is created
automatically.

2. **Enable MariaDB and create the accounting database.**

```bash
sudo systemctl enable --now mariadb
sudo mysql <<'SQL'
CREATE DATABASE IF NOT EXISTS slurm_acct_db;
CREATE USER IF NOT EXISTS 'slurm'@'localhost' IDENTIFIED BY 'slurmdbpass';
GRANT ALL PRIVILEGES ON slurm_acct_db.* TO 'slurm'@'localhost';
FLUSH PRIVILEGES;
SQL
```

Lab-only password. A production site would generate a strong
random password, store it in a secret manager, and restrict the
DB user's reach to `localhost` (done here) plus TLS.

3. **Create the runtime directories Slurm expects.**

```bash
sudo install -d -m 755 -o slurm -g slurm /var/log/slurm /var/spool/slurmctld /var/spool/slurmd
sudo tee /etc/tmpfiles.d/slurm.conf >/dev/null <<'TMPF'
d /run/slurm 0755 slurm slurm -
TMPF
sudo systemd-tmpfiles --create /etc/tmpfiles.d/slurm.conf
```

`/run/slurm/` is where slurmctld/slurmdbd drop their PID files.
`/run` is tmpfs, so the tmpfiles.d drop-in recreates it on every
reboot.

4. **Write `slurmdbd.conf`.**

```bash
sudo tee /etc/slurm/slurmdbd.conf >/dev/null <<'SLURMDBD'
AuthType=auth/munge
AuthInfo=/var/run/munge/munge.socket.2
DbdAddr=localhost
DbdHost=localhost
SlurmUser=slurm
DebugLevel=info
LogFile=/var/log/slurm/slurmdbd.log
PidFile=/run/slurm/slurmdbd.pid
StorageType=accounting_storage/mysql
StorageHost=localhost
StorageUser=slurm
StoragePass=slurmdbpass
StorageLoc=slurm_acct_db
SLURMDBD
sudo chown slurm:slurm /etc/slurm/slurmdbd.conf
sudo chmod 600 /etc/slurm/slurmdbd.conf
```

Mode `600` owned by `slurm` because the file contains the DB
password. slurmdbd refuses to start otherwise.

5. **Start slurmdbd.**

```bash
sudo systemctl enable --now slurmdbd
```

Verify:

```bash
systemctl is-active slurmdbd
sudo mysql slurm_acct_db -e 'SHOW TABLES;' | head -5
```

Expected:

```
active
Tables_in_slurm_acct_db
acct_coord_table
acct_table
cluster_table
```

On first start, slurmdbd creates ~12 tables. A log warning
"Database settings not recommended values" is harmless at this
scale.

6. **Write `slurm.conf`.**

```bash
sudo tee /etc/slurm/slurm.conf >/dev/null <<'SLURM'
# Cluster identity
ClusterName=hpc
SlurmctldHost=head(10.0.0.1)

# Process tracking and auth
ProctrackType=proctrack/cgroup
MpiDefault=pmix                     # OpenHPC's OpenMPI 4 uses PMIx, not PMI-1/2
ReturnToService=1
SlurmUser=slurm
SlurmctldPidFile=/run/slurm/slurmctld.pid
SlurmctldPort=6817
SlurmdPidFile=/run/slurm/slurmd.pid
SlurmdPort=6818
SlurmdSpoolDir=/var/spool/slurmd
StateSaveLocation=/var/spool/slurmctld
SwitchType=switch/none
TaskPlugin=task/affinity
PropagateResourceLimitsExcept=MEMLOCK

# Timers
InactiveLimit=0
KillWait=30
MinJobAge=300
SlurmctldTimeout=120
SlurmdTimeout=600                   # Generous; compute reboots take ~90s
Waittime=0

# Scheduling
SchedulerType=sched/backfill
SelectType=select/cons_tres
SelectTypeParameters=CR_Core_Memory

# Logging
SlurmctldDebug=info
SlurmctldLogFile=/var/log/slurm/slurmctld.log
SlurmdDebug=info
SlurmdLogFile=/var/log/slurm/slurmd.log

# Accounting via slurmdbd
JobAcctGatherType=jobacct_gather/linux
JobAcctGatherFrequency=30
JobCompType=jobcomp/filetxt
AccountingStorageType=accounting_storage/slurmdbd
AccountingStorageHost=localhost

# NOTE: Epilog= deliberately omitted. OpenHPC's example epilog
# runs `pkill -KILL -U $UID` which returns 1 when no processes
# match — Slurm reads non-zero as "epilog failed" and drains
# the node after every trivial job.
SlurmctldParameters=enable_configless
LaunchParameters=use_interactive_step

# Nodes and partition
NodeName=compute[01-02] CPUs=2 Sockets=1 CoresPerSocket=2 ThreadsPerCore=1 RealMemory=1900 State=UNKNOWN
PartitionName=normal Nodes=compute[01-02] Default=YES MaxTime=INFINITE State=UP
SLURM
sudo chown slurm:slurm /etc/slurm/slurm.conf
```

Three lines worth pointing out:

- `MpiDefault=pmix` — OpenHPC's OpenMPI ships with PMIx support
  but not Slurm's classic PMI-1/2. Setting this now means
  Section 09 can launch MPI jobs with `srun` directly.
- `SlurmdTimeout=600` — generous timeout so that compute
  reboots don't auto-down the nodes.
- `SlurmctldParameters=enable_configless` — compute nodes
  fetch `slurm.conf` from the controller at boot instead of
  carrying a synced copy in every overlay.

7. **Start slurmctld.**

```bash
sudo systemctl enable --now slurmctld
```

Verify:

```bash
systemctl is-active slurmctld
sudo tail -5 /var/log/slurm/slurmctld.log
```

Expected tail includes `Running as primary controller`.

8. **Register the cluster with sacctmgr (idempotent).**

```bash
sudo sacctmgr -i add cluster Name=hpc
```

Modern Slurm (22.05+) auto-registers a cluster the first time
slurmctld contacts slurmdbd, so you'll usually see:

```
 This cluster hpc already exists.  Not adding.
```

That IS the success path.

### What just happened

Slurm's control plane is alive. `slurmctld` holds the cluster
state. `slurmdbd` records every job, step, and exit through
MariaDB. The two daemons authenticate to each other over Munge.
`MpiDefault=pmix` and `SlurmdTimeout=600` are set so Section 09
runs clean without any reconfig.

The accounting DB is the HPC equivalent of the chargeback
system enterprise IT already knows. Every job's start/end,
CPU-seconds, memory high-water, partition, user, account — all
logged.

### Validation

```bash
sudo sacctmgr list cluster
sinfo
squeue
sudo scontrol show config | grep -E 'ClusterName|MpiDefault|SlurmctldHost|SlurmdTimeout'
```

Expected:

```
   Cluster     ControlHost  ControlPort   RPC     Share ...
---------- --------------- ------------ ----- --------- ...
       hpc       127.0.0.1         6817 10752         1 ...

PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
normal*      up   infinite      2   unk* compute[01-02]

             JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)

ClusterName             = hpc
MpiDefault              = pmix
SlurmctldHost[0]        = head(10.0.0.1)
SlurmdTimeout           = 600 sec
```

Both compute nodes in state `unk*` is the correct Section 06
endpoint. They transition to `idle` in Section 08 when `slurmd`
on each compute contacts the controller.

---

## Section 07 — The science payload

### What this section does

Install the researcher-facing layer: GCC 12 toolchain, OpenMPI
4.1.5, Lmod, OSU Micro-Benchmarks, and HPL (High Performance
Linpack, built from source because OpenHPC 3.3 no longer ships
it as a package). Everything lands under `/opt/ohpc/`, which is
bind-mounted onto `/export/apps` so the whole software tree is
already on the NFS-exported path compute nodes will mount in
Section 08.

### Prerequisites

Section 06 complete.

### Steps

1. **Install the OpenHPC software stack + build tools.**

```bash
sudo dnf install -y \
  lmod-ohpc-8.7.64-340.ohpc.2.1 \
  gnu12-compilers-ohpc-12.2.0-300.ohpc.3.6 \
  openmpi4-gnu12-ohpc-4.1.5-300.ohpc.3.2 \
  omb-gnu12-openmpi4-ohpc-6.1-300.ohpc.2.6 \
  ohpc-autotools-3.4-340.ohpc.4.1 \
  make \
  openblas-devel-0.3.29-1.el9
```

- `make` — Rocky minimal doesn't ship it and `ohpc-autotools`
  only provides autoconf/automake/libtool. HPL build below
  needs it.
- `openblas-devel` pulls in `openblas-serial` (which ships the
  actual `/lib64/libopenblas.so.0` that `xhpl` links against).

Everything else installs under `/opt/ohpc/`.

2. **Bind-mount `/opt/ohpc` onto `/export/apps`.**

```bash
sudo rsync -a /opt/ohpc/ /export/apps/
sudo find /opt/ohpc -mindepth 1 -maxdepth 1 -exec rm -rf {} +
sudo tee -a /etc/fstab >/dev/null <<'FSTAB'

# OpenHPC software tree: /opt/ohpc is a bind of /export/apps so package
# installs land on the NFS-exported path for compute nodes.
/export/apps  /opt/ohpc  none  bind  0 0
FSTAB
sudo mount /opt/ohpc
```

Same pattern as `/home` in Section 04. Future writes to
`/opt/ohpc` (package installs, the HPL build below) land on
`/export/apps`. Compute nodes in Section 08 NFS-mount
`head-data:/export/apps` onto `/opt/ohpc` and see the same tree.

Verify:

```bash
findmnt /opt/ohpc
```

Expected:

```
TARGET    SOURCE                  FSTYPE OPTIONS
/opt/ohpc /dev/sda4[/export/apps] xfs    rw,...
```

3. **Build HPL 2.3 from source.**

HPL is not packaged in OpenHPC 3.3 (dropped upstream — production
sites build against tuned BLAS, so a generic packaged HPL isn't
representative). Source from netlib.

```bash
cd /tmp
curl -sLO https://www.netlib.org/benchmark/hpl/hpl-2.3.tar.gz
tar xzf hpl-2.3.tar.gz
cd hpl-2.3

tee Make.Linux_gnu12_openmpi4 >/dev/null <<'MAKE'
SHELL        = /bin/sh
CD           = cd
CP           = cp
LN_S         = ln -s
MKDIR        = mkdir
RM           = /bin/rm -f
TOUCH        = touch
ARCH         = Linux_gnu12_openmpi4
TOPdir       = /tmp/hpl-2.3
INCdir       = $(TOPdir)/include
BINdir       = $(TOPdir)/bin/$(ARCH)
LIBdir       = $(TOPdir)/lib/$(ARCH)
HPLlib       = $(LIBdir)/libhpl.a
MPdir        =
MPinc        =
MPlib        =
LAdir        = /usr/lib64
LAinc        =
LAlib        = -lopenblas
F2CDEFS      = -DAdd_ -DF77_INTEGER=int -DStringSunStyle
HPL_INCLUDES = -I$(INCdir) -I$(INCdir)/$(ARCH) $(LAinc) $(MPinc)
HPL_LIBS     = $(HPLlib) $(LAlib) $(MPlib)
HPL_OPTS     = -DHPL_PROGRESS_REPORT
HPL_DEFS     = $(F2CDEFS) $(HPL_OPTS) $(HPL_INCLUDES)
CC           = mpicc
CCNOOPT      = $(HPL_DEFS)
CCFLAGS      = $(HPL_DEFS) -O3 -funroll-loops
LINKER       = mpicc
LINKFLAGS    = $(CCFLAGS)
ARCHIVER     = ar
ARFLAGS      = r
RANLIB       = echo
MAKE

source /etc/profile.d/lmod.sh
module load gnu12 openmpi4
make arch=Linux_gnu12_openmpi4
```

`module load gnu12 openmpi4` puts `mpicc` on PATH and pulls in
the MPI headers and libraries. `-lopenblas` resolves against
EPEL's `/lib64/libopenblas.so.0`. Build takes ~2 minutes on the
head VM.

Output at `/tmp/hpl-2.3/bin/Linux_gnu12_openmpi4/xhpl`.

4. **Stage HPL under `/opt/ohpc/pub/apps/` and install the modulefile.**

```bash
sudo install -d -m 755 /opt/ohpc/pub/apps/hpl/2.3/bin
sudo install -m 755 /tmp/hpl-2.3/bin/Linux_gnu12_openmpi4/xhpl /opt/ohpc/pub/apps/hpl/2.3/bin/xhpl
sudo install -m 644 /tmp/hpl-2.3/bin/Linux_gnu12_openmpi4/HPL.dat /opt/ohpc/pub/apps/hpl/2.3/bin/HPL.dat

sudo install -d -m 755 /opt/ohpc/pub/moduledeps/gnu12-openmpi4/hpl
sudo tee /opt/ohpc/pub/moduledeps/gnu12-openmpi4/hpl/2.3.lua >/dev/null <<'LUA'
help([[HPL 2.3 — High Performance Linpack,
built with GCC 12 + OpenMPI 4.1.5 + OpenBLAS.]])
whatis("Name: HPL")
whatis("Version: 2.3")
whatis("Category: benchmark")
local root = "/opt/ohpc/pub/apps/hpl/2.3"
prepend_path("PATH", pathJoin(root, "bin"))
setenv("HPL_DIR", root)
LUA
```

The modulefile sits under `/opt/ohpc/pub/moduledeps/gnu12-openmpi4/`
— the Lmod hierarchy slot reserved for software depending on
both GCC 12 and OpenMPI 4. Lmod only surfaces `hpl` after both
`gnu12` and `openmpi4` are loaded.

### What just happened

The researcher-facing stack is in place. A user logging in,
loading the right modules, and compiling a C+MPI program gets
the OpenHPC toolchain. The software tree lives at `/opt/ohpc/`
on head and will appear at `/opt/ohpc/` on every compute node
via NFS from `head-data:/export/apps`.

### Validation

```bash
source /etc/profile.d/lmod.sh
module avail
module load gnu12 openmpi4
module avail hpl
module load hpl omb
which xhpl
module list
```

Expected (condensed):

```
-------------------------- /opt/ohpc/pub/modulefiles --------------------------
   autotools   gnu12/12.2.0   hwloc/2.12.0   ... ucx/1.18.0

------------------- /opt/ohpc/pub/moduledeps/gnu12-openmpi4 -------------------
   hpl/2.3

/opt/ohpc/pub/apps/hpl/2.3/bin/xhpl

Currently Loaded Modules:
  1) gnu12/12.2.0   3) ucx/1.18.0         5) openmpi4/4.1.5   7) omb/6.1
  2) hwloc/2.12.0   4) libfabric/1.18.0   6) hpl/2.3
```

The moduledep hierarchy is the teaching point: `hpl` doesn't
appear in `module avail` output until both `gnu12` and
`openmpi4` are loaded, because HPL is linked against that
specific combination.

---

## Section 08 — Compute nodes come alive

### What this section does

Import a Rocky 9 base image into Warewulf. Install all the compute
packages in one shot, including the non-obvious ones — especially
`warewulf-ohpc-dracut` (the boot-time image fetcher),
`openblas-serial` (the actual BLAS library), `lua-posix` +
`lua-filesystem` (Lmod's runtime deps), and `node-exporter` (for
Section 10). Bake in secrets (Munge key, root pubkey), cluster
hostnames, users, slurmd config, log directories, and a dracut
config with `hostonly="no"`. Regenerate the initramfs with the
`wwinit` module included. Point the default profile at the image,
define both compute nodes on both fabrics, swap Warewulf's
`undionly.kpxe` for a full iPXE binary (VirtualBox's UNDI breaks
iPXE's built-in variant), enable DHCP with `always-broadcast on;`,
and create the diskless VMs. By the end, `sinfo` shows two `idle`
nodes and `srun` runs across them.

### Prerequisites

Sections 01-07 complete. This section takes ~15 min, dominated by
the image package install.

### Steps

1. **Import the Rocky 9 base image.**

```bash
sudo wwctl image import docker://rockylinux:9 rocky-9
sudo wwctl image list
```

Expected: `rocky-9` appears in the list.

2. **Install all compute-side packages in one shot.**

```bash
sudo tee /srv/warewulf/chroots/rocky-9/rootfs/tmp/install-compute.sh >/dev/null <<'SCRIPT'
#!/bin/bash
set -e
dnf install -y http://repos.openhpc.community/OpenHPC/3/EL_9/x86_64/ohpc-release-3-1.el9.x86_64.rpm
dnf install -y epel-release
dnf install -y \
    kernel-core kernel-modules dracut \
    nfs-utils \
    munge \
    openssh openssh-server openssh-clients \
    slurm-slurmd-ohpc-24.11.5-331.ohpc.1.1 \
    chrony \
    openblas-0.3.29-1.el9 openblas-serial-0.3.29-1.el9 \
    warewulf-ohpc-dracut-4.6.4-340.ohpc.1.3 \
    lua lua-posix lua-filesystem \
    node-exporter-1.11.1-1.el9
systemctl enable munge slurmd chronyd sshd node_exporter
SCRIPT
sudo chmod +x /srv/warewulf/chroots/rocky-9/rootfs/tmp/install-compute.sh
sudo wwctl image exec rocky-9 -- /tmp/install-compute.sh
```

Non-obvious packages and why:

- **`openblas-serial`** — on EL9, `openblas` is an umbrella
  package with no library content; `openblas-serial` actually
  ships `/lib64/libopenblas.so.0` that `xhpl` links against.
- **`warewulf-ohpc-dracut`** — the dracut module that downloads
  the root image over HTTP at boot. Without it, boot panics at
  "unable to mount root fs".
- **`lua lua-posix lua-filesystem`** — Lmod's runtime Lua deps;
  `lmod-ohpc` doesn't pull them.
- **`node-exporter`** — enabled now so Section 10 metrics work
  without a second compute reboot.

This step takes ~10-12 minutes.

3. **Bake in secrets and cluster context.**

```bash
IMG=/srv/warewulf/chroots/rocky-9/rootfs

# Munge key
sudo install -d -m 700 "$IMG/etc/munge"
sudo cp /etc/munge/munge.key "$IMG/etc/munge/munge.key"

# Root SSH pubkey (so head can ssh root@compute without a password)
sudo install -d -m 700 "$IMG/root/.ssh"
sudo cp /root/.ssh/id_ed25519.pub "$IMG/root/.ssh/authorized_keys"
sudo chmod 600 "$IMG/root/.ssh/authorized_keys"

# Lmod init script from head
sudo cp /etc/profile.d/lmod.sh "$IMG/etc/profile.d/lmod.sh"

# Cluster /etc/hosts — eliminates early-boot name-resolution surprises
sudo tee -a "$IMG/etc/hosts" >/dev/null <<'HOSTS'

# Cluster
10.0.0.1    head
10.0.0.11   compute01
10.0.0.12   compute02
10.1.0.1    head-data
10.1.0.11   compute01-data
10.1.0.12   compute02-data
HOSTS
```

4. **Customize users, slurmd, dracut, log dirs; regenerate initramfs.**

```bash
sudo tee /srv/warewulf/chroots/rocky-9/rootfs/tmp/customize.sh >/dev/null <<'SCRIPT'
#!/bin/bash
set -e

# HPC users with UIDs matching head; -M = no home creation (NFS-mounted)
useradd -u 1100 -M -d /home/hpcadmin -s /bin/bash -c 'HPC admin' hpcadmin 2>/dev/null || true
useradd -u 1101 -M -d /home/hpcuser  -s /bin/bash -c 'HPC user'  hpcuser  2>/dev/null || true

# Munge ownership — MUST be done inside chroot (image's munge UID may
# differ from head's). Fix the /etc/munge directory, not just the key.
chown munge:munge /etc/munge /etc/munge/munge.key
chmod 400 /etc/munge/munge.key

# Log dirs — munge refuses to start without /var/log/munge existing
install -d -m 700 -o munge -g munge /var/log/munge
install -d -m 755 -o root  -g root  /var/log/slurm

# slurmd configless — fetch slurm.conf from head:6817 at boot
mkdir -p /etc/sysconfig
echo 'SLURMD_OPTIONS="--conf-server=head"' > /etc/sysconfig/slurmd

# tmpfiles.d for /run/slurm
mkdir -p /etc/tmpfiles.d
echo 'd /run/slurm 0755 slurm slurm -' > /etc/tmpfiles.d/slurm.conf

# dracut: hostonly=no + include wwinit (critical for image-over-HTTP boot)
mkdir -p /etc/dracut.conf.d
cat > /etc/dracut.conf.d/99-warewulf.conf <<'CONF'
hostonly="no"
add_dracutmodules+=" wwinit "
CONF

# NFS mount points (the fstab entries come from WW profile overlay)
mkdir -p /home /opt/ohpc /scratch

# SELinux permissive to match head
[ -f /etc/selinux/config ] && sed -i 's/^SELINUX=.*/SELINUX=permissive/' /etc/selinux/config

# Regenerate initramfs with wwinit baked in
KVER=$(ls /lib/modules/ | head -1)
dracut --force --kver "$KVER" "/boot/initramfs-${KVER}.img"
SCRIPT
sudo chmod +x /srv/warewulf/chroots/rocky-9/rootfs/tmp/customize.sh
sudo wwctl image exec rocky-9 -- /tmp/customize.sh
```

5. **Replace Warewulf's bundled `undionly.kpxe` with a full iPXE binary.**

```bash
cd /tmp
curl -sfLO https://boot.ipxe.org/ipxe.pxe
sudo cp /srv/tftpboot/warewulf/undionly.kpxe /srv/tftpboot/warewulf/undionly.kpxe.orig
sudo cp /tmp/ipxe.pxe /srv/tftpboot/warewulf/undionly.kpxe
```

VirtualBox's UNDI interface has a known bug that prevents iPXE's
undionly variant from transmitting DHCP packets. The full
`ipxe.pxe` binary ships its own native Intel NIC drivers and
bypasses UNDI entirely. Same filename on disk — no DHCP config
change needed.

6. **Configure the default Warewulf profile.**

```bash
sudo wwctl profile set --yes default \
  --image=rocky-9 \
  --kernelargs='crashkernel=no net.ifnames=0 biosdevname=0 console=ttyS0,115200 console=tty0' \
  --tagadd=IPXEMenuEntry=dracut

sudo python3 <<'PY'
import yaml
with open('/etc/warewulf/nodes.conf') as f:
    cfg = yaml.safe_load(f)
cfg['nodeprofiles']['default']['resources']['fstab'] = [
  {'file': '/home',     'spec': '10.1.0.1:/export/home',    'vfstype': 'nfs', 'mntops': 'defaults,_netdev,nofail,vers=4'},
  {'file': '/opt/ohpc', 'spec': '10.1.0.1:/export/apps',    'vfstype': 'nfs', 'mntops': 'defaults,_netdev,nofail,vers=4,ro'},
  {'file': '/scratch',  'spec': '10.1.0.1:/export/scratch', 'vfstype': 'nfs', 'mntops': 'defaults,_netdev,nofail,vers=4'},
]
with open('/etc/warewulf/nodes.conf', 'w') as f:
    yaml.dump(cfg, f, default_flow_style=False, sort_keys=False)
PY
```

Three notable settings:

- `IPXEMenuEntry=dracut` forces the two-stage dracut boot. The
  BIOS default loads the whole 1.4 GB image as initrd, which
  OOMs in a 2 GB tmpfs. (Compute VMs have 4 GB; tmpfs default is
  ½ RAM = 2 GB.) The dracut method fetches the image over HTTP
  into a roomy tmpfs after it's mounted.
- `Kernel.Args` — classic `ethN` names + serial console to
  file (handy for debugging in `compute0N.console.log`).
- `Resources[fstab]` uses `10.1.0.1:` literal IP rather than
  `head-data:` because Warewulf's `hosts` overlay doesn't
  include head's fabric aliases.

7. **Define both compute nodes (both fabrics upfront).**

```bash
# hpc-mgmt fabric (PXE boots through this NIC)
sudo wwctl node add compute01 --ipaddr=10.0.0.11 --netmask=255.255.255.0 \
  --netdev=eth0 --hwaddr=52:54:00:C0:01:01 --type=ethernet --primarynet=default
sudo wwctl node add compute02 --ipaddr=10.0.0.12 --netmask=255.255.255.0 \
  --netdev=eth0 --hwaddr=52:54:00:C0:01:02 --type=ethernet --primarynet=default

# hpc-data fabric (NFS + MPI)
sudo wwctl node set --yes compute01 --netname=data --netdev=eth1 \
  --hwaddr=52:54:00:D0:01:01 --ipaddr=10.1.0.11 --netmask=255.255.255.0 --type=ethernet
sudo wwctl node set --yes compute02 --netname=data --netdev=eth1 \
  --hwaddr=52:54:00:D0:01:02 --ipaddr=10.1.0.12 --netmask=255.255.255.0 --type=ethernet
```

MACs are pinned; `scripts/create-compute-vm.sh` assigns matching
MACs to the VirtualBox VMs it creates.

8. **Enable DHCP + `always-broadcast`, start dhcpd.**

```bash
# Flip dhcp.enabled: false -> true in warewulf.conf
sudo sed -i '/^dhcp:/,/^[a-z]/{s|^    enabled: false|    enabled: true|}' /etc/warewulf/warewulf.conf

# Regenerate dhcpd.conf, tftp, overlays
sudo wwctl configure --all

# Force DHCP replies to be broadcast. VirtualBox internal-network
# driver drops unicast DHCP to unconfigured clients, which breaks
# iPXE's DHCP handshake.
sudo sed -i '/subnet 10.0.0.0 netmask 255.255.255.0 {/a\    always-broadcast on;' /etc/dhcp/dhcpd.conf

sudo systemctl enable --now dhcpd
sudo wwctl overlay build
```

Verify:

```bash
sudo ss -tulnp | awk '/:(9873|69|67) / || NR==1'
```

Expected: three listeners — warewulfd (tcp/9873), in.tftpd
(udp/69), dhcpd (udp/67).

9. **Create the compute VMs from the Windows host.**

Open **Git Bash on the Windows host** (not inside `vagrant ssh`
— the compute VMs need `VBoxManage`, which is a host command)
and run:

```bash
./scripts/create-compute-vm.sh 1
./scripts/create-compute-vm.sh 2
```

Each invocation creates a diskless VirtualBox VM with 4 GB RAM,
2 vCPU, ich9/IOAPIC/HPET firmware, two NICs on the `hpc-mgmt` /
`hpc-data` internal networks with pinned MACs, PXE boot from
NIC 1, and a serial console redirected to `compute0N.console.log`
in the project directory for boot debugging.

10. **Wait for both nodes to reach `idle`.**

Back inside head:

```bash
sinfo
```

Expected within ~90 seconds per node:

```
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
normal*      up   infinite      2   idle compute[01-02]
```

### What just happened

Two compute nodes booted from nothing. They pulled their root
filesystem over HTTP from head's warewulfd, extracted it into
RAM (tmpfs), switched root, and came online as fully functional
Linux hosts. The Warewulf overlay system gave each its own
hostname, IP addresses on both fabrics, NFS mounts for `/home`
and `/opt/ohpc`, and a Munge key shared with head. Their
`slurmd` daemons fetched `slurm.conf` from `slurmctld` over port
6817 and registered themselves in the `normal` partition.

This is real HPC compute provisioning: stateless, image-driven,
network-booted. A node is cattle — provisioned, used,
decommissioned, re-provisioned with zero human ceremony. If
compute01 dies, power a replacement with the same MAC on the
same fabric and it rejoins in two minutes with no config
change on head.

### Validation

```bash
sinfo
sudo -u hpcadmin srun -N2 -n2 --export=ALL hostname
```

Expected: both `compute01` and `compute02` print (order varies).

---

## Section 09 — Run a real job

### What this section does

Four demonstrations, escalating in weight:

1. `srun hostname` across both nodes — Slurm connectivity sanity.
2. MPI hello world — proves MPI is wired and inter-node launching works.
3. HPL across both compute nodes — produces a real GFLOPS number.
4. OSU bandwidth + latency — characterizes the fabric between them.

### Prerequisites

Section 08 complete. Both compute nodes in `idle`. HPL and OSU
Micro-Benchmarks installed under `/opt/ohpc/` (Section 07).

### Steps

1. **Log in as `hpcuser`.**

```bash
sudo -iu hpcuser
mkdir -p ~/section09 && cd ~/section09
```

2. **Sanity: `srun` across both nodes.**

```bash
srun -N2 -n2 --export=ALL hostname
```

Expected (order varies):

```
compute01
compute02
```

If both come back, Slurm placement is working, Munge auth is
holding, and slurmd on both computes is responding.

3. **MPI hello world.**

```bash
cat > mpi_hello.c <<'EOF'
#include <mpi.h>
#include <stdio.h>
#include <unistd.h>
int main(int argc, char **argv) {
    int rank, size;
    char host[256];
    MPI_Init(&argc, &argv);
    MPI_Comm_size(MPI_COMM_WORLD, &size);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    gethostname(host, sizeof host);
    printf("Hello from rank %d of %d on %s\n", rank, size, host);
    MPI_Finalize();
    return 0;
}
EOF

source /etc/profile.d/lmod.sh
module load gnu12 openmpi4
mpicc -O2 mpi_hello.c -o mpi_hello
srun -N2 -n4 --export=ALL ./mpi_hello | sort
```

Expected:

```
Hello from rank 0 of 4 on compute01
Hello from rank 1 of 4 on compute01
Hello from rank 2 of 4 on compute02
Hello from rank 3 of 4 on compute02
```

**`--export=ALL` is load-bearing.** Without it, `srun` scrubs
the `LD_LIBRARY_PATH` that `module load` set on the submitting
shell, and the binary fails to find OpenHPC shared libraries on
compute. Use `--export=ALL` on every MPI `srun` invocation.

4. **HPL across both compute nodes.**

Use the *full* HPL.dat below. HPL 2.3 is strict about input-file
format; abbreviated variants fail with "Illegal input in file
HPL.dat".

```bash
mkdir -p ~/section09/hpl-run && cd ~/section09/hpl-run

cat > HPL.dat <<'EOF'
HPLinpack benchmark input file
Innovative Computing Laboratory, University of Tennessee
HPL.out      output file name (if any)
6            device out (6=stdout,7=stderr,file)
1            # of problems sizes (N)
7000         Ns
1            # of NBs
128          NBs
0            PMAP process mapping (0=Row-,1=Column-major)
1            # of process grids (P x Q)
2            Ps
2            Qs
16.0         threshold
1            # of panel fact
2            PFACTs (0=left, 1=Crout, 2=Right)
1            # of recursive stopping criterium
4            NBMINs (>= 1)
1            # of panels in recursion
2            NDIVs
1            # of recursive panel fact.
2            RFACTs (0=left, 1=Crout, 2=Right)
1            # of broadcast
0            BCASTs (0=1rg,1=1rM,2=2rg,3=2rM,4=Lng,5=LnM)
1            # of lookahead depth
0            DEPTHs (>=0)
2            SWAP (0=bin-exch,1=long,2=mix)
64           swapping threshold
0            L1 in (0=transposed,1=no-transposed) form
0            U  in (0=transposed,1=no-transposed) form
1            Equilibration (0=no,1=yes)
8            memory alignment in double (> 0)
EOF

module load hpl
time srun -N2 -n4 --export=ALL $(which xhpl) > hpl.out 2>&1
grep -A1 'T/V' hpl.out
grep -E '(PASSED|FAILED)' hpl.out | head -1
```

Expected (numbers vary):

```
T/V                N    NB     P     Q               Time                 Gflops
--------------------------------------------------------------------------------
WR00R2R4        7000   128     2     2              16.80             1.3611e+01
...
||Ax-b||_oo/(eps*(||A||_oo*||x||_oo+||b||_oo)*N)=  2.97e-03 ...... PASSED
```

**~12-16 GFLOPS** on this 2-node toy cluster. Frame the number:

- Modern workstation CPU (single socket): 50-200 GFLOPS.
- Top-500 leader (2026): ~1 exaflop = 10⁹ GFLOPS.
- This cluster: ~14 GFLOPS.

Seven orders of magnitude below the world's biggest, but the
*shape* is identical. Every Top-500 entry runs HPL the same
way, over InfiniBand instead of Ethernet, at N ≈ 10⁷ instead of
5 × 10³, across millions of cores instead of four.

5. **OSU bandwidth + latency.**

```bash
module load omb
OMB=$(dirname $(which osu_latency))

echo "=== osu_bw ==="
srun -N2 -n2 --export=ALL $OMB/osu_bw | tail -25

echo "=== osu_latency ==="
srun -N2 -n2 --export=ALL $OMB/osu_latency | tail -25
```

Expected (shape, numbers vary):

- Peak bandwidth ~200-230 MB/s at 1-4 MB messages
- Small-message latency ~140 µs
- Latency grows to ~5 ms at 1 MB messages

For comparison, a production InfiniBand fabric hits 12+ GB/s
bandwidth and 1-2 µs latency — two orders of magnitude better
on both axes. Seeing these numbers side by side is the moment
the fabric choice in the Section 01 diagram (plain Ethernet vs.
InfiniBand) stops being abstract.

### What just happened

The cluster ran four jobs escalating from trivial to meaningful:

- `hostname` proves Slurm places work and returns results.
- MPI hello world proves the MPI runtime assembles correctly
  across nodes and the PMIx launcher works.
- HPL produces a real GFLOPS number, passing residual checks.
- OSU measures actual fabric capacity.

Every job was tracked in `slurmdbd`'s accounting DB. Every rank
launched on a compute node authenticated back to the controller
via Munge. Every binary ran off NFS from `/opt/ohpc/` on head's
exported `/export/apps`. No file was ever copied to a compute
node.

### Validation

```bash
sacct --format=JobID,JobName,Partition,NNodes,NodeList,State,Elapsed | tail -8
```

Expected: entries for `hostname`, `mpi_hello`, `xhpl`, `osu_bw`,
`osu_latency`, all in `COMPLETED` state on `compute[01-02]` in
the `normal` partition.

---

## Section 10 — See it running

### What this section does

Install Prometheus on head, ensure `node_exporter` is active on
head and both compute nodes, and install Grafana with a pre-loaded
HPC dashboard. Port 3000 is NAT-forwarded by the Vagrantfile so
Grafana opens at `http://localhost:3000` on the Windows host.
Re-run HPL and watch CPU saturation spread across both compute
nodes live.

Observability without something to observe is a screensaver. The
payoff here is watching an MPI job light up the dashboard —
seeing HPL push compute01 and compute02 to 100% CPU busy at the
same moment, memory climb as the matrix loads, then everything
return to idle.

### Prerequisites

Sections 01-09 complete. Both compute nodes in `idle`.
`node_exporter` was installed into the compute image in Section 08
and enabled at boot — it's already scraping.

### Steps

1. **Install Prometheus + node-exporter on head.**

```bash
sudo dnf install -y prometheus-3.11.2-1.el9 node-exporter-1.11.1-1.el9
```

2. **Write `/etc/prometheus/prometheus.yml`.**

```bash
sudo tee /etc/prometheus/prometheus.yml >/dev/null <<'EOF'
global:
  scrape_interval: 5s
  evaluation_interval: 15s

scrape_configs:
  - job_name: head
    static_configs:
      - targets: ['localhost:9100']
        labels: { role: head }
  - job_name: compute
    static_configs:
      - targets: ['10.0.0.11:9100']
        labels: { host: compute01, role: compute }
      - targets: ['10.0.0.12:9100']
        labels: { host: compute02, role: compute }
EOF
sudo chown prometheus:prometheus /etc/prometheus/prometheus.yml
```

5-second scrape interval is tighter than Prometheus' default
(15 s) — useful for a live HPL demo where a job runs for 5-10
seconds. Three scrape targets: head plus the two compute nodes.

3. **Start Prometheus + node_exporter on head.**

```bash
sudo systemctl enable --now node_exporter prometheus
```

Verify all three scrape targets are `up`:

```bash
curl -s http://localhost:9090/api/v1/targets \
  | python3 -c 'import json,sys; d=json.load(sys.stdin); [print(t["labels"].get("host") or t["labels"].get("role","head"), t["scrapeUrl"], t["health"]) for t in d["data"]["activeTargets"]]'
```

Expected:

```
head        http://localhost:9100/metrics  up
compute01   http://10.0.0.11:9100/metrics  up
compute02   http://10.0.0.12:9100/metrics  up
```

4. **Install Grafana from the Grafana OSS repo.**

```bash
sudo tee /etc/yum.repos.d/grafana.repo >/dev/null <<'EOF'
[grafana]
name=grafana
baseurl=https://rpm.grafana.com
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://rpm.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
EOF
sudo dnf install -y grafana-13.0.1-1
```

5. **Provision the Prometheus data source and the HPC dashboard.**

```bash
sudo tee /etc/grafana/provisioning/datasources/prometheus.yaml >/dev/null <<'EOF'
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://localhost:9090
    isDefault: true
EOF

sudo install -d -o grafana -g grafana /var/lib/grafana/dashboards
sudo tee /etc/grafana/provisioning/dashboards/default.yaml >/dev/null <<'EOF'
apiVersion: 1
providers:
  - name: hpc-cluster
    orgId: 1
    folder: HPC
    type: file
    disableDeletion: false
    editable: true
    options:
      path: /var/lib/grafana/dashboards
EOF

sudo cp /vagrant/scripts/grafana-hpc-cluster.json /var/lib/grafana/dashboards/hpc-cluster.json
sudo chown grafana:grafana /var/lib/grafana/dashboards/hpc-cluster.json
```

The `/vagrant/` path is Vagrant's rsync-mounted project
directory inside head. The dashboard JSON lives in
[scripts/grafana-hpc-cluster.json](scripts/grafana-hpc-cluster.json)
— CPU busy %, memory used %, load average, eth1 MPI-fabric
bytes/s — across all nodes.

6. **Start Grafana.**

```bash
sudo systemctl enable --now grafana-server
sleep 5
curl -s http://localhost:3000/api/health
```

Expected:

```json
{"database": "ok", "version": "13.0.1", "commit": "..."}
```

7. **Open Grafana from the Windows host browser.**

Grafana (3000) and Prometheus (9090) are NAT-forwarded through
Vagrant's default adapter — the Vagrantfile has:

```ruby
config.vm.network "forwarded_port", guest: 3000, host: 3000, id: "grafana"
config.vm.network "forwarded_port", guest: 9090, host: 9090, id: "prometheus"
```

Open `http://localhost:3000` on the Windows host. Default login:
`admin` / `admin`. Grafana prompts to change the password on
first login. After login, navigate to **Dashboards → HPC → HPC
Cluster**.

Four panels are provisioned:

- **CPU Busy (%)** — `100 * (1 - avg(rate(node_cpu_seconds_total{mode="idle"}[30s])))` per host
- **Memory Used (%)** — `1 - MemAvailable/MemTotal`
- **Load Average (1m)** — `node_load1`
- **eth1 MPI-fabric bytes/s** — RX+TX on each node's data-fabric NIC

8. **Re-run HPL and watch the dashboard light up.**

Keep Grafana open. From head as `hpcuser`:

```bash
cd ~/section09/hpl-run
source /etc/profile.d/lmod.sh
module load gnu12 openmpi4 hpl
srun -N2 -n4 --export=ALL $(which xhpl) > hpl-live.out 2>&1 &
```

During the ~5-second HPL run:

- **CPU Busy** jumps to ~100 % on both compute nodes
  simultaneously — matrix multiply saturating all cores.
- **Memory Used** climbs ~200 MB on each compute as HPL
  allocates matrix pieces.
- **Load Average** lags CPU by a few seconds (1-minute average).
- **eth1** shows a modest spike as MPI moves panels between
  ranks.

### What just happened

A standard enterprise-facing observability stack (Prometheus +
Grafana) is now watching the cluster. The pattern is identical
to what an enterprise IT team already runs for their
infrastructure; the only thing HPC-specific is which metrics
matter (CPU saturation, job-queue depth, fabric bandwidth —
rather than, say, HTTP request latency or database connection
counts).

Observability was boring in isolation; it became useful the
moment a real job started generating real metrics. That's how
operators learn to read a dashboard — not by staring at flat
lines, but by watching an incident (or a scheduled HPL) make
the metrics move.

Production HPC sites extend this stack with Slurm-aware
exporters (`prometheus-slurm-exporter` built from source — no
EPEL package yet), job-level accounting panels, GPU metrics if
applicable, fabric-aware exporters for InfiniBand counters, and
alerting rules on things like "node marked down for >5m" or
"scratch filesystem >80% full."

### Validation

With HPL running, CPU Busy on both compute01 and compute02
touches or exceeds 95 %. After HPL finishes:

```bash
grep -A1 'WR' hpl-live.out
```

Produces a GFLOPS line similar to the earlier run.

---

## Section 11 — What we skipped and what's next

### What this section does

Close honestly. The bootcamp built a working 1+2 cluster and ran
real jobs on it in four hours. That is a lot, and it is also a
sliver of what production HPC is. This section names what we did
not cover, then gives role-based next steps so you leave with a
map rather than a destination.

### The promise, re-stated

You saw diskless compute nodes PXE-boot from a provisioning
server and come online authenticated and NFS-mounted. You
watched Slurm place a job across both of them, HPL produce a
GFLOPS number, OSU measure the fabric, and Grafana render the
work in real time. The shape of the demonstration is identical
at every scale — from this 4-hour toy to the Top-500. Different
components, different vendors, different orders of magnitude;
same pieces in the same places.

### What this bootcamp did not cover

A deliberately incomplete list, in roughly the order a growing
site encounters each:

- **Parallel filesystems.** We used NFS for `/home` and
  `/opt/ohpc`. Production sites run **Lustre**, **IBM Storage
  Scale (GPFS)**, or **BeeGFS** — filesystems that stripe data
  across many servers and scale out past NFS's hard limits.
- **High-performance fabric.** Our `hpc-data` is plain 1 GbE.
  OSU measured ~140 µs latency. Real HPC fabrics are
  **InfiniBand** (HDR, NDR) or **Slingshot** at 1-5 µs latency,
  100-400 Gb/s. MPI over IB isn't just faster — the collective
  algorithms change.
- **Container workloads.** Researchers increasingly ship
  scientific software as containers. **Apptainer** (formerly
  Singularity) is the HPC-native runtime — rootless, MPI-aware,
  works inside Slurm jobs.
- **User-facing software management.** OpenHPC gives you a
  packaged stack. **Spack** is how sites build and deliver the
  long tail — hundreds of scientific libraries, thousands of
  compiler/MPI/variant combinations.
- **Identity at cluster scale.** We used local users. Real sites
  run **FreeIPA** / **LDAP+SSSD** (academic) or hook into
  **Active Directory** (commercial).
- **Security hardening.** SELinux permissive, firewalld disabled
  — our lab posture. Production enforces SELinux, segments
  fabrics, requires SSH certificates or MFA, and in regulated
  environments adds **GxP** (pharma) or **ITAR/NIST 800-171**
  (defense) compliance.
- **High-availability head.** One head is a single point of
  failure. Production runs two head nodes in active/passive
  with shared storage for `/var/spool/slurmctld` and
  `/etc/slurm`, managed by Pacemaker/Corosync, with `slurmdbd`
  replicated to a MariaDB cluster.
- **GPUs.** CUDA-capable accelerators are in a majority of new
  HPC deployments. The full story (GRES, NVLink, driver
  pinning, MIG, CUDA-aware MPI) is its own field.
- **Power and cooling.** A real node draws 500-1500 W; a cabinet
  is 20-50 kW. HPC is as much an HVAC problem as a compute one.
- **Scheduler depth.** We ran Slurm on default settings.
  Production Slurm has **fairshare priority**, **QoS tiers**,
  **job arrays**, **heterogeneous jobs**, **reservations**,
  **preemption**, **TRES billing**.
- **Backup and DR.** `/home` and `/apps` should be backed up;
  `/scratch` is disposable. We set up no backups.

### Community, conferences, certifications

- **SC** (ACM/IEEE Supercomputing, November, US) — the flagship
  conference.
- **ISC High Performance** (June, Germany) — research-focused
  European counterpart.
- **PEARC** (July, US) — campus-scale HPC operators.
- **CaRCC** (Campus Research Computing Consortium) — active
  community for academic research computing.
- **HPC Certification Forum** — structured skills framework and
  exams.
- **Beowulf mailing list** — oldest HPC-sysadmin list, still
  active.
- Books: *High Performance Computing: Modern Systems and
  Practices* (Sterling, Anderson, Brodowicz).

### Closing

The cluster you watched boot this afternoon is a miniature of
every HPC system in the world. The same pattern — diskless
compute, job scheduling with accounting, parallel filesystems,
message-passing libraries, observability — scales through every
order of magnitude. When you see it at a vendor briefing next
quarter, when a researcher asks why their job is queueing, when
your procurement team asks whether cloud or on-prem, you now
have a map. Follow the role that pulls you most.

---

## Post-bootcamp self-study

You watched the cluster come up; now make it yours. The exercises
below are deliberately **hints, not solutions**. You have everything
you need in the runbook above and in `man`. Struggling through them
is the point.

### Exercise 1 — Make compute boot faster

**Observation.** During Section 08 you saw compute nodes spend ~90
seconds at `A start job is running for Wait for Network to be
Configured` before login. That's `NetworkManager-wait-online.service`
waiting on interfaces that (for a diskless compute node with one
active link) don't need that much hand-holding.

**Goal.** Shave that 90 s off the compute boot, rebuild the image,
and re-provision the two compute nodes.

**Hints, not recipes.**

- The service to disarm lives in the **image**, not on the head. Use
  `wwctl image exec <imagename> -- systemctl disable ...` to act
  inside the chroot, then `wwctl image build`.
- Compute nodes are diskless — the change only takes effect after
  they PXE-boot again. `wwctl node set --ipxe=reboot` + a
  `VBoxManage controlvm compute0N reset` is one way.
- Time the boot with `systemd-analyze` on the compute after it
  reconverges. Compare to your Section 08 timing.

### Exercise 2 — Bring up compute03

**Goal.** Add a third compute node (`compute03`, MAC ending in `:03`,
IP `10.10.0.103`) without touching anything that's already working.

**Hints.**

- The head already knows how to provision an arbitrary compute — the
  DHCP/TFTP/image pipeline doesn't care whether there's one node or
  ten. What's missing is a **node definition** in Warewulf
  (`wwctl node add`) and a **VirtualBox VM** that matches.
- `scripts/create-compute-vm.sh` is parameterized. Read it. Adapt it
  — or better, extend it to take the node number as an argument.
- Slurm needs to know about the new node too. `NodeName=compute03`
  in `slurm.conf`, then `scontrol reconfigure`. Check `sinfo` before
  and after.
- Don't forget to size the head: 16 GB host RAM with head 4 GB +
  two computes 4 GB each is already tight. A third 4 GB compute
  may force you to drop head or each compute to 2-3 GB. Measure
  before assuming.

### Exercise 3 — Add a Slurm-aware exporter

**Goal.** Make `sinfo`-style cluster state visible in Grafana without
running `sinfo` by hand.

**Hints.**

- `prometheus-slurm-exporter` (the vpenso/prometheus-slurm-exporter
  Go project) isn't in EPEL yet. Build it from source: `golang`
  is in Rocky 9 AppStream; `go install` or `make` in the repo
  produces a single static binary. A minimal systemd unit plus
  a scrape target in `prometheus.yml` wires it in.
- It exposes `slurm_nodes_idle`, `slurm_jobs_pending`,
  `slurm_queue_pending`, and friends. Build a Grafana panel that
  graphs them.
- Alternatively, add a **simpler** exporter — `process-exporter`
  scoped to `slurmd` and `munged` (available as a prebuilt RPM in
  some Copr repos). Point Prometheus at it, verify scrape targets
  go green, build a panel.
- The Grafana dashboard JSON you imported is editable. Right-click
  a panel → Edit → change the PromQL → Apply → Save.

### What "done" looks like

For each exercise, the finished state is: the cluster still passes
the [Section 09](#section-09-run-a-real-job) MPI job, and you can
explain to someone else what changed and why. If you broke the
cluster, `vagrant snapshot restore s07` (or whichever snapshot is
pre-Section 08) gets you back to a known good — this is why the
snapshots exist.

---

## Cleanup / start over

If anything goes sideways and you want to start fresh, run the
provided cleanup script from the **Windows host** in Git Bash:

```bash
./scripts/cleanup.sh
```

The script is paste-safe (no heredocs, no complex loops) so a copy
from the HTML and paste into Git Bash works without surprises. It
powers off and deletes `compute01` + `compute02`, runs `vagrant
destroy -f` on the head, clears console logs and `.vagrant/`, and
prints the final state. Individual steps resume if earlier ones
fail — safe to re-run.

What's preserved:

- The Rocky 9 base box stays cached at
  `~/.vagrant.d/boxes/rockylinux-VAGRANTSLASH-9/9.7.0/`. Next
  `vagrant up` skips the ~1 GB box download.
- Git history is intact.
- All other repo files (RUNBOOK.md, scripts, etc.) are intact.

What's gone:

- Both compute VMs and the head VM (their VirtualBox VM
  directories under `C:\Users\<you>\VirtualBox VMs\`).
- Any user data on `/home`, `/scratch`, or `/opt/ohpc` from the
  old run. The runbook rebuilds these from scratch.

After cleanup, restart from [Section 02](#section-02-bring-up-the-head-node).
