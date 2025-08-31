# Day 1 – Kubernetes Foundations

## Key Concepts
- Control plane runs the brains (API server, scheduler, controller-manager, etcd).
- Worker nodes run workloads (kubelet, kube-proxy, pods).

## Cluster Setup
- Proxmox → 3x Ubuntu VMs.
- Installed K3s with embedded etcd.
- Verified with `kubectl get nodes`.

## Hands-On
- Deployed `nginx` with 2 replicas.
- Exposed via NodePort → accessible at `<nodeIP>:<port>`.

## Questions for Myself
- How does kube-proxy know where pods are running?
- What happens if the control-plane VM is shut down?
