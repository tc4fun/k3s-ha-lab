```
kubectl get pods -A
NAMESPACE     NAME                                      READY   STATUS      RESTARTS      AGE
default       nginx-5869d7778c-6sst5                    1/1     Running     0             18m
default       nginx-5869d7778c-jqhhp                    1/1     Running     0             18m
kube-system   coredns-64fd4b4794-n6r6n                  1/1     Running     1 (21h ago)   22h
kube-system   helm-install-traefik-crd-hdkls            0/1     Completed   0             22h
kube-system   helm-install-traefik-zrdbc                0/1     Completed   1             22h
kube-system   local-path-provisioner-774c6665dc-4fhfx   1/1     Running     1 (21h ago)   22h
kube-system   metrics-server-7bfffcd44-88hs6            1/1     Running     1 (21h ago)   22h
kube-system   svclb-traefik-70ba3939-b4dz6              2/2     Running     4 (21h ago)   22h
kube-system   svclb-traefik-70ba3939-q9prm              2/2     Running     2 (21h ago)   22h
kube-system   svclb-traefik-70ba3939-tgczv              2/2     Running     2             22h
kube-system   traefik-c98fdf6fb-88pfx                   1/1     Running     1 (21h ago)   22h

```


🟦 Namespace: default

nginx-... pods

These are the test app you deployed on Day 1.

Deployment created 2 replicas → hence 2 pods.

Status = Running, so both are healthy.

🟦 Namespace: kube-system

These are system components that K3s installs automatically to make the cluster functional:

coredns-...

Provides DNS inside the cluster (pods can resolve service.namespace.svc.cluster.local).

Every Kubernetes cluster has CoreDNS.

helm-install-traefik-crd-* and helm-install-traefik-*

One-time Helm jobs that K3s uses to install Traefik (the default ingress controller).

Status = Completed → they already ran and exited successfully.

local-path-provisioner-...

K3s’s default storage provisioner.

Lets you create PersistentVolumes backed by local disk (/var/lib/rancher/k3s/storage).

Useful for testing, but not HA storage (we’ll cover that on Day 4).

metrics-server-...

Collects CPU/memory usage from nodes & pods.

Enables commands like:

kubectl top pod
kubectl top node


svclb-traefik-... (multiple pods, each with 2/2 containers running)

This is the ServiceLB implementation built into K3s.

It creates “service load balancer” pods on each node to expose LoadBalancer services without needing MetalLB.

In your case, it’s used to front Traefik.

Each pod runs 2 containers (hence 2/2).

traefik-...

The actual Traefik ingress controller pod.

This listens for Ingress objects (HTTP routes) and proxies traffic into services.


