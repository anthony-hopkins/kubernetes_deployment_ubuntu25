### Deploying a 3-Node Kubernetes Cluster on Ubuntu 25.10 (Questing Quokka) with Cilium CNI

This guide provides a step-by-step method for deploying a 3-node Kubernetes cluster (1 controller and 2 workers) on Ubuntu 25 using `kubeadm` and `Cilium` as the CNI.

#### Cluster Topology
- **Controller (1):** Master Node
- **Worker Nodes (2):** Worker-1, Worker-2

---

### 1. Prerequisites (All Nodes)

Perform these steps on all three nodes.

#### 1.1 Update System and Install Dependencies
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y apt-transport-https ca-certificates curl gpg
```

#### 1.2 Disable Swap
Kubernetes requires swap to be disabled for the kubelet to function correctly.
```bash
sudo swapoff -a
# To make it permanent, comment out any swap entry in /etc/fstab
sudo sed -i '/swap/s/^/#/' /etc/fstab
```

#### 1.3 Configure Networking (IP Forwarding)
Enable the `overlay` and `br_netfilter` kernel modules and set sysctl parameters.
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

#### 1.4 Configure Firewall
The following ports must be open for the cluster to function correctly. Ensure that routing is allowed.

##### 1.4.1 Control Plane (Controller Node Only)
```bash
sudo ufw allow 6443/tcp                   # Kubernetes API server
sudo ufw allow 2379:2380/tcp             # etcd server client API
sudo ufw allow 10250/tcp                  # Kubelet API
sudo ufw allow 10259/tcp                  # kube-scheduler
sudo ufw allow 10257/tcp                  # kube-controller-manager
```

##### 1.4.2 Worker Nodes (Worker Nodes Only)
```bash
sudo ufw allow 10250/tcp                  # Kubelet API
sudo ufw allow 30000:32767/tcp            # NodePort Services
sudo ufw allow 8472/udp                    # Flannel CNI (optional, if migrating)
sudo ufw allow 179/tcp                     # Calico CNI (BGP)
sudo ufw allow 5473/tcp                    # Calico CNI (Typha, optional)
sudo ufw allow 4240/tcp                    # Cilium (Health Check)
```

##### 1.4.3 Additional Firewall Configuration (All Nodes)
Ensure that routing is allowed via the firewall:
```bash
sudo ufw default allow routed
sudo ufw reload
```

---

### 2. Install Container Runtime (All Nodes)

We will use `containerd`.

#### 2.1 Install containerd
```bash
sudo apt install -y containerd
```

#### 2.2 Configure containerd to use SystemdCgroup
Generate the default config and ensure `SystemdCgroup` is set to `true`.
```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
sudo systemctl restart containerd
```

---

### 3. Install Kubernetes Tools (All Nodes)

Install `kubeadm`, `kubelet`, and `kubectl`.

#### 3.1 Add Kubernetes Repository (v1.32 or latest)
```bash
# Download the public signing key for the Kubernetes package repositories
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add the appropriate kubernetes apt repository
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

#### 3.2 Install Packages
```bash
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

---

### 4. Initialize the Control Plane (Controller Node Only)

On the **Controller** node:

#### 4.1 Initialize Cluster
```bash
sudo kubeadm init --skip-phases=addon/kube-proxy
```
*Note: This will output a kubectl join command for the worker nodes. Ensure you copy and save this somewhere for that step later.*

*Note: We skip `kube-proxy` because Cilium will replace its functionality for better performance and security.*

#### 4.2 Configure kubectl for the User
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### 4.3 Configure API Server Bind Address
To ensure the API server binds to the proper IPv4 interface, verify the `--bind-address` option in the api server manifest.

Edit `/etc/kubernetes/manifests/kube-apiserver.yaml` and ensure `- --bind-address=0.0.0.0` is present directly below the `--advertise-address` line:
```yaml
    - --advertise-address=<YOUR_CONTROLLER_IP>
    - --bind-address=0.0.0.0
```

#### 4.4 Remove host entry from probes
Remove the `host` entry from `/etc/kubernetes/manifests/kube-apiserver.yaml` in the `probes` section to ensure the API server uses the local interface. Locate the `livenessProbe`, `readinessProbe`, and `startupProbe` and remove the line `host: <IP>`:
```yaml
livenessProbe:
      failureThreshold: 8
      httpGet:
        path: /livez
        port: 6443
        scheme: HTTPS
      initialDelaySeconds: 10
      periodSeconds: 10
      timeoutSeconds: 15
    name: kube-apiserver
    readinessProbe:
      failureThreshold: 3
      httpGet:
        path: /readyz
        port: 6443
        scheme: HTTPS
      periodSeconds: 1
      timeoutSeconds: 15
    resources:
      requests:
        cpu: 250m
    startupProbe:
      failureThreshold: 24
      httpGet:
        path: /livez
        port: 6443
        scheme: HTTPS
      initialDelaySeconds: 10
      periodSeconds: 10
      timeoutSeconds: 15
```

#### 4.5 Install Cilium CNI
Install the Cilium CLI and deploy Cilium to the cluster.
```bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}

cilium install --set kubeProxyReplacement=true
```

---

### 5. Join Worker Nodes (Worker Nodes Only)

After `kubeadm init` finishes on the controller, it will output a join command. Run this command on **both worker nodes**.

```bash
sudo kubeadm join <CONTROLLER_IP>:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```

---

### 6. Verify Cluster Status

On the **Controller** node:
```bash
kubectl get nodes
cilium status
```
You should see all three nodes listed with a status of `Ready` and Cilium showing as `OK`.
