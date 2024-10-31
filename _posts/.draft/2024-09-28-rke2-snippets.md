---
layout: post
title: Useful RKE2 snippets for deploying in corner cases.
date: 2024-09-28 10:01:00
description:
tags: kubernetes GFW
categories:
---

## Cannot pull the `pause` image

Edit the `/etc/rancher/rke/config.yaml` configuration and add this section

```yaml
# /etc/rancher/rke/config.yaml
system-default-registry: "mirror.example.com"
```

where `mirror.example.com` is your image mirror. You can build one of your own using Cloudflare Workers. For details, refer to [this post]().

> Remember to check `/etc/resolv.conf` and set nameserver to functional ones.

```bash
#!/bin/bash

# Define variables
IMAGES=("rancher/mirrored-grafana-grafana:9.1.5" "rancher/mirrored-kiwigrid-k8s-sidecar:1.19.2" "rancher/mirrored-library-nginx:1.24.0-alpine" "rancher/mirrored-library-busybox:1.31.1")  # Replace with your image names
MIRROR_REGISTRY="core.harbor.speit.site"  # Replace with your registry

# Loop over images
for IMAGE in "${IMAGES[@]}"
do
    # Pull the image
    echo "Pulling ${IMAGE}..."
    docker pull "${IMAGE}"

    # Retag the image
    echo "Retagging ${IMAGE}..."
    docker tag "${IMAGE}" "${MIRROR_REGISTRY}/${IMAGE}"

    # Push the image
    echo "Pushing ${MIRROR_REGISTRY}/${IMAGE}..."
    docker push "${MIRROR_REGISTRY}/${IMAGE}"
done

# Loop over images
for IMAGE in "${IMAGES[@]}"
do
    # Pull the image
    echo "Pulling ${IMAGE}..."
    ctr -n k8s.io --address /run/k3s/containerd/containerd.sock image pull "${MIRROR_REGISTRY}/${IMAGE}"

    # Retag the image
    echo "Retagging ${IMAGE}..."
    ctr -n k8s.io --address /run/k3s/containerd/containerd.sock image tag "${MIRROR_REGISTRY}/${IMAGE}" "docker.io/${IMAGE}"
done

```
