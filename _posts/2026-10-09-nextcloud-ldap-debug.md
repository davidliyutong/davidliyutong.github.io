---
layout: post
title: Debugging Nextcloud LDAP with occ
date: 2026-10-09 00:01:00
description: Inspect, change, and test Nextcloud's LDAP configuration from the command line in Docker and Kubernetes.
tags: nextcloud ldap docker kubernetes IT
categories: IT
---

When Nextcloud's LDAP backend stops working, for example after the LDAP server moves to a new host or port, the admin UI is a slow place to debug it. The `occ` command-line tool lets you dump the LDAP configuration, change single keys, and test the connection in seconds. These are my notes for doing that in Docker and in Kubernetes, including a memory-limit pitfall I hit in the Kubernetes case.

## The three commands

LDAP debugging with `occ` comes down to three subcommands:

- `ldap:show-config [configID]` prints all LDAP configurations, or only the given one.
- `ldap:set-config <configID> <key> <value>` changes a single setting, such as `ldapHost` or `ldapPort`.
- `ldap:test-config <configID>` validates the configuration and tries to connect to the LDAP server.

The first LDAP configuration gets the ID `s01`; `ldap:show-config` lists the IDs of any others. `occ` must run as the web server user, which is `www-data` in the official Nextcloud image.

## Docker

`docker exec --user www-data` runs `occ` as the right user and keeps the container's environment. Replace `nextcloud-app` with your container name:

```bash
docker exec --user www-data -it nextcloud-app php occ ldap:show-config
docker exec --user www-data -it nextcloud-app php occ ldap:set-config "s01" "ldapPort" "389"
docker exec --user www-data -it nextcloud-app php occ ldap:test-config s01
```

The usual loop is to change `ldapHost` or `ldapPort`, run `ldap:test-config`, and repeat until it reports "The configuration is valid and the connection could be established!". The screenshot below starts with the tail of the `ldap:show-config` output (user filter, object classes, UUID attributes). After that come two rounds of the loop: first `s01` points at an LDAP server by hostname (`ldap-backend` on port 389), then at a server on another host listening on port 30389. Both pass the test.

{% include figure.liquid loading="eager" path="assets/img/posts/nextcloud-ldap-debug/occ-ldap-show-test-config.png" class="img-fluid rounded z-depth-1" zoomable=true %}

<div class="caption">Tail of <code>ldap:show-config</code>, then changing <code>ldapHost</code>/<code>ldapPort</code> and checking each change with <code>ldap:test-config s01</code>.</div>

## Kubernetes

`kubectl exec` has no `--user` flag, so you enter the pod as the container's user (root in the official image) and must switch to `www-data` yourself. Running `occ` as root fails: it refuses and suggests `sudo -u #33` (UID 33 is `www-data`). If your pod already runs as UID 33, you can call `php occ` directly.

### Switch users with su

The simplest way is `su`, which is already in the image. `www-data` has `/usr/sbin/nologin` as its login shell, so pass a shell with `-s`:

```bash
kubectl exec -it <nextcloud-pod> -- su -s /bin/bash www-data
# then, inside that shell:
php occ ldap:show-config
php occ ldap:test-config s01
```

Or run a single command, the way the Nextcloud Helm chart documents it (`/bin/sh` also works on Alpine-based images, which have no bash):

```bash
kubectl exec <nextcloud-pod> -- su -s /bin/sh www-data -c "php occ ldap:test-config s01"
```

Plain `su` (without `-` or `-l`) keeps the current environment. That matters, for the reason below.

### Pitfall: sudo drops PHP_MEMORY_LIMIT

My first attempt used `sudo` instead, which meant installing it first because the image doesn't include it. `occ` then failed with an error that has nothing to do with users:

{% include figure.liquid loading="eager" path="assets/img/posts/nextcloud-ldap-debug/occ-k8s-php-memory-limit.png" class="img-fluid rounded z-depth-1" zoomable=true %}

<div class="caption">Inside the pod: <code>occ</code> refuses to run as root, dies at a 2 MB memory limit under <code>sudo -u www-data</code>, and works after <code>export PHP_MEMORY_LIMIT=512M</code>.</div>

The cause is in the image itself. Its Dockerfile sets `ENV PHP_MEMORY_LIMIT 512M` and writes `memory_limit=${PHP_MEMORY_LIMIT}` into PHP's `conf.d/nextcloud.ini`, so PHP reads its memory limit from an environment variable. `sudo` resets the environment by default, so the variable disappears and `memory_limit` comes out empty. PHP then falls back to its 2 MB minimum, which is the 2097152 bytes in the error.

If you use `sudo` anyway (install it as root with `apt-get update && apt-get install -y sudo`), set the variable again:

```bash
sudo -u www-data bash
export PHP_MEMORY_LIMIT=512M
php occ ...
```

Or preserve the environment with `-E`, as the Nextcloud admin manual does:

```bash
sudo -E -u www-data php occ ldap:test-config s01
```

Still, `su` is the better choice. It needs no extra package, and anything installed into a running pod is lost on the next restart anyway.
