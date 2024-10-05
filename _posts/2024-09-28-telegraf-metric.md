---
layout: post
title: Establishing a telegraf metric system
date: 2024-09-28 00:01:00
description:
tags: telemetry maintenance IT
categories: IT
---

Telegraf is an agent for collecting, processing, aggregating, and writing metrics. With over 200 plugins, it provides all the metrics you need to track and understand your systems and applications effectively. This blog will demonstrate how to set up a telegraf metric stack.

To persist the data, we will store the telemetry data using **InfluxDB.** This efficient time-series database is well suited to store the data for subsequent analysis (e.g. visualization via Grafana).

## Deploy A InfluxDB Service

There are two ways to deploy an InfluxDB database:

1. Docker Compose
2. Kubernetes

```yaml
services:
  influxdb:
    image: influxdb:2.2 # image name
    container_name: influxdb # container name
    ports:
      - "8086:8086"
    volumes:
      - influxdb_conf:/etc/influxdb # conf volume
      - influxdb_data:/var/lib/influxdb # data volume
volumes:
  influxdb_data:
  influxdb_conf:
```

Below is the reference configuration file for deployment using K8S. Please note that NFS data volumes are used here and the service is exposed through Nginx Ingress, so the actual deployment needs to be modified according to the specific cluster configuration (the places that need to be modified have been labeled). Deployment type resources are used here, actually using StatefulSet would be better.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    workload.user.cattle.io/workloadselector: apps.deployment-<namespace>-influxdb # TO BE MODIFED
  name: influxdb
  namespace: <namespace> # TO BE MODIFED
spec:
  replicas: 1
  selector:
    matchLabels:
      workload.user.cattle.io/workloadselector: apps.deployment-<namespace>-influxdb # TO BE MODIFED
  template:
    metadata:
      creationTimestamp: null
      labels:
        workload.user.cattle.io/workloadselector: apps.deployment-<namespace>-influxdb # TO BE MODIFED
    spec:
      containers:
        - image: influxdb:2.2
          imagePullPolicy: IfNotPresent
          name: container-0
          ports:
            - containerPort: 8086
              name: 8086tcp
              protocol: TCP
          resources:
            limits:
              cpu: "2"
              memory: 4Gi
            requests:
              cpu: 50m
              memory: 128Mi
          volumeMounts:
            - mountPath: /var/lib/influxdb2
              name: <volume_name> # TO BE MODIFED
              subPath: db-data
            - mountPath: /etc/influxdb
              name: <volume_name> # TO BE MODIFED
              subPath: db-config
      restartPolicy: Always
      volumes:
        - name: <volume_name> # TO BE MODIFED
          nfs:
            path: /<volume_path> # TO BE MODIFED
            server: 0.0.0.0 # TO BE MODIFED
---
apiVersion: v1
kind: Service
metadata:
  name: influxdb
  namespace: <namespace> # TO BE MODIFED
spec:
  ports:
    - name: 8086tcp
      port: 8086
      protocol: TCP
      targetPort: 8086
  selector:
    workload.user.cattle.io/workloadselector: apps.deployment-<namespace>-influxdb # TO BE MODIFED
  type: ClusterIP
---
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx # TO BE MODIFED
  name: influxdb-ingress
  namespace: <namespace> # TO BE MODIFED
spec:
  rules:
    - host: influxdb.example.com # TO BE MODIFED
      http:
        paths:
          - backend:
              service:
                name: influxdb
                port:
                  number: 8086
            path: /
            pathType: Prefix
  tls:
    - hosts:
        - influxdb.example.top # TO BE MODIFED
      secretName: <tls_secret> # TO BE MODIFED
