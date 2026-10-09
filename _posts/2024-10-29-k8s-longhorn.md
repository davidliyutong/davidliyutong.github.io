---
layout: post
title: Installing Longhorn Storage on Kubernetes
date: 2024-10-29 00:04:00
description: Prepare nodes for Longhorn, install it with Helm, change its settings, and define a custom StorageClass.
tags: kubernetes helm longhorn storage IT
categories: IT
---

[Longhorn](https://longhorn.io/docs/) is a distributed block storage system for Kubernetes. It replicates volumes across nodes using local disks. These notes cover installation with Helm on RHEL-family nodes such as Rocky Linux.

## Pre-flight

Install the packages Longhorn needs on every node. iSCSI is used to attach volumes, and NFS is needed for ReadWriteMany volumes and backups:

```shell
dnf install iscsi-initiator-utils jq nfs-utils
systemctl enable --now iscsid
```

Load the `iscsi_tcp` kernel module now and at every boot:

```shell
modprobe iscsi_tcp
echo iscsi_tcp | sudo tee /etc/modules-load.d/iscsi_tcp.conf
```

Then run Longhorn's environment check script, which reports any missing dependency:

```shell
curl -sSL https://raw.githubusercontent.com/longhorn/longhorn/refs/tags/v1.7.2/scripts/environment_check.sh | bash
```

## Helm install

```shell
helm repo add longhorn https://charts.longhorn.io --force-update
helm install longhorn longhorn/longhorn --namespace longhorn-system --create-namespace --version 1.7.2
```

### Custom installation

To change the defaults, download the chart's `values.yaml`, edit it, and install with it:

```shell
curl -Lo values.yaml https://raw.githubusercontent.com/longhorn/charts/master/charts/longhorn/values.yaml
```

```shell
helm install longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --create-namespace \
  --values values.yaml
```

To apply changes to `values.yaml` later without changing the Longhorn version, upgrade to the version that is already installed:

```shell
helm upgrade longhorn longhorn/longhorn --namespace longhorn-system --values ./values.yaml --version `helm list -n longhorn-system -o json | jq -r .'[0].app_version'`
```

## Storage class

The chart creates a `longhorn` StorageClass and makes it the default. To use other settings, such as fewer replicas on a small cluster, create another StorageClass:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-single
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: Delete
volumeBindingMode: Immediate
parameters:
  numberOfReplicas: "1"
  staleReplicaTimeout: "2880"
  fsType: ext4
  dataLocality: best-effort
```

Reference it from a PersistentVolumeClaim with `storageClassName: longhorn-single`.
