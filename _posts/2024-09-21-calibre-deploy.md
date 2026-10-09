---
layout: post
title: Deploying Calibre-Web with Docker Compose and Kubernetes
date: 2024-09-21 00:01:00
description: Run Calibre-Web on Docker Compose or Kubernetes, and fix the two errors you are most likely to hit on a fresh instance.
tags: calibre kubernetes docker web IT
categories: IT
---

[Calibre](https://calibre-ebook.com/) is a well-known e-book manager. The [Calibre-Web](https://github.com/janeczku/calibre-web) project provides a web UI for browsing, reading and uploading books stored in a Calibre database (`metadata.db`). These are my notes for running it with Docker Compose and on Kubernetes, plus the two problems I ran into.

Both setups below use the [linuxserver/calibre-web](https://docs.linuxserver.io/images/docker-calibre-web/) image, which listens on port `8083` and uses two volumes:

- `/config` for Calibre-Web's own settings and user database.
- `/books` for the Calibre library, the directory that contains `metadata.db`.

The default login is `admin` / `admin123`. Change it right after the first login.

## Deploy with Docker Compose

```yaml
services:
  calibre-web:
    image: lscr.io/linuxserver/calibre-web:latest
    container_name: calibre-web
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/UTC
    volumes:
      - ./config:/config
      - ./books:/books
    ports:
      - 8083:8083
    restart: unless-stopped
```

`PUID` and `PGID` set the user the app runs as. They must be able to write to `./config` and `./books`.

## Deploy with Kubernetes

The manifest below creates a namespace, two PersistentVolumeClaims, a Deployment and a Service. Adjust the storage sizes and add an Ingress for your cluster.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: calibre-web
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: calibre-web-config
  namespace: calibre-web
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: calibre-web-books
  namespace: calibre-web
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 50Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: calibre-web
  namespace: calibre-web
spec:
  replicas: 1
  strategy:
    type: Recreate # the volumes are ReadWriteOnce
  selector:
    matchLabels:
      app: calibre-web
  template:
    metadata:
      labels:
        app: calibre-web
    spec:
      containers:
        - name: calibre-web
          image: lscr.io/linuxserver/calibre-web:latest
          env:
            - name: PUID
              value: "1000"
            - name: PGID
              value: "1000"
            - name: TZ
              value: Etc/UTC
          ports:
            - containerPort: 8083
          volumeMounts:
            - name: config
              mountPath: /config
            - name: books
              mountPath: /books
      volumes:
        - name: config
          persistentVolumeClaim:
            claimName: calibre-web-config
        - name: books
          persistentVolumeClaim:
            claimName: calibre-web-books
---
apiVersion: v1
kind: Service
metadata:
  name: calibre-web
  namespace: calibre-web
spec:
  selector:
    app: calibre-web
  ports:
    - port: 8083
      targetPort: 8083
```

## Troubleshooting

### "DB location is not valid" on a new instance

On first start Calibre-Web asks for the location of the Calibre database, and it refuses a directory that has no `metadata.db`. Copy an empty `metadata.db` (for example from a fresh library created by the Calibre desktop app) into the books volume, then enter `/books` as the database location. The Calibre-Web process must have write permission on both the directory and the file.

### "File size may be too big" when uploading

There are two likely causes:

- Calibre-Web is deployed behind a reverse proxy that limits the request body size. For ingress-nginx, raise it with the annotation `nginx.ingress.kubernetes.io/proxy-body-size: "0"` (no limit) or a fixed size such as `"200m"`.
- Your client goes through a proxy application such as Surge or Clash, and the upstream V2Ray/Shadowsocks server rejects large uploads. Exclude the Calibre-Web domain from the proxy rules and try again.
