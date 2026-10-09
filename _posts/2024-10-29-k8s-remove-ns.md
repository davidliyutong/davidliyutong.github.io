---
layout: post
title: Removing a Kubernetes Namespace Stuck in Terminating
date: 2024-10-29 00:05:00
description: Clear the finalizers of a namespace stuck in the Terminating state through the finalize API.
tags: kubernetes IT
categories: IT
---

A namespace can stay in the `Terminating` state forever, typically when a resource inside it has a finalizer whose controller is gone, for example after the operator was uninstalled first. The namespace's own `spec.finalizers` cannot be emptied with `kubectl edit`. It can only be changed through the namespace's `finalize` subresource.

> Clearing the finalizers skips whatever cleanup they guard. First check `kubectl get all,pvc -n <namespace>` and remove what you can normally.

In the first terminal, start an API proxy on `127.0.0.1:8001`:

```shell
kubectl proxy
```

In a second terminal, write the namespace with an empty finalizer list and `PUT` it to the `finalize` endpoint:

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

curl -k -H "Content-Type: application/json" -X PUT --data-binary @tmp.json http://127.0.0.1:8001/api/v1/namespaces/$namespacename/finalize
```

The namespace disappears right after the request succeeds. Stop `kubectl proxy` afterwards.
