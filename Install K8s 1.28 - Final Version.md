### ALL:
Rename VMs
k8s-cp01
k8s-n01

# Set Manual IP/subnet/gateway/dns Address and Update printf line with 3 nodes values
printf "\n192.168.4.121 k8s-nd01\n192.168.4.120 k8s-cp01\n\n" >> /etc/hosts

Test connection
ping k8s-cp01
ping k8s-nd01

# Restart VM

sudo -s
# Forwarding IPv4 and letting iptables see bridged traffic
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system


# Install Containerd
wget https://github.com/containerd/containerd/releases/download/v1.7.5/containerd-1.7.5-linux-amd64.tar.gz -P /tmp/
tar Cxzvf /usr/local /tmp/containerd-1.7.5-linux-amd64.tar.gz

wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service -P /etc/systemd/system/
systemctl daemon-reload
systemctl enable --now containerd

# Install Runc
wget https://github.com/opencontainers/runc/releases/download/v1.1.9/runc.amd64 -P /tmp/
install -m 755 /tmp/runc.amd64 /usr/local/sbin/runc

# Install cni
wget https://github.com/containernetworking/plugins/releases/download/v1.3.0/cni-plugins-linux-amd64-v1.3.0.tgz -P /tmp/
mkdir -p /opt/cni/bin
tar Cxzvf /opt/cni/bin /tmp/cni-plugins-linux-amd64-v1.3.0.tgz


mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml
# manually edit and change systemdCgroup to true and sandbox_image to "registry.k8s.io/pause:3.9"
nano /etc/containerd/config.toml
systemctl restart containerd

# Disable swap
swapoff -a
nano /etc/fstab

# Kubernetes package repositories
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

# Download the public signing key for the Kubernetes package repositories
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add the appropriate Kubernetes apt repository
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Update the apt package index, install kubelet, kubeadm and kubectl, and pin their version
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

reboot

free -m


### Only on Control Node/Control Plane
kubeadm init --pod-network-cidr 10.10.0.0/16 --kubernetes-version 1.28.1 --node-name k8s-cp01

# add Calico 3.25 CNI
### https://docs.tigera.io/calico/3.25/getting-started/kubernetes/quickstart
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml
wget https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/custom-resources.yaml
nano custom-resources.yaml <<<<<< edit the CIDR for pods if its custom
kubectl apply -f custom-resources.yaml


# Join additional nodes into cluster
# before join node, check if service kubelet is auto-restart, if dead, reboot the server
systemctl status kubelet

kubeadm token create --print-join-command