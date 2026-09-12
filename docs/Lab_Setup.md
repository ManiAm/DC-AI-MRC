# Lab Setup

This guide walks through the one-time setup for the MRC emulator — system requirements, software installation, and deploying your first topology. Once complete, return to the [main README](../README.md#the-srctl-cli) to run traffic scenarios.

---

## Requirements

- **Native Linux with kernel ≥ 6.1** — required for uSID support (see [Kernel Prerequisites](#kernel-prerequisites) for details). On Ubuntu, this means **24.04 LTS** (ships with 6.8) or **22.04 LTS with the HWE kernel**.
- **Hardware** — at least 16 vCPU and 32 GB RAM for the `2p-4x8` topology. The `4p-8x16` topology needs roughly 3× more resources.
- **Docker** (v20.10+)
- **containerlab** (v0.50+)

### Kernel Prerequisites

The emulator's SONiC switches and host containers all share the **host machine's Linux kernel** for network processing. Three kernel features must be present, each building on the previous one:

#### 1. Lightweight tunnels (`CONFIG_LWTUNNEL`)

SRv6 uses the kernel's lightweight-tunnel (`lwtunnel`) infrastructure to attach encapsulation and decapsulation actions to routes. Without this, the kernel cannot install or process any SRv6 route.

#### 2. SRv6 seg6local actions (`CONFIG_IPV6_SEG6_LWTUNNEL`)

On top of `lwtunnel`, the `seg6local` module provides the [SRv6 endpoint actions](../README.md#srv6-usid-and-endpoint-actions) (`End`, `End.X`, `End.DT6`) that each switch and host container uses to process SRv6 packets.

#### 3. uSID / next-csid flavor (`CONFIG_IPV6_SEG6_NEXT_CSID`) — requires kernel ≥ 6.1

The emulator uses [uSID encoding](../README.md#srv6-usid-and-endpoint-actions) rather than a full SRH. For this to work, the kernel's `seg6local` actions need the **next-csid flavor**, which teaches `End` and `End.X` how to shift micro-segments in the destination address instead of reading from an SRH. This flavor was introduced in **Linux kernel 6.1**.

**Without it, the switches silently drop every packet** because the standard `End` / `End.X` actions see no SRH and discard the packet. Senders will report packets sent, but receivers will receive nothing.

#### Verification

Run these commands on the **host machine** (the physical or virtual machine running the emulator, not a container) to confirm your system is ready. In the kernel config output, `=y` means the feature is built directly into the kernel (always active), while `=m` means it is compiled as a loadable module (may need to be loaded manually):

```bash
# 1. Kernel version — must be 6.1 or later
uname -r

# 2. Lightweight tunnels
grep CONFIG_LWTUNNEL /boot/config-$(uname -r)
# Expect: CONFIG_LWTUNNEL=y

# 3. SRv6 seg6local
grep CONFIG_IPV6_SEG6 /boot/config-$(uname -r)
# Expect: CONFIG_IPV6_SEG6_LWTUNNEL=y
#         CONFIG_IPV6_SEG6_HMAC=y
```

If `seg6` and `vrf` appear as loadable modules (`=m` instead of `=y`), load them explicitly:

```bash
sudo modprobe seg6
sudo modprobe vrf
```

If they are built directly into the kernel (`=y`), `modprobe` will report "Module not found" — this is normal and means the support is already active.

---

## Installation

### 1. Install Docker

If Docker is not already installed:

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
newgrp docker
```

### 2. Install containerlab

```bash
curl -sL https://containerlab.dev/setup | sudo bash -s install-containerlab
```

> **Note:** The setup script uses a function-dispatch pattern (`"$@"`). You **must** pass the function name via `bash -s <function>`, otherwise the script exits silently. Use `install-containerlab` to install only containerlab, or `all` to install Docker + containerlab together.

> **Note:** On systems with pending kernel upgrades, the install script may show an interactive "Pending kernel upgrade" dialog. Set `DEBIAN_FRONTEND=noninteractive` before running, or press Enter/Ok to dismiss it.

Verify the install (containerlab installs to `/usr/bin/containerlab`):

```bash
containerlab version
```

### 3. Download and load the SONiC VS Docker image

The SONiC build system publishes two Virtual Switch (VS) artifacts — they run the same SONiC software but are packaged differently:

| Artifact             | Format          | What it is |
|----------------------|-----------------|------------|
| `sonic-vs.img.gz`    | QEMU disk image | A full VM image — boot it under QEMU/KVM to get a complete SONiC system. |
| `docker-sonic-vs.gz` | Docker image    | SONiC packaged as a single Docker container — this is what Containerlab uses. |

The emulator uses `docker-sonic-vs`. Inside this top-level container, SONiC runs its standard architecture of **nested Docker containers** just as it would on real hardware. Containerlab manages the outer container; SONiC manages the inner ones.

Download the latest master branch `docker-sonic-vs` image from the [SONiC build system](https://sonic.software/):

```bash
wget -O /tmp/docker-sonic-vs.gz \
  "https://sonic-build.azurewebsites.net/api/sonic/artifacts?branchName=master&platform=vs&target=target%2Fdocker-sonic-vs.gz"
```

Verify the download is a valid gzip file, then load it:

```bash
file /tmp/docker-sonic-vs.gz
# Should say: gzip compressed data

docker load -i /tmp/docker-sonic-vs.gz
# Should say: Loaded image: docker-sonic-vs:latest
```

### 4. Build the host container image

Build the Alpine SRv6 host image from source. This bakes in the `srv6_mrc` Python package and CLI tools (`srctl`, `spray`, `routes`, `run-scenario`):

```bash
cd emulator
make image
```

This produces `alpine-srv6-scapy:1.0`. One image serves every topology — the active `topo.yaml` is bind-mounted into each host container at runtime by containerlab.

> **Note:** Do not substitute `docker pull bmcdougall/alpine-srv6-scapy:1.0` — the Docker Hub image is a base image and may not contain the latest `srv6_mrc` CLI tools.

### 5. Check for Docker network conflicts

The containerlab management network uses `172.20.18.0/24` (defined in `topology.clab.yaml`). Verify no existing Docker network overlaps:

```bash
docker network inspect $(docker network ls -q) 2>/dev/null | grep '"Subnet"'
```

If any subnet overlaps `172.20.0.0/16`, either remove the conflicting network (`docker network rm <name>`) or edit the `ipv4-subnet` field in `emulator/topologies/<topo>/topology.clab.yaml`.

---

## Deploy and Configure

All commands below use the `2p-4x8` topology as an example. Replace `TOPO=2p-4x8` with any topology from the [topologies table](../README.md#emulator) above. Run from the `emulator/` directory.

### Deploy the topology

```bash
sudo make TOPO=2p-4x8 deploy
```

This runs `containerlab deploy` and creates all containers (40 for `2p-4x8`: 8 spines + 16 leaves + 8 green hosts + 8 yellow hosts). All containers should show `running` state with no errors.

### Push SONiC configuration

```bash
sudo make TOPO=2p-4x8 config
```

This pushes `config_db.json` and `frr.conf` into every SONiC switch container, reloads the configuration, and auto-verifies that SRv6 SIDs are correctly programmed on all leaves. Takes 1–3 minutes. Wait for `Configuration complete!` at the end.

### Install host routes

```bash
sudo make TOPO=2p-4x8 host-routes
```

Applies `topologies/2p-4x8/routes/full-mesh.yaml` to set up full-mesh SRv6 routing between all hosts.

### Run a traffic scenario

```bash
sudo make TOPO=2p-4x8 scenario SCEN=yellow-baseline
```

This invokes the scenario orchestrator. See [Running Traffic Scenarios](../README.md#running-traffic-scenarios) for the full list of traffic patterns, spray policies, and example commands.

### Tear down

> **Note:** Don't tear down the topology yet — the [srctl CLI](../README.md#the-srctl-cli), [spray](../README.md#the-spray-tool), and [iperf3](../README.md#fabric-connectivity-with-iperf3) sections in the main README all run against the live fabric. Destroy it only when you are done with the lab.

```bash
sudo make TOPO=2p-4x8 destroy
```
