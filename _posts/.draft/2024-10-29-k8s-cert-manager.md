https://cert-manager.io/docs/

## Helm way

helm repo add jetstack https://charts.jetstack.io --force-update

helm install \
 cert-manager jetstack/cert-manager \
 --namespace cert-manager \
 --create-namespace \
 --version v1.16.1 \
 --set crds.enabled=true

## Cloudflare DNS Issurer

Issuer

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

Secrete

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
