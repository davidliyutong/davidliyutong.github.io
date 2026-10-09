---
layout: post
title: InfiniBand Quick Command Reference
date: 2025-10-28 00:01:00
description: A cheat sheet for bringing up InfiniBand on Ubuntu, from packages and firmware to IPoIB, OpenSM and benchmarks.
tags: infiniband rdma network IT
categories: IT
---

This is a cheat sheet of the commands I use to set up InfiniBand on Ubuntu with Mellanox/NVIDIA ConnectX cards. On Ubuntu 24.04 you no longer need to install the driver from the vendor ISO (MLNX_OFED), because the in-box drivers are enough.

## Install RDMA packages

```shell
sudo apt-get install rdma-core
sudo apt-get install libibverbs1 librdmacm1 libibmad5 libibumad3 ibverbs-providers rdmacm-utils infiniband-diags libfabric1 ibverbs-utils
```

On Ubuntu 24.04, `librdmacm1` was renamed to `librdmacm1t64` during the 64-bit `time_t` transition. On 64-bit systems apt picks the new package automatically.

## Install InfiniBand diagnostic tools

```shell
sudo apt install infiniband-diags
```

This package provides tools such as `ibstat`, which shows the HCA and port state.

## Install MST tools

You need these for Mellanox NICs. The `mstflint` package provides both `mstflint` and `mstconfig`.

```shell
sudo apt update && sudo apt install mstflint
```

## Find the NIC's PCI address

```shell
lspci | grep Mel
```

The first column (for example `05:00.0`) is the PCI address that you pass to `mstflint`/`mstconfig` with `-d`.

## Flash firmware

Check the current firmware version and PSID, then burn the new image:

```shell
sudo mstflint -d 86:00.0 query
sudo mstflint -d 86:00.0 -i fw-ConnectX3-rel-2_42_5000-MCX354A-FCB_A2-A5-FlexBoot-3.4.752.bin -allow_psid_change burn
```

- `86:00.0` is the NIC's PCI address.
- `-allow_psid_change` lets you burn an image whose PSID (Parameter Set ID) differs from the one on the card. This is how you put stock firmware on an OEM-branded card. Use it with care, because a wrong PSID can make the card malfunction.

Firmware images: <https://network.nvidia.com/support/firmware/firmware-downloads/>

## Set the port link type to InfiniBand

VPI cards can run each port as InfiniBand or Ethernet. To switch both ports to InfiniBand:

```shell
sudo mstconfig -d 05:00.0 query | grep LINK_TYPE
sudo mstconfig -d 05:00.0 set LINK_TYPE_P1=IB LINK_TYPE_P2=IB  # valid values: IB, ETH, VPI
sudo reboot
```

- `LINK_TYPE_P1` and `LINK_TYPE_P2` configure port 1 and port 2. On a single-port card, set only `LINK_TYPE_P1`.
- Valid values are `IB`, `ETH` and `VPI`. `query` shows them as `IB(1)` and `ETH(2)`. Not every adapter supports `VPI`.
- The new setting takes effect after a reboot.

## Load the IPoIB modules

To load the modules at boot, use one of these two methods.

Option 1: append them to `/etc/modules`:

```conf
# /etc/modules
ib_ipoib
ib_umad
```

Option 2: put them in their own file under `/etc/modules-load.d/`:

```shell
sudo tee /etc/modules-load.d/ipoib.conf << EOF
ib_ipoib
ib_umad
EOF
```

To load them right away without rebooting:

```shell
sudo modprobe ib_ipoib
sudo modprobe ib_umad
```

## Run a subnet manager (OpenSM)

Every InfiniBand fabric needs one subnet manager. If your switch doesn't run one (for example, an unmanaged switch or two hosts connected back-to-back), run OpenSM on one of the hosts:

```shell
sudo apt install opensm
sudo systemctl enable --now opensm
```

## Configure an InfiniBand switch

See this post: <https://blog.xjn819.com/post/Upgrade-from-10Gbps-to-40Gbps-network.html>

## Test RDMA bandwidth and latency with qperf

Install `qperf` on both nodes:

```shell
sudo apt install qperf
```

Server side (with no arguments, qperf runs in server mode):

```shell
qperf
```

Client side, running RC (Reliable Connection) RDMA bandwidth and latency tests:

```shell
qperf <server-ip> rc_bw rc_lat
```

qperf sets up the test over a TCP control connection, so `<server-ip>` must be reachable over IP (IPoIB or Ethernet).

## Test IPoIB TCP throughput with iperf

`iperf` measures plain TCP throughput over the IPoIB interface. It doesn't test RDMA.

```shell
sudo apt install iperf
```

Server side:

```shell
iperf -s -i 1 -f m
```

Client side, using the server's IPoIB address (for example on `ib0`):

```shell
iperf -c <server-ip> -i 1 -t 30 -f m
```

`-i 1` prints a report every second, `-f m` reports in Mbit/s, and `-t 30` runs the test for 30 seconds.
