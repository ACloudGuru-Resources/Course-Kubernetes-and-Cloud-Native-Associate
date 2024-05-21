# Building a Kubernetes Cluster
# https://learn.acloud.guru/course/kubernetes-and-cloud-native-associate/learn/e055eacd-8e18-4b34-be8c-faa010698939/8bd99486-8f26-48c4-9b0c-a4a0e8beadd8/watch

# Relevant Documentation

- [Installing kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
- [Creating a cluster with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)

# Lesson Reference

# If you are using cloud playground, create three servers with the following settings:

- Distribution: Ubuntu 20.04 Focal Fossa LTS
- Size: Medium
- Tags: k8s-control, k8s-worker1, k8s-worker2

# If you wish, you can set an appropriate hostname for each node.

# On the control plane node:

sudo hostnamectl set-hostname k8s-control

# On the first worker node:

sudo hostnamectl set-hostname k8s-worker1

# On the second worker node:

sudo hostnamectl set-hostname k8s-worker2

# On all nodes, set up the hosts file to enable all the nodes to reach each other using these hostnames.

sudo vi /etc/hosts

# On all nodes, add the following at the end of the file. You will need to supply the actual private IP address for each node.

<control plane node private IP> k8s-control
<worker node 1 private IP> k8s-worker1
<worker node 2 private IP> k8s-worker2

# Log out of all three servers and log back in to see these changes take effect.

# On all nodes, set up containerd. You will need to load some kernel modules and modify some system settings as part of this process.

cat << EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system

# Install and configure containerd. Note: use the following command "sudo kill -9 $( sudo lsof /var/lib/dpkg/lock-frontend | awk '{ print $2 }' | tail -1 )" if you receive a "E: Unable to acquire the dpkg frontend lock (/var/lib/dpkg/lock-frontend), is another process using it?" message.

sudo apt-get update && sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd

# On all nodes, disable swap.

sudo swapoff -a

# On all nodes, install kubeadm, kubelet, and kubectl.

sudo apt-get update && sudo apt-get install -y apt-transport-https ca-certificates curl gpg
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

sudo systemctl enable --now kubelet

# On the control plane node only, initialize the cluster and set up kubectl access.

sudo kubeadm init --pod-network-cidr 192.168.0.0/16

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Verify the cluster is working.

kubectl get nodes

# Install the Calico network add-on.

kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml

# Get the join command (this command is also printed during kubeadm init . Feel free to simply copy it from there).

kubeadm token create --print-join-command

# Copy the join command from the control plane node. Run it on each worker node as root (i.e. with sudo ).

sudo kubeadm join ...

# On the control plane node, verify all nodes in your cluster are ready. Note that it may take a few moments for all of the nodes to
enter the READY state.

kubectl get nodes
