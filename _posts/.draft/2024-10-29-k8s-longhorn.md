## Pre-flight

```shell
curl -sSL https://raw.githubusercontent.com/longhorn/longhorn/refs/tags/v1.7.2/scripts/environment_check.sh | bash
```

```shell
dnf install iscsi-initiator-utils jq nfs-utils
```

```
echo iscsi_tcp | sudo tee -a /etc/modules
```

## Helm Install

https://longhorn.io/docs/

```shell
helm repo add longhorn https://charts.longhorn.io --force-update
```

```
helm install longhorn longhorn/longhorn --namespace longhorn-system --create-namespace --version 1.7.2
```

### Custom Installation

```shell
curl -Lo values.yaml https://raw.githubusercontent.com/longhorn/charts/master/charts/longhorn/values.yaml
```

```shell
helm install longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --create-namespace \
  --values values.yaml
```

Update settings

```shell
helm upgrade longhorn longhorn/longhorn --namespace longhorn-system --values ./values.yaml --version `helm list -n longhorn-system -o json | jq -r .'[0].app_version'`
```

## Storage Class
