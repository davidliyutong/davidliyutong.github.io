# Remove NS

terminal1:

```shell
namespacename=...
cat > tmp.json <<EOF
{
  "apiVersion": "v1",
  "kind": "Namespace",
  "metadata": {
    "name": "$namespacename"
  },
  "spec": {
    "finalizers": []
  }
}
EOF
```

terminal2:

```shell
curl -k -H "Content-Type: application/json" -X PUT --data-binary @tmp.json http://127.0.0.1:8001/api/v1/namespaces/$namespacename/finalize
```
