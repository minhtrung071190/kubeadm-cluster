Kubernetes Cluster Setup Using Kubeadm
Source: https://devopscube.com/setup-kubernetes-cluster-kubeadm/
Following are the high-level steps involved in setting up a kubeadm-based Kubernetes cluster.

Install container runtime on all nodes- We will be using cri-o.
Install Kubeadm, Kubelet, and kubectl on all the nodes.
Initiate Kubeadm control plane configuration on the master node.
Save the node join command with the token.
Install the Calico network plugin (operator).
Join the worker node to the master node (control plane) using the join command.
Validate all cluster components and nodes.
Install Kubernetes Metrics Server
Deploy a sample app and validate the app

Step 2: Disable swap on all the Nodes
For kubeadm to work properly, you need to disable swap on all the nodes using the following command.
```
sudo swapoff -a
(crontab -l 2>/dev/null; echo "@reboot /sbin/swapoff -a") | crontab - || true
```
The fstab entry will make sure the swap is off on system reboots.

You can also, control swap errors using the kubeadm parameter --ignore-preflight-errors Swap we will look at it in the latter part.

Note: From 1.28 kubeadm has beta support for using swap with kubeadm clusters. Read this to understand more.
Step 3: Install CRI-O Runtime On All The Nodes
Note: We are using cri-o instead if containerd because, in Kubernetes certification exams, cri-o is used as the container runtime in the exam clusters.

The basic requirement for a Kubernetes cluster is a container runtime. You can have any one of the following container runtimes.

CRI-O
containerd
Docker Engine (using cri-dockerd)
We will be using CRI-O instead of Docker for this setup as Kubernetes deprecated Docker engine

Execute the following commands on all the nodes to install required dependencies and the latest version of CRIO.

```
sudo apt-get update -y
sudo apt-get install -y software-properties-common gpg curl apt-transport-https ca-certificates

curl -fsSL https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/deb/Release.key |
   sudo gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/deb/ /" |
   sudo tee /etc/apt/sources.list.d/cri-o.list

sudo apt-get update -y
sudo apt-get install -y cri-o

sudo systemctl daemon-reload
sudo systemctl enable crio --now
sudo systemctl start crio.service
```

Install crictl.

```
VERSION="v1.30.0"
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz
sudo tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/local/bin
rm -f crictl-$VERSION-linux-amd64.tar.gz
```

crictl, a CLI utility to interact with the containers created by the container runtime.

When you use container runtimes other than Docker, you can use the crictl utility to debug containers on the nodes. Also, it is useful in CKS certification where you need to debug containers.

Step 4: Install Kubeadm & Kubelet & Kubectl on all Nodes
Download the GPG key for the Kubernetes APT repository on all the nodes.

```
KUBERNETES_VERSION=1.30

sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v$KUBERNETES_VERSION/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v$KUBERNETES_VERSION/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Update apt repo

```
sudo apt-get update -y
```

Note: If you are preparing for Kubernetes certification, install the specific version of kubernetes. For example, the current Kubernetes version for CKA, CKAD and CKS exams is Kubernetes version 1.30

Install the latest version from the repo use the following command without specifying any version.

```
sudo apt-get install -y kubelet kubeadm kubectl
```

Add hold to the packages to prevent upgrades.

```
sudo apt-mark hold kubelet kubeadm kubectl
```

Now we have all the required utilities and tools for configuring Kubernetes components using kubeadm.

Add the node IP to KUBELET_EXTRA_ARGS.

```
sudo apt-get install -y jq
local_ip="$(ip --json addr show eth0 | jq -r '.[0].addr_info[] | select(.family == "inet") | .local')"
echo KUBELET_EXTRA_ARGS=--node-ip=$local_ip | sudo tee /etc/default/kubelet
```

Step 5: Initialize Kubeadm On Master Node To Setup Control Plane
Here you need to consider two options.

Master Node with Private IP: If you have nodes with only private IP addresses the API server would be accessed over the private IP of the master node.
Master Node With Public IP: If you are setting up a Kubeadm cluster on Cloud platforms and you need master Api server access over the Public IP of the master node server.
Only the Kubeadm initialization command differs for Public and Private IPs.

Execute the commands in this section only on the master node.

If you are using a Private IP for the master Node,

Set the following environment variables. Replace 10.0.0.10 with the IP of your master node.

```
IPADDR="10.0.0.10"
NODENAME=$(hostname -s)
POD_CIDR="192.168.0.0/16"
```

If you want to use the Public IP of the master node,

Set the following environment variables. The IPADDR variable will be automatically set to the server’s public IP using ifconfig.me curl call. You can also replace it with a public IP address

```
IPADDR=$(curl ifconfig.me && echo "")
NODENAME=$(hostname -s)
POD_CIDR="192.168.0.0/16"
```

Now, initialize the master node control plane configurations using the kubeadm command.

For a Private IP address-based setup use the following init command.

```
sudo kubeadm init --apiserver-advertise-address=$IPADDR  --apiserver-cert-extra-sans=$IPADDR  --pod-network-cidr=$POD_CIDR --node-name $NODENAME --ignore-preflight-errors Swap
--ignore-preflight-errors Swap is actually not required as we disabled the swap initially.
```

For Public IP address-based setup use the following init command.

Here instead of --apiserver-advertise-address we use --control-plane-endpoint parameter for the API server endpoint.

```
sudo kubeadm init --control-plane-endpoint=$IPADDR  --apiserver-cert-extra-sans=$IPADDR  --pod-network-cidr=$POD_CIDR --node-name $NODENAME --ignore-preflight-errors Swap
All the other steps are the same as configuring the master node with private IP.
```

Use the following commands from the output to create the kubeconfig in master so that you can use kubectl to interact with cluster API.

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

On a successful kubeadm initialization, you should get an output with kubeconfig file location and the join command with the token as shown below. Copy that and save it to the file. we will need it for joining the worker node to the master.

Step 6: Join Worker Nodes To Kubernetes Master Node
We have set up cri-o, kubelet, and kubeadm utilities on the worker nodes as well.

Now, let’s join the worker node to the master node using the Kubeadm join command you have got in the output while setting up the master node.

If you missed copying the join command, execute the following command in the master node to recreate the token with the join command.

kubeadm token create --print-join-command
Here is what the command looks like. Use sudo if you running as a normal user. This command performs the TLS bootstrapping for the nodes.

```
sudo kubeadm join 10.128.0.37:6443 --token j4eice.33vgvgyf5cxw4u8i \
    --discovery-token-ca-cert-hash sha256:37f94469b58bcc8f26a4aa44441fb17196a585b37288f85e22475b00c36f1c61
```
Step 7: Install Calico Network Plugin for Pod Networking
Kubeadm does not configure any network plugin. You need to install a network plugin of your choice for kubernetes pod networking and enable network policy.

I am using the Calico network plugin for this setup.

Note: Make sure you execute the kubectl command from where you have configured the kubeconfig file. Either from the master of your workstation with the connectivity to the kubernetes API.

Execute the following commands to install the Calico network plugin operator on the cluster.

```
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

After a couple of minutes, if you check the pods in kube-system namespace, you will see calico pods and running CoreDNS pods.
On successful execution, you will see the output saying, “This node has joined the cluster”.
