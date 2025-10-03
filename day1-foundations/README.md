# Day 1 – K3s Cluster Setup on Debian

## 1. Prepare Base VM
- Debian VM at `192.168.1.102` with internet access
- Update system:
  ```bash
  sudo apt update && sudo apt upgrade -y
  sudo apt install -y curl git net-tools htop
  ```
- Disable swap:
  ```bash
  sudo swapoff -a
  ```
  delete any line in fstab with `swap` or use the sed like below:
  ```
  sudo sed -i '/swap/d' /etc/fstab
  ```
- Enable IP forwarding & iptables bridging:
  ```bash
  sudo modprobe br_netfilter
  echo "br_netfilter" | sudo tee /etc/modules-load.d/k8s.conf
  cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
  net.bridge.bridge-nf-call-iptables = 1
  net.ipv4.ip_forward = 1
  EOF
  sudo sysctl --system
  ```

---

## 2. Clone VM in Proxmox
- Create **3 VMs** from the base.
- Assign static IPs:
  - `192.168.1.241` (control-plane)
  - `192.168.1.242` (control-plane or worker)
  - `192.168.1.243` (control-plane or worker)

---

## 3. Form the Cluster
- On **first control-plane node**:
  ```bash
  curl -sfL https://get.k3s.io | sh -s - server --cluster-init
  ```
  Obtain the `TOKEN_STRING`
  ```
  sudo cat /var/lib/rancher/k3s/server/node-token
  ```

- On **other nodes**:
  
  - Must use the `TOKEN_STRING` from first control node
    ```bash
    export TOKEN=<TOKEN_STRING>
    ```
  - To join as **control-plane**:
    ```bash
    curl -sfL https://get.k3s.io | K3S_URL=https://192.168.1.241:6443 K3S_TOKEN="$TOKEN" sh -s - server
    ```
    
  - To join as **worker**:
    ```bash
    curl -sfL https://get.k3s.io | K3S_URL=https://192.168.1.241:6443 K3S_TOKEN="$TOKEN" sh -
    ```

---

Verify:
```bash
sudo kubectl get nodes -o wide
```
Note: Embedded etcd is managed by k3s. You do not install etcd separately; just ensure ports `2379-2380/tcp` are open between servers.

---

## 4. Verify Cluster
```bash
sudo kubectl get nodes -o wide
```

---

## 5. Test App
```bash
kubectl create deployment nginx --image=nginx --replicas=2
kubectl expose deployment nginx --type=NodePort --port=80
kubectl get svc
```

Something like this below

```bash
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.43.0.1      <none>        443/TCP        22h
nginx        NodePort    10.43.240.48   <none>        80:31486/TCP   107s
```
here nodePort = 31486, access via browser at: `http://<any-node-IP>:<nodePort>`

Here’s what happens:

#### kubectl create deployment nginx --image=nginx --replicas=2

This tells the cluster (via the API server):
“Please create a Deployment object named nginx that manages 2 replicas (pods) running the container image nginx:latest.”

The control plane stores this spec in etcd.

The scheduler assigns pods to nodes.

On each node, the kubelet + container runtime (containerd, installed with K3s) pulls the nginx image from Docker Hub (unless it’s already cached locally).

Two pods are created, each running an nginx container.

#### kubectl expose deployment nginx --type=NodePort --port=80

This creates a Service object of type NodePort.

A NodePort Service allocates a port (by default in the range 30000–32767) on every node.

Any request sent to <nodeIP>:<nodePort> will be forwarded to one of the nginx pods (load-balanced).

Again, the spec is stored in etcd; nothing is saved locally on your VM.

#### kubectl get svc

Shows the Service you just created.

You’ll see columns like NAME, TYPE, CLUSTER-IP, EXTERNAL-IP, PORT(S), AGE.

Example:

NAME         TYPE       CLUSTER-IP   EXTERNAL-IP   PORT(S)        AGE
nginx        NodePort   10.43.7.82   <none>        80:31500/TCP   2m


In this example, 31500 is the NodePort → so you can open http://<any-node-IP>:31500 in your browser to hit nginx.

#### 🚫 What it does not do

It does not save deployment files (YAML) to your local folder.
(Everything is stored in the cluster’s etcd datastore.)

It does not require you to have Docker installed separately — K3s uses containerd under the hood.

It does not persist anything outside of Kubernetes.
(If you delete the Deployment, the pods vanish; nothing is written to disk except container layers pulled into /var/lib/rancher/k3s/agent/containerd).

---

## 6. Undeploy and teardown everythingß

#### Undeploy:
```
# Delete deployment (removes pods too)
kubectl delete deployment nginx

# Delete the Service
kubectl delete service nginx

# Confirm everything is gone
kubectl get all
```

#### Explain:

1. Deployment

    A Deployment is a higher-level controller that manages Pods.
    In our test, we created it with:
    ```
    kubectl create deployment nginx --image=nginx --replicas=2
    ```
    This told Kubernetes: “Keep 2 pods of nginx running at all times.” If one dies, recreate it. Store this desired state in etcd.

    When you delete the Deployment:
    ```
    kubectl delete deployment nginx
    ```

    All Pods that belong to it are also deleted. But the Service still exists (it doesn’t automatically delete).

2. Service

    A Service is a stable network abstraction that lets you reach your Pods. In our test, we created it with:
    ```
    kubectl expose deployment nginx --type=NodePort --port=80
    ```
    This told Kubernetes: 
    - “Assign a port (NodePort) on every node.” 
    - “Forward requests to whichever pods the Deployment created.”

    If you delete the Deployment but not the Service, the Service still exists, but it has no backing Pods → traffic goes nowhere.

    To remove the Service:
    ```
    kubectl delete service nginx
    ```

### Teardown everything
For controller node
```
sudo /usr/local/bin/k3s-uninstall.sh
```
for worker node
```
sudo /usr/local/bin/k3s-agent-uninstall.sh
```

## ✅ End of Day 1 Checkpoint
- 3-node K3s cluster running
- Able to deploy and reach workloads
- Ready for HA exploration (Day 2)
