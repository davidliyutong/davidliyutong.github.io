Official Document at https://docs.goauthentik.io/docs/install-config/install/kubernetes

## Helm installation

Generate strong SecretKey:

```shell
pwgen -s 50 1
openssl rand 60 | base64 -w 0
```

Create values.yaml

```yaml
authentik:
  secret_key: ""
  # This sends anonymous usage-data, stack traces on errors and
  # performance data to sentry.io, and is fully opt-in
  error_reporting:
    enabled: true
  postgresql:
    password: "ThisIsNotASecurePassword"

server:
  ingress:
    # Specify kubernetes ingress controller class name
    ingressClassName: nginx | traefik | kong
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

Install

```shell
helm repo add authentik https://charts.goauthentik.io --force-update
helm upgrade --namespace authentik --create-namespace --install authentik authentik/authentik -f values.yaml
```

## Upgrade Guide

https://docs.goauthentik.io/docs/troubleshooting/postgres/upgrade_kubernetes

cannot across version

breaking changes when upgrade to 2023.8.0, postgres changed to 15 from 11
