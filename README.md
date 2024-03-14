# Documentation: Installing Kubernetes

## 1. System Requirements

Before installing Kubernetes, ensure that your system meets the following requirements:

- **CPU:** At least 2 CPU cores.
- **RAM:** At least 2GB of RAM.

## 2. Provisioning Instances on AWS

Create three instances on AWS, one designated as the master node and the other two as worker nodes.

## 3. Configuration and Installation on Master Instance

Follow these steps to configure and install Kubernetes on the master instance:

1. **Install Docker:**
   ```
   sudo apt install docker.io -y
   ```

2. **Enable Docker:**
   ```
   sudo systemctl enable docker
   ```

3. **Install Kubernetes Signed Key:**
   ```
   curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes.gpg
   ```
   ```
   echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/kubernetes.gpg] http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list
   ```

## 4. Installation Steps on All 3 Nodes

1. **Download Kubectl Binary with Curl:**
   ```
   curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
   ```

2. **Download Kubectl Checksum File:**
   ```
   curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
   ```

3. **Validate Kubectl Checksum:**
   ```
   echo "$(cat kubectl.sha256) kubectl" | sha256sum –check
   ```

4. **Install Kubectl:**
   ```
   sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
   ```
   ```
   chmod +x kubectl
   mkdir -p ~/.local/bin
   mv ./kubectl ~/.local/bin/kubectl
   ```

5. **Update and Install Kubernetes Components:**
   ```
   sudo apt update
   sudo apt install kubeadm kubelet kubectl
   sudo apt-mark hold kubeadm kubelet kubectl
   ```

6. **Verify Installation:**
   ```
   kubeadm version
   ```

## 5. Deploy Kubernetes

### Step 1: Prepare for Kubernetes Deployment

1. **Disable Swap Spaces:**
   ```
   sudo swapoff -a
   ```
   ```
   sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
   ```

2. **Load Required Containerd Modules:**
   ```
   sudo vi /etc/modules-load.d/containerd.conf
   ```
   Add:
   ```
   overlay
   br_netfilter
   ```
   ```
   sudo modprobe overlay
   sudo modprobe br_netfilter
   ```

3. **Configure Kubernetes Networking:**
   ```
   sudo nano /etc/sysctl.d/kubernetes.conf
   ```
   Add:
   ```
   net.bridge.bridge-nf-call-ip6tables = 1
   net.bridge.bridge-nf-call-iptables = 1
   net.ipv4.ip_forward = 1
   ```
   ```
   sudo sysctl –system
   ```

### Step 2: Assign Unique Hostnames for Each Server Node

1. **Set Hostname for Master Node:**
   ```
   sudo hostnamectl set-hostname master-node
   ```

2. **Set Hostname for Worker Nodes:**
   ```
   sudo hostnamectl set-hostname worker01
   ```

### Step 3: Initialize Kubernetes on Master Node

1. **Edit Kubelet Configuration:**
   ```
   sudo nano /etc/default/kubelet
   ```
   Add:
   ```
   KUBELET_EXTRA_ARGS="--cgroup-driver=cgroupfs"
   ```
   ```
   sudo systemctl daemon-reload && sudo systemctl restart kubelet
   ```

2. **Edit Docker Daemon Configuration:**
   ```
   sudo nano /etc/docker/daemon.json
   ```
   Add:
   ```
   {
     "exec-opts": ["native.cgroupdriver=systemd"],
     "log-driver": "json-file",
     "log-opts": {
       "max-size": "100m"
     },
     "storage-driver": "overlay2"
   }
   ```
   ```
   sudo systemctl daemon-reload && sudo systemctl restart docker
   ```

3. **Edit Kubelet Service Configuration:**
   ```
   sudo nano /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
   ```
   Add:
   ```
   Environment="KUBELET_EXTRA_ARGS=--fail-swap-on=false"
   ```
   ```
   sudo systemctl daemon-reload && sudo systemctl restart kubelet
   ```

4. **Initialize the Cluster:**
   ```
   sudo kubeadm init --control-plane-endpoint=master-node –upload-certs
   ```
while initalizing make sure to copy the kubeadm join command, this command will be used to make worker node to get connected to master node
   ```
   mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config
   ```

### Step 4: Deploy Pod Network to Cluster

Apply the Flannel pod network manager:
   ```
   kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
   ```
   ```
   kubectl taint nodes --all node-role.kubernetes.io/control-plane-
   ```

### Step 5: Join Worker Node to Cluster

1. **Stop and Disable AppArmor:**
   ```
   sudo systemctl stop apparmor && sudo systemctl disable apparmor
   ```

2. **Restart Containerd:**
   ```
   sudo systemctl restart containerd.service
   ```

3. **Join Worker Nodes to the Cluster:**
   ```
   sudo kubeadm join [master-node-ip]:6443 --token [token] –discovery-token-ca-cert-hash sha256:[hash]
   ```

4. **Check Node Status:**
   ```
   kubectl get nodes
   ```

## References
- [Kubernetes Documentation - Install kubectl binary with Curl on Linux](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-kubectl-binary-with-curl-on-linux)
