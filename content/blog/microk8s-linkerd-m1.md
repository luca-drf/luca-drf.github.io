---
title: "Microk8s Linkerd M1"
date: 2022-04-19T16:50:23+01:00
draft: true
tags: []
featured: false
---


https://microk8s.io/docs/addon-dashboard

```
microk8s status --wait-ready
An error occurred when trying to execute 'sudo microk8s.status' with 'multipass': returned exit code 1.
```

```
An error occurred when trying to execute 'sudo ls -1 /snap/bin/' with 'multipass': returned exit code 2
```

```bash
multipass exec microk8s-vm -- sudo /snap/bin/microk8s kubectl port-forward -n kube-system service/kubernetes-dashboard 10443:443 --address 0.0.0.0
```

```bash
 multipass exec microk8s-vm -- sudo /snap/bin/microk8s kubectl -n kube-system describe secret $(multipass exec microk8s-vm -- sudo /snap/bin/microk8s kubectl -n kube-system get secret | grep default-token | cut -d " " -f1)
```

```bash
multipass info microk8s-vm | grep IPv4 | awk '{ print $2 }'
```


```bash
microk8s linkerd check
```


```
...
× is running the minimum kubectl version
    exec: "kubectl": executable file not found in $PATH
    see https://linkerd.io/2.10/checks/#kubectl-version for hints
...
```

```
sudo snap alias microk8s.kubectl kubectl
```

```
...
× viz extension self-check
    Error calling Prometheus from the control plane: Post "http://prometheus.linkerd-viz.svc.cluster.local:9090/api/v1/query": dial tcp 127.0.0.1:9090: connect: connection refused
    see https://linkerd.io/2.10/checks/#l5d-viz-metrics-api for hints
...
```