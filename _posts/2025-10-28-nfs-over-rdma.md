---
layout: post
title: NFS over RDMA Configuration Guide
date: 2025-10-28 00:01:00
description: Set up an NFS over RDMA server and client on Ubuntu 22.04/24.04 using nfs.conf and an RDMA mount.
tags: nfs rdma infiniband IT
categories: IT
---

This guide sets up NFS over RDMA on an InfiniBand fabric: load the kernel transport, turn on nfsd's RDMA listener, export a filesystem, and mount it from a client over RDMA. It targets **Ubuntu 22.04 LTS and 24.04 LTS**. Since 22.04, all NFS services read `/etc/nfs.conf` and `/etc/nfs.conf.d/*.conf`. On 20.04 and earlier, the equivalent settings live in `/etc/default/nfs-kernel-server`.

In the examples, `10.0.0.0/24` is the IPoIB subnet, `10.0.0.2` is the server, and `/mnt/fs0` is the exported directory.

## Before you start

- Run `ibv_devinfo` to confirm that the RDMA device and its port are up.
- Run `ib_send_bw` (from perftest) to measure raw RDMA bandwidth between the nodes.
- If a host firewall is active, open port 20049 for both TCP and UDP.

## Kernel module (server and client)

The server-side (`svcrdma`) and client-side (`xprtrdma`) transports both live in `rpcrdma.ko`, and those two names are just module aliases for it. The kernel loads the module on demand when nfsd opens an RDMA listener or a client mounts with `proto=rdma`. Loading it at boot is optional, but it makes the dependency explicit:

```bash
echo rpcrdma | sudo tee /etc/modules-load.d/rdma.conf
sudo modprobe rpcrdma
```

## Server

### 1. Configure nfsd

On Ubuntu 22.04+, `nfs-server.service` (also reachable under its alias `nfs-kernel-server.service`) runs a bare `/usr/sbin/rpc.nfsd`, which reads its settings from the `[nfsd]` section of nfs.conf. `RPCNFSDCOUNT` and `RPCNFSDOPTS` in `/etc/default/nfs-kernel-server` are no longer read. Put the settings in a drop-in file instead:

```bash
sudo tee /etc/nfs.conf.d/rdma.conf << EOF
[nfsd]
threads=16
vers4.2=y
rdma=y
rdma-port=20049
EOF
```

- `threads`: start from the server's CPU core count, then adjust for InfiniBand bandwidth and the actual client load. This example uses 16. You can change it at runtime by writing a number to `/proc/fs/nfsd/threads`.
- `rdma=y` makes `rpc.nfsd` open the RDMA listener itself (it writes `rdma 20049` to `/proc/fs/nfsd/portlist`). With `rdma=y` alone, the port comes from the `nfsrdma` service name. Setting `rdma-port=20049` explicitly removes the dependency on `/etc/services`.
- `vers4.2=y` makes sure NFSv4.2 is offered. RDMA does not depend on it: the RDMA transport works with NFSv3 and NFSv4.x alike.

### 2. Export the filesystem

```bash
sudo tee /etc/exports << EOF
/mnt/fs0 10.0.0.0/24(rw,async,crossmnt,insecure,fsid=0,no_subtree_check,no_root_squash,no_all_squash)
EOF
```

- `insecure` is required because the NFS/RDMA client does not use a reserved source port.
- `fsid=0` makes `/mnt/fs0` the NFSv4 pseudo-root, so NFSv4 clients mount it as `10.0.0.2:/`, not `10.0.0.2:/mnt/fs0`. If you'd rather mount by the full server path, drop `fsid=0`.
- `no_root_squash` should be paired with network-level access control. In production, consider adding a security flavor such as `sec=krb5`.

### 3. Restart and check

```bash
sudo systemctl restart nfs-server

cat /proc/fs/nfsd/portlist   # should include "rdma 20049"
cat /proc/fs/nfsd/threads    # should print 16
nfsconf --dump               # effective config, nfs.conf + nfs.conf.d merged
```

You don't need to edit the systemd unit. If you ever do have to customize it, use a drop-in (`sudo systemctl edit nfs-server`, which writes `/etc/systemd/system/nfs-server.service.d/override.conf`) containing only the lines you're adding. Don't edit the packaged unit file directly, because package upgrades overwrite it.

To add the RDMA listener by hand on a running server, for example for a quick test without touching nfs.conf, run the following. The listener is lost the next time nfsd restarts:

```bash
echo 'rdma 20049' | sudo tee /proc/fs/nfsd/portlist
```

## Client

### 1. Mount over RDMA

Add the export to `/etc/fstab`:

```text
10.0.0.2:/ /mnt/rdma_test nfs rw,async,noatime,nodiratime,rsize=1048576,wsize=1048576,vers=4.2,rdma,port=20049,_netdev 0 0
```

`rdma` is shorthand for `proto=rdma`, and `port=20049` matches the server's `rdma-port`. Pinning `vers=4.2` makes a misconfigured export fail loudly instead of silently falling back to NFSv3. Without a version, if the NFSv4 path lookup fails (for example by using the full path with an `fsid=0` export), `mount.nfs` retries with v3.

### 2. Apply

```bash
sudo mkdir -p /mnt/rdma_test
sudo systemctl daemon-reload   # regenerate mount units from the edited fstab
sudo mount -av
```

You don't need to modify `rpcbind.service`. An NFSv4 client connects straight to the given port without asking rpcbind, and the kernel loads `rpcrdma` when the mount needs it (or at boot via `modules-load.d`).

## Verification and troubleshooting

```bash
# Client: mount options should include proto=rdma,port=20049
nfsstat -m
grep rdma /proc/mounts

# Server: confirm the RDMA listener
cat /proc/fs/nfsd/portlist

# RDMA link state
rdma link show

# NFS network-layer statistics
nfsstat -o net
```

The original setup was tested on Linux 5.15 LTS (the Ubuntu 22.04 GA kernel) with Mellanox ConnectX NICs.

## Optional tuning

### SUNRPC TCP slot table (TCP mounts only)

These `sunrpc` parameters configure the **TCP** RPC transport. They have no effect on a `proto=rdma` mount, so they're only worth setting if the same client also mounts NFS over TCP:

```bash
sudo tee -a /etc/modprobe.d/sunrpc.conf << EOF
options sunrpc tcp_slot_table_entries=128
options sunrpc tcp_max_slot_table_entries=128
EOF
sudo sysctl -w sunrpc.tcp_slot_table_entries=128
sudo sysctl -w sunrpc.tcp_max_slot_table_entries=128
cat /proc/sys/sunrpc/tcp_slot_table_entries  # check
```

`tcp_slot_table_entries` is the initial number of RPC slots per TCP connection (kernel default 2). The table then grows dynamically up to `tcp_max_slot_table_entries` (kernel default 65536). Setting both to 128 therefore caps each TCP connection at 128 in-flight RPCs.

### RDMA credits on the server

The RDMA transport's counterpart is `/proc/sys/sunrpc/svc_rdma/max_requests`: the number of credits (concurrent RPC requests) nfsd grants each RDMA connection. The default is 64 on 5.15 and 128 in recent kernels. The value is read when a connection is accepted, so change it before clients mount:

```bash
cat /proc/sys/sunrpc/svc_rdma/max_requests
```
