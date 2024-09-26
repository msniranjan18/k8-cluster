## Important note: steps taken from: https://hbayraktar.medium.com/how-to-install-kubernetes-cluster-on-ubuntu-22-04-step-by-step-guide-7dbf7e8f5f99
Some of the steps which were missing or I used seperatly.

Disclaimer: This is just for education puspose, I am not claiming the below steps.


# Step 1: Update and Upgrade Ubuntu (all nodes)
sudo apt update && sudo apt upgrade -y

# Step 2: Disable Swap (all nodes)
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Step 3: Add Kernel Parameters (all nodes)
Load the required kernel modules on all nodes:
sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter


sudo tee /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

reload the changes:
sudo sysctl --system

# Step 4: Install Containerd Runtime (all nodes)
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/docker.gpg
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt update
sudo apt install -y containerd.io
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd

# Step 5: Add Apt Repository for Kubernetes (all nodes)
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --

# Step 6: Install Kubectl, Kubeadm, and Kubelet (all nodes)
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl


# Step 7: Initialize Kubernetes Cluster with Kubeadm (master node)
sudo kubeadm init

After the initialization is complete make a note of the kubeadm join command for future reference.

Run the following commands on the master node:
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Step 8: Install a Pod Network (CNI Plugin)
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

check the cluster and node status:
kubectl get nodes

check the component health:
kubectl get componentstatuses

# Step 9: Add Worker Nodes to the Cluster (worker nodes)
use the kubeadm join command you noted down earlier from kubeadm init output.

# Step 10: Install k8 dashboard (optional):
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.5/aio/deploy/recommended.yaml

Create a new file called dashboard-adminuser.yaml:
```apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: admin-user
    namespace: kubernetes-dashboard
```
apply the file:
kubectl apply -f dashboard-adminuser.yaml

Get the Bearer Token
kubectl -n kubernetes-dashboard create token admin-user

make a note of the bearer token

Access the Dashboard via kubectl proxy:
kubectl proxy &

use below url for dashboard:
http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/

# Step 11: Add the master and worker label to the nodes (optional):
kubectl label node <master-node-name> node-role.kubernetes.io/master=
kubectl label node <worker-node-name> node-role.kubernetes.io/worker=

# Step 12: Enable Bash Completion (optional):
sudo apt-get install bash-completion

Make sure to add the following line to your .bashrc:
```
[[ -r /usr/share/bash-completion/bash_completion ]] && . /usr/share/bash-completion/bash_completion
```

# Step 13: Enable kubectl Autocompletion (optional):
echo "source <(kubectl completion bash)" >> ~/.bashrc
source ~/.bashrc


# Remove the kubernetes 
ref: https://webhostinggeeks.com/howto/how-to-uninstall-kubernetes-on-ubuntu/

kubectl delete all --all-namespaces --all
sudo apt-get purge kubeadm kubectl kubelet kubernetes-cni kube*   
sudo apt-get autoremove 
sudo rm -rf ~/.kube
sudo rm -rf /etc/cni
sudo rm -rf /etc/kubernetes
sudo rm -rf /var/lib/etcd
sudo rm -rf /var/lib/kubelet
Reset iptables:
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X
Revert Changes to the /etc/hosts File: check the host file and remove them if added for k8.



















