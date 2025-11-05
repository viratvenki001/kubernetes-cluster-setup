# kubernetes-cluster-setup

Minimum Requirement 2 CPU cores and 4GB RAM for master
ubuntu version used 22.04


Below block of Steps to be run on both master and worker

*************************************both master and worker***************************************
sudo apt update && sudo apt upgrade -y
sudo apt install -y apt-transport-https ca-certificates curl gpg

#Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

#Enable required kernel modules and settings
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system


#Install Container Runtime (Containerd)
sudo apt install -y containerd

#Configure containerd with systemd as cgroup driver
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd



#Install Kubernetes Tools
sudo curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list


#Install kubeadm, kubelet, kubectl
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl


sudo systemctl enable --now kubelet

********************************************************************************

Once all the above steps are done on all the nodes, run below commands on mentioned nodes




Below block of steps only on master

*********************************only on master************************************************

sudo kubeadm init
#When complete, you’ll see a kubeadm join ... command — copy that for joining workers.

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config


kubectl get nodes
#Should show the control plane as NotReady until networking is installed.


#Install Pod Network (e.g., Calico)
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.2/manifests/calico.yaml


Wait a few minutes, then:

kubectl get nodes
#All should become Ready.


kubectl get pods -A
#all pods should be running

*****************************************************************************



Below block of steps only on Worker

************************************only on Worker****************************

#On each worker node, run the command shown at the end of kubeadm init, e.g.:
#below command should be picked from ur kubeadm init command output

sudo kubeadm join 192.168.56.10:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>

*****************************************************************************




Finally, verify on the master:

kubectl get nodes
#should see all master and worker node listed and ready
