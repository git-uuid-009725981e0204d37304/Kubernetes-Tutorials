---
layout: post
title: "Kubernetes Tutorials (13)"
description: "Traefik"
tags: [Kubernetes, Docker]
---

# Kubernetes Tutorials (13)

## Deploy Traefik Ingress Controller

Create `traefik.yaml`.

traefik.yaml

```yaml
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
rules:
  - apiGroups:
      - ""
    resources:
      - pods
      - services
      - endpoints
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-controller
subjects:
- kind: ServiceAccount
  name: traefik-ingress-controller
  namespace: kube-system
---
apiVersion: v1
kind: DaemonSet
apiVersion: extensions/v1beta1
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
  labels:
    k8s-app: traefik-ingress-lb
spec:
  selector:
    matchLabels:
      k8s-app: traefik-ingress-lb
  template:
    metadata:
      labels:
        k8s-app: traefik-ingress-lb
        name: traefik-ingress-lb
    spec:
      serviceAccountName: traefik-ingress-controller
      terminationGracePeriodSeconds: 60
      hostNetwork: true
      containers:
      - image: traefik
        name: traefik-ingress-lb
        resources:
          limits:
            cpu: 200m
            memory: 30Mi
          requests:
            cpu: 100m
            memory: 20Mi
        ports:
        - name: http
          containerPort: 80
          hostPort: 80
        - name: admin
          containerPort: 8081
        args:
        - -d
        - --web
        - --web.address=:8081
        - --kubernetes
      nodeSelector:
        role: edge-router
```

First, we created a `ClusterRole` named `traefik-ingress-controller`.

```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
rules:
  - apiGroups:
      - ""
    resources:
      - pods
      - services
      - endpoints
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
```

Second, we created a ServiceAccount named `traefik-ingress-controller` under the namespace `kube-system.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
```

Third, we created a `ClusterRoleBinding` to bind them to together. 

```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-controller
subjects:
- kind: ServiceAccount
  name: traefik-ingress-controller
  namespace: kube-system
```

And in the last, we create a `DaemonSet` with the same name. 

Please be attention, when we deploy a DaemonSet, it should assign pods to every node, including the Master nodes and Worker nodes. They would be totally 30 pods running in the current cluster!

Sometimes we like to have this kind of results, but sometimes we dont. For that kind of situation , we can use `nodeSelector` to let the pod deployed on the nodes we chosen.

You can see we're using the `nodeSelector` in our Traefik DaemonSet deployment in the very end of the yaml file.

```yaml
      nodeSelector:
        role: edge-router
```

But if we directly deploy this , there will no pod being created. For there is no nodes can fit the requirement with the `role: edge-router`, so kubernetes will not create any pod on any nodes.

Now let's create the relate label to the nodes we need.

First, run `kubectl get nodes` to get the names of your cluster nodes.

```bash
[root@e11k8sctl01 ~]# kubectl get nodes
NAME             STATUS                     AGE       VERSION
172.16.164.111   Ready,SchedulingDisabled   118d      v1.6.4+coreos.0
172.16.164.112   Ready,SchedulingDisabled   117d      v1.6.4+coreos.0
172.16.164.113   Ready,SchedulingDisabled   117d      v1.6.4+coreos.0
172.16.164.121   Ready                      117d      v1.6.4+coreos.0
172.16.164.122   Ready                      117d      v1.6.4+coreos.0
172.16.164.123   Ready                      115d      v1.6.4+coreos.0
172.16.164.124   Ready                      115d      v1.6.4+coreos.0
172.16.164.125   Ready                      115d      v1.6.4+coreos.0
........
```
Here, I don't need all the node to have the traefik ingress controller deployed. I will just use `172.16.164.121`, `172.16.164.123` and `172.16.164.125`.

so let's attach the label `role: edge-router` to these three nodes by running :

```bash 
kubectl label nodes 172.16.164.121 role=edge-router

kubectl label nodes 172.16.164.123 role=edge-router

kubectl label nodes 172.16.164.125 role=edge-router
```

Then we can run `kubectl get nodes --show-labels` to verify if the labels have been attached properly.

```bash
[root@e11k8sctl01 ~]# kubectl get nodes --show-labels
NAME             STATUS                     AGE       VERSION           LABELS
...
172.16.164.121   Ready                      117d      v1.6.4+coreos.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=172.16.164.121,role=edge-router
172.16.164.122   Ready                      117d      v1.6.4+coreos.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=172.16.164.122
172.16.164.123   Ready                      115d      v1.6.4+coreos.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=172.16.164.123,role=edge-router
172.16.164.124   Ready                      115d      v1.6.4+coreos.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=172.16.164.124
172.16.164.125   Ready                      115d      v1.6.4+coreos.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=172.16.164.125,role=edge-router
...
```

Then we shall see the traefik ingress pods should appear, and they were deployed on the only 3 nodes we selected.

```bash
[root@e11k8sctl01 ~]# kubectl get pods -n kube-system -o wide| grep traefik
traefik-ingress-controller-3nhks                            1/1       Running   0          9m        172.16.164.121   172.16.164.121
traefik-ingress-controller-knlbc                            1/1       Running   1          114d      172.16.164.123   172.16.164.123
traefik-ingress-controller-w73hm                            1/1       Running   2          114d      172.16.164.125   172.16.164.125
```

If you want to add more node or remove one nodes, you can always label or remove label from one node.

Add label to node 

```
kubectl label node <nodename> <labelname>=<labelValue>
```

Remove label from Node

```
kubectl label node <nodename> <labelname>-
```

After our traefik ingress controller has been deployed, let's move on and create some ingress.