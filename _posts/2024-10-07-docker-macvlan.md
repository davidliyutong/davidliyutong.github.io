---
layout: post
title: Docker MacVLAN Configuration Guide
date: 2024-10-05 00:01:00
description:
tags: docker network IT
categories: IT
---

In Docker networking, MacVLAN is a powerful tool that allows containers to connect directly with the host's physical network. It is particularly useful in environments where you require your containers to be a part of the same network as your host, appearing as physical devices connected to the network switch.

```bash
docker network create -d macvlan --subnet=192.168.1.0/24 --gateway=192.168.1.1 -o parent=enp6s18 lan
```

- `enp6s18` is the NIC's name
- `192.168.1.0/24` is definition of subnet
- `192.168.1.1` is the gateway

To use this network, add this section to `docker-compose.yaml`

```yaml
networks:
  lan:
    external: true
```
