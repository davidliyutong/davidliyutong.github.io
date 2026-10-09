---
layout: post
title: RKE2 Snippets for Clusters Without Docker Hub Access
date: 2024-09-28 10:01:00
description: Point RKE2 at an image mirror and copy individual images into a private registry when nodes cannot reach Docker Hub.
tags: kubernetes rke2 containerd GFW
categories: IT
---

RKE2 pulls its system images, starting with the `pause` image, from Docker Hub. When the nodes cannot reach Docker Hub, for example behind the GFW, the node never becomes ready. These snippets handle that case. For the full node setup, see [Setting Up RKE2 on Rocky Linux]({% post_url 2024-10-29-k8s-rke2-setup %}).

## Cannot pull the `pause` image

Edit `/etc/rancher/rke2/config.yaml` and set `system-default-registry`:

```yaml
# /etc/rancher/rke2/config.yaml
system-default-registry: "mirror.example.com"
```

`mirror.example.com` is your image mirror, for example a Harbor instance or a Docker Hub proxy that you run yourself. RKE2 then pulls every system image, including `pause`, from that registry. Restart `rke2-server` (or `rke2-agent`) after the change.

> Also check `/etc/resolv.conf` and make sure its nameservers work. A broken resolver looks like a registry problem.

## Copy images into a private registry

Some charts reference images that the mirror does not have. Copy them by hand in two steps.

On a machine that can reach Docker Hub, pull each image, retag it for your registry and push it:

```bash
#!/bin/bash

# Images to copy, and the registry to copy them to
IMAGES=("rancher/mirrored-grafana-grafana:9.1.5" "rancher/mirrored-kiwigrid-k8s-sidecar:1.19.2" "rancher/mirrored-library-nginx:1.24.0-alpine" "rancher/mirrored-library-busybox:1.31.1")
MIRROR_REGISTRY="registry.example.com"

for IMAGE in "${IMAGES[@]}"; do
    echo "Pulling ${IMAGE}..."
    docker pull "${IMAGE}"

    echo "Retagging ${IMAGE}..."
    docker tag "${IMAGE}" "${MIRROR_REGISTRY}/${IMAGE}"

    echo "Pushing ${MIRROR_REGISTRY}/${IMAGE}..."
    docker push "${MIRROR_REGISTRY}/${IMAGE}"
done
```

On each RKE2 node, pull the images from your registry with `ctr` and tag them with their original `docker.io` names, so that pods referencing the original names start without pulling:

```bash
#!/bin/bash

IMAGES=("rancher/mirrored-grafana-grafana:9.1.5" "rancher/mirrored-kiwigrid-k8s-sidecar:1.19.2" "rancher/mirrored-library-nginx:1.24.0-alpine" "rancher/mirrored-library-busybox:1.31.1")
MIRROR_REGISTRY="registry.example.com"
CTR="ctr -n k8s.io --address /run/k3s/containerd/containerd.sock"

for IMAGE in "${IMAGES[@]}"; do
    echo "Pulling ${MIRROR_REGISTRY}/${IMAGE}..."
    $CTR image pull "${MIRROR_REGISTRY}/${IMAGE}"

    echo "Retagging ${IMAGE}..."
    $CTR image tag "${MIRROR_REGISTRY}/${IMAGE}" "docker.io/${IMAGE}"
done
```

RKE2 ships `ctr` in `/var/lib/rancher/rke2/bin`. Add that directory to `PATH` if the command is not found.
