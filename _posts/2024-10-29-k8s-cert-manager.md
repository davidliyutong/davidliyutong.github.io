---
layout: post
title: cert-manager with Cloudflare DNS-01 on Kubernetes
date: 2024-10-29 00:02:00
description: Install cert-manager with Helm and issue Let's Encrypt certificates through Cloudflare DNS validation.
tags: kubernetes helm cert-manager cloudflare tls IT
categories: IT
---

[cert-manager](https://cert-manager.io/docs/) issues and renews TLS certificates for Kubernetes workloads. With the DNS-01 challenge and the Cloudflare API, it can get Let's Encrypt certificates for any domain hosted on Cloudflare, including wildcard certificates and hosts that are not reachable from the internet.

## Install with Helm

```shell
helm repo add jetstack https://charts.jetstack.io --force-update

helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.16.1 \
  --set crds.enabled=true
```

## Cloudflare DNS issuer

First create a Cloudflare API token under **My Profile → API Tokens** with these permissions:

- Zone → DNS → Edit
- Zone → Zone → Read

Store the token in a Secret. A ClusterIssuer reads its secrets from the namespace cert-manager runs in, so the Secret must be in `cert-manager`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-token-secret
  namespace: cert-manager
type: Opaque
stringData:
  api-token: <your_secret>
```

Then create the ClusterIssuer:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-cf-dns
spec:
  acme:
    privateKeySecretRef:
      name: letsencrypt-cf-dns
    server: https://acme-v02.api.letsencrypt.org/directory
    solvers:
      - dns01:
          cloudflare:
            apiTokenSecretRef:
              key: api-token
              name: cloudflare-api-token-secret
```

To test without hitting Let's Encrypt's rate limits, use the staging server `https://acme-staging-v02.api.letsencrypt.org/directory` first.

## Use it from an Ingress

Add the `cert-manager.io/cluster-issuer` annotation and a `tls` section. cert-manager then creates a Certificate and stores the result in the named Secret:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-cf-dns
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - app.example.com
      secretName: app-example-com-tls
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: example
                port:
                  number: 80
```

Check progress with `kubectl describe certificate -n <namespace>` and `kubectl get challenges -A`.
