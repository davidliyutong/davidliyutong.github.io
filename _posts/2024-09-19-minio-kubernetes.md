---
layout: post
title: Deploying an experimental MinIO single-node cluster with DirectPV
date: 2024-09-19 00:01:00
description: for experiment purpose, only
tags: kubernetes minio
categories: IT
---

[MinIO](https://github.com/minio/minio) is an open-source high performance object storage solution. It is designed to be distributed and can be deployed on Kubernetes clusters with ease. MinIO requires some sort of persistent storage to store its objects. However, with its built-in erasure coding and hashmap, deploying MinIO on redundant volumes is a surplus, and could result in performance drops and disk overhead. To simplify its storage model, MinIO project has proposed [DirectPV](https://github.com/minio/directpv), a CSI driver for Direct Attached Storage. In practice, it is recommended to provision storage for MinIO using DirectPV.

<div class="col-sm mt-3 mt-md-0">
  {% include figure.liquid loading="eager" path="https://github.com/minio/directpv/blob/master/docs/images/architecture.png?raw=true" class="img-fluid rounded z-depth-1" zoomable=true %}
</div>

Here is a simple guide on deploying a single-node MinIO instance with DirectPV on Kubernetes. **TBD**
