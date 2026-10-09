---
layout: post
title: Deploying Authentik on Kubernetes with Helm
date: 2024-10-29 00:01:00
description: Install the Authentik identity provider on Kubernetes with its Helm chart, and what to watch for when upgrading.
tags: kubernetes helm authentik sso IT
categories: IT
---

[Authentik](https://goauthentik.io/) is an open-source identity provider that supports SSO through OAuth2/OpenID Connect, SAML and LDAP. These are my notes for installing it with the official Helm chart. The official guide is at [Kubernetes installation](https://docs.goauthentik.io/docs/install-config/install/kubernetes).

## Helm installation

Generate a strong secret key with either command:

```shell
pwgen -s 50 1
openssl rand 60 | base64 -w 0
```

Create `values.yaml`:

```yaml
authentik:
  secret_key: "<your-secret-key>"
  # This sends anonymous usage-data, stack traces on errors and
  # performance data to sentry.io, and is fully opt-in
  error_reporting:
    enabled: true
  postgresql:
    password: "ThisIsNotASecurePassword"

server:
  ingress:
    # Kubernetes ingress controller class name, e.g. nginx, traefik or kong
    ingressClassName: nginx
    enabled: true
    hosts:
      - authentik.domain.tld

postgresql:
  enabled: true
  auth:
    password: "ThisIsNotASecurePassword"
redis:
  enabled: true
```

`authentik.postgresql.password` and `postgresql.auth.password` must be the same value. Replace both with a real password.

Install the chart:

```shell
helm repo add authentik https://charts.goauthentik.io --force-update
helm upgrade --namespace authentik --create-namespace --install authentik authentik/authentik -f values.yaml
```

Then open `https://authentik.domain.tld/if/flow/initial-setup/` to set the password of the `akadmin` user.

## Upgrade guide

- Read the release notes before upgrading. Authentik does not support skipping major versions, so upgrade one version at a time.
- The chart's bundled PostgreSQL changed its major version over time. For example, upgrading across 2023.8.0 moves PostgreSQL from 11 to 15, and the existing data directory does not work with the new version. You have to dump the database, upgrade, and restore it. Follow the official guide: [Upgrade PostgreSQL on Kubernetes](https://docs.goauthentik.io/docs/troubleshooting/postgres/upgrade_kubernetes).
