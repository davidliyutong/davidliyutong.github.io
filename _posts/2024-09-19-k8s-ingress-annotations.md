---
layout: post
title: Kubernetes Ingress annotations for Harbor
date: 2024-09-19 00:01:00
description: solution for push timeout with large images
tags: kubernetes ingress
categories: IT
---

When pushing large images to [harbor](https://goharbor.io/) image registry, client may run into push operation timeout due to incorrect `proxy-*-timeout` configuration, or 403 forbidden due to insufficient `proxy-body-size` configuration. The solution is to patch ingress resource of harbor. Here is an example:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "0" # Allow large uploads, other legal values are 20000m
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "60000" # allow long-time connection
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60000" # allow long-time connection
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60000" # allow long-time connection
    nginx.ingress.kubernetes.io/proxy_buffering: "off" # Optional
    nginx.ingress.kubernetes.io/proxy_ignore_headers: X-Accel-Buffering # Optional
    nginx.ingress.kubernetes.io/ssl-redirect: "true" # Optional
```

These annotations also applies to other applications (e.g. Nextcloud and MinIO) with large uploads/downloads.
