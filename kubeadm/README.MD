# cluster to create kubeadm

## create control-plan
```shell
sudo swapoff -a
sudo hostnamectl set-hostname control-plane
bash
```
## Install container-runtime & configure the Linux
```shell
# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
sysctl net.ipv4.ip_forward
```


## Install containerd for runtime
```shell
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update -y
sudo apt-get install containerd.io -y 
sudo systemctl start containerd
sudo systemctl enable containerd
sudo systemctl status containerd
sudo vim /etc/containerd/config.toml  # main configuration file for containerd 
    # comment: `disabled_plugins =  ["cri"]`
sudo systemctl restart containerd
```
## When Installing the container-engine on Kubernetes kubeadm, we required the three components, those all components need to install manually 
CRI - Container Runtime interface, It will interact container with k8s Engine by default it store in containerd
RUNC - Component to run the container, It to start container on Container Engine 
CNI - Called as a container network interface, Interact & communication the networking of container

# Install CNI components
```shell
# https://github.com/containernetworking/plugins/releases
curl -O https://github.com/containernetworking/plugins/releases/download/v1.4.1/cni-plugins-linux-amd64-v1.4.1.tgz
sudo mkdir -p /opt/cni/bin
sudo tar -xvzf /home/ubuntu/cni-plugins-linux-amd64-v1.4.1.tgz -C /opt/cni/bin
ls /opt/cni/bin
```

# Installing kubeadm, kubelet and kubectl
```shell
sudo apt-get update

# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
# sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update -y
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl start kubelet
sudo systemctl enable kubelet
sudo systemctl enable --now kubelet
sudo systemctl status kubelet
```

# Install Kubeadm with single node-clutser
```shell
sudo kubeadm init --pod-network-cidr "10.0.0.0/16"
sudo kubectl get pods # coudn't connect for that we need kubeconfig file to configure the kubeadm with k8s

# To make kubectl work for your non-root user, run these commands, which are also part of the kubeadm init output:  
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
sudo kubect cluster-info
sudo kubectl get pods
# we can see the pod will not work because of lack of pulgin so here we download the network plugin 
```

# set-up pod network using external project, not managed by k8s.
1. It has very complex network, every node has to install container-engine --> on host --> 2 host connect using overlay.
2. Install Networking and Network Policy - Cicso & Wave Net # https://kubernetes.io/docs/concepts/cluster-administration/addons/#networking-and-network-policy

# Will go with wave-Net
```shell
kubectl apply -f https://reweave.azurewebsites.net/k8s/v1.29/net.yaml

# now try to get nodes
sudo kubectl get nodes
sudo kubectl get ns
sudo kubectl get pods -n kube-system
sudo kubectl get pods -n kube-system

# Control-plane installation has done ! 
```

# Install worker-node and connect to the worker-node to control-plan

Step 1: Creat a new EC2-Instance, Prepare the worker-node for k8s kubeadm
step 2: add worker-node using [Kubeadm join <IP>:6443 -- discovery token] of outut taken from control-plan

Step 3: Creat Node-1 Server 
```shell
sudo swapoff -a
sudo hostnamectl set-hostname node-01
```
## Install container-runtime & configure the Linux
```shell
# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
sysctl net.ipv4.ip_forward
```
## Install containerd for runtime
```shell
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update -y
sudo apt-get install containerd.io -y 
sudo systemctl start containerd
sudo systemctl enable containerd
sudo systemctl status containerd
sudo vim /etc/containerd/config.toml  # main configuration file for containerd 
    # comment: `disabled_plugins =  ["cri"]`
sudo systemctl restart containerd
```

# Install CNI components
```shell
# https://github.com/containernetworking/plugins/releases
curl -O https://github.com/containernetworking/plugins/releases/download/v1.4.1/cni-plugins-linux-amd64-v1.4.1.tgz
sudo mkdir -p /opt/cni/bin
sudo tar -xvzf /home/ubuntu/cni-plugins-linux-amd64-v1.4.1.tgz -C /opt/cni/bin
ls /opt/cni/bin
```

# Installing kubeadm, kubelet and kubectl
```shell
sudo apt-get update

# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
# sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update -y
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl start kubelet
sudo systemctl enable kubelet
sudo systemctl enable --now kubelet
sudo systemctl status kubelet
```

# Run 
kubedeam join ## command

# Finally Run the commands on control-plane
```shell
kubectl get nodes
kubectl run jarvis-pod --image=nginx
kubectl get pods
```