```

## Deploy A Telegraf Collector

These are steps to deploy telegraf collector

1. Install telegraf package according to this [guide](https://github.com/influxdata/telegraf/blob/master/docs/INSTALL_GUIDE.md).
2. Generate access token from influxdb's webui.
3. Configure telegraf's configuration `/etc/telegraf.conf` according to the access token.
4. Test the configuration.

You can use this command to generate the default configuration as a reference:

```shell
telegraf config > telegraf.conf
```

Here is a selection of useful telegraf configurations:

```conf
[global_tags]
[agent]
  interval = "60s"
  round_interval = true
  metric_batch_size = 1000
  metric_buffer_limit = 10000
  collection_jitter = "0s"
  flush_interval = "10s"
  flush_jitter = "0s"
  precision = "0s"
  hostname = ""
  omit_hostname = false
[[outputs.influxdb_v2]]
    urls = ["https://influxdb.example.com"]
    token = "xxx"
    organization = "<organization_name>"
    bucket = "telegraf"
[[inputs.cpu]]
  percpu = true
  totalcpu = true
  collect_cpu_time = false
  report_active = false
[[inputs.disk]]
  ignore_fs = ["tmpfs", "devtmpfs", "devfs", "iso9660", "overlay", "aufs", "squashfs", "loop"]
[[inputs.diskio]]
[[inputs.kernel]]
[[inputs.mem]]
[[inputs.processes]]
[[inputs.swap]]
[[inputs.system]]
[[inputs.bond]]
[[inputs.net]]
[[inputs.nvidia_smi]]
    timeout = "25s"
[[inputs.smart]]
    interval = "600s"
    attributes = true
[[inputs.temp]]
```

### [agent]

This configures how agent behaves, `hostname` defaults to system's hostname.

```conf
[agent]
  interval = "60s"
  round_interval = true
  metric_batch_size = 1000
  metric_buffer_limit = 10000
  collection_jitter = "0s"
  flush_interval = "10s"
  flush_jitter = "0s"
  precision = "0s"
  hostname = ""
  omit_hostname = false
```

### [outputs.influxdb_vs]

This section instructs telegraf to send metric to InfluxDB database, `url`, `token`, `organization` and `bucket` need to be properly set.

```conf
[[outputs.influxdb_v2]]
    urls = ["https://influxdb.example.com"]
    token = "xxx"
    organization = "<organization_name>"
    bucket = "telegraf"
```

### [[inputs.cpu]]

This enables the collection of CPU usage.

```conf
[[inputs.cpu]]
  percpu = true
  totalcpu = true
  collect_cpu_time = false
  report_active = false
```

### [[inputs.disk]]

This enables the collection of disk and filesystem, except some types.

```conf
[[inputs.disk]]
  ignore_fs = ["tmpfs", "devtmpfs", "devfs", "iso9660", "overlay", "aufs", "squashfs", "loop"]
```

### [[inputs.diskio]]

This enables the collection of disk io.

```conf
[[inputs.diskio]]
```

### [[inputs.kernel]]

This enables the collection of kernel info.

```conf
[[inputs.kernel]]
```

### [[inputs.mem]]

This enables the collection of memory usage.

```conf
[[inputs.mem]]
```

### [[inputs.processes]]

This enables the collection of processes.

```conf
[[inputs.processes]]
```

### [[inputs.swap]]

This enables the collection of swap usage.

```conf
[[inputs.swap]]
```

### [[inputs.system]]

This enables the collection of system.

```conf
[[inputs.system]]
```

### [[inputs.bond]]

This enables the collection of traffic.

```conf
[[inputs.bond]]
```

### [[inputs.net]]

This enables the collection of network.

```conf
[[inputs.net]]
```

### [[inputs.nvidia_smi]]

This enables the collection of NVIDIA GPUs.

```conf
[[inputs.nvidia_smi]]
    timeout = "25s"
```

### [[inputs.smart]]

This enables the collection of SMART information.

```conf
[[inputs.smart]]
    interval = "600s"
    attributes = true
```

### [[inputs.temp]]

This enables the collection of thermo.

```conf
[[inputs.temp]]
```

After you have generated the configuration, run `telegraf` command to test it. If everything works fine, enable the telegraf system service and let it run in the background.
