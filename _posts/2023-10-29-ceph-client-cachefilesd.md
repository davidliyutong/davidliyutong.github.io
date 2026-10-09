---
layout: post
title: Mounting CephFS on a Client with cachefilesd Caching
date: 2023-10-29 00:01:00
description: Install the Ceph client, mount CephFS via /etc/fstab, and cache frequently used files on local NVMe with cachefilesd.
tags: ceph cephfs cachefilesd IT
categories: IT
---

These are my notes for setting up an Ubuntu 20.04 (focal) machine as a client of an existing Ceph Quincy cluster. The steps cover installing the packages, mounting two CephFS file systems from `/etc/fstab`, and enabling cachefilesd so that frequently read files are cached on a local NVMe drive.

## Install Ceph

Add the official Ceph Quincy repository, then install the packages. `apt-key` is deprecated, so the release key goes into its own keyring file and the repository entry references it with `signed-by`:

```bash
sudo install -d -m 0755 /etc/apt/keyrings
wget -q -O- 'https://download.ceph.com/keys/release.asc' | gpg --dearmor | sudo tee /etc/apt/keyrings/ceph.gpg > /dev/null
echo 'deb [signed-by=/etc/apt/keyrings/ceph.gpg] https://download.ceph.com/debian-quincy/ focal main' | sudo tee /etc/apt/sources.list.d/ceph.list # codename = focal
sudo apt-get update
sudo apt-get install ceph=17.2.7-1focal ceph-base lvm2
```

17.2.7 was the current Quincy release when I wrote this. The repository index now carries only the latest Quincy point release, so drop the version pin or pin every Ceph package to the same version.

The `fuse.ceph` mount below requires the `ceph-fuse` package. `ceph` pulls it in through recommended packages, so install it explicitly if you use `--no-install-recommends`.

## Configure the Client

### Ceph Config and Keyring

Copy these two files verbatim from an existing client node of the cluster:

- `/etc/ceph/ceph.conf`
- `/etc/ceph/ceph.client.myclient.keyring`

### /etc/fstab

Add these entries:

```conf
<MON_IP>:6789:/     /mnt/homes    ceph    name=myclient,fs=homes,mon_addr=<MON_IP>:6789,secret=<CEPH_CLIENT_SECRET>,noatime,_netdev    0       2
none    /mnt/public     fuse.ceph       ceph.name=client.myclient,ceph.client_fs=public,_netdev
```

The `homes` file system is mounted with the kernel driver instead of `ceph-fuse` so that it can use cachefilesd, which is set up in the next section. The `public` file system uses `ceph-fuse`.

`/etc/fstab` is world-readable, so keeping the key in it with `secret=` exposes it to every local user. The keyring is already in `/etc/ceph`, so you can omit `secret=` and let `mount.ceph` look up the key for `name=myclient` in the keyring. Alternatively, use `secretfile=` to point at a root-only file that contains just the key.

## Cache CephFS with cachefilesd

cachefilesd caches frequently used files from network file systems on a local disk, in this case an NVMe drive. The cache directory is set by `dir` in `/etc/cachefilesd.conf` (default `/var/cache/fscache`). Make sure that directory is on the NVMe drive.

On Ubuntu 20.04, the package ships only a SysV init script. That script exits without starting the daemon unless `RUN=yes` is set in `/etc/default/cachefilesd`, so enable that line and then restart the service:

```bash
sudo apt-get install cachefilesd
sudo sed -i 's/^#RUN=yes/RUN=yes/' /etc/default/cachefilesd
sudo systemctl enable cachefilesd
sudo systemctl restart cachefilesd
```

Next, add `fsc` to the mount options of the `/mnt/homes` entry in `/etc/fstab`:

```conf
<MON_IP>:6789:/     /mnt/homes    ceph    name=myclient,fs=homes,mon_addr=<MON_IP>:6789,secret=<CEPH_CLIENT_SECRET>,noatime,fsc,_netdev    0       2
```

## Verify

`mount -a` skips file systems that are already mounted, so unmount `/mnt/homes` first. Then remount it and check the FS-Cache statistics:

```bash
sudo umount /mnt/homes
sudo mount -a
cat /proc/fs/fscache/stats
```
