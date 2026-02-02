---
tags:
  - Kubernetes
---

%%Date: 2023-11-13%%



---
## Using Kubeadm


> [!tip] Resulting Instances and Their Private IP Addresses
> k-master: `10.0.1.82`
> worker1: `10.0.1.57`
> worker2: `10.0.1.119`
> 

^b8d759

### Step 1: Master
I will launch an EC2 instance with the following configuration:
- AMI: Ubuntu 20.04 %%==Doesn't work with 22==%%
- Instance Type: t2.medium
- Security Group Settings: Configured to allow all traffic.

#### Installing everything needed for Kubeadm
```bash
sudo apt update -y
sudo apt upgrade -y
sudo apt install docker.io -y
sudo apt install -y curl apt-transport-https ca-certificates software-properties-common
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
sudo add-apt-repository -y "deb http://apt.kubernetes.io/ kubernetes-xenial main"
sudo swapoff -a
sudo apt update -y
sudo apt install -y kubelet kubeadm kubectl
```

To verify the installed versions of Kubernetes components, I use the following commands:
```bash
kubectl version --client
kubeadm version
kubelet --version
```


![[Pasted image 20231113111114.png]]


> **Note:** As of the current date, the latest version installed is 1.28.

#### Initializing kubeadm
To establish the control plane of my Kubernetes cluster using the Calico network plugin, I run the `kubeadm` initialization with a specified pod network CIDR block. The command I use is:
```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```
> This command sets up the pod networking configuration necessary for the Calico plugin to operate properly within the cluster.

%%#question why here we had to specify advertise 
![[Pasted image 20231107230634.png]]


<br>![[Pasted image 20231215211736.png]]
added `--ignore-preflight-errors=NumCPU,Mem` not sure what I did the first time when I did not use the ignore thing
i think i just provision a large enought EC2

%%

<br>![[initialization output.png]]

The output tells us we need to run additional commands:
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Additionally, I make a note of the `kubeadm join` token from the output. This token will be necessary later when I need to join worker nodes to the cluster.

#### Installing pod network (Calico)
To enable communication between pods in my Kubernetes cluster, I'm installing the Calico network plugin. I apply the Calico manifest by executing the following command:
```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```


### Step 2: Worker
I will launch two EC2 worker instances simultaneously by adjusting the "Number of Instances" field during the launch process. The specifications for these instances are as follows:
- AMI: Ubuntu 20.04
- Instance Type: t2.micro
- Security Group Configuration: Allow all traffic
- 
Advanced details - User data
```bash
#!/bin/bash
sudo apt update -y
sudo apt upgrade -y
sudo apt install docker.io -y
sudo apt install -y curl apt-transport-https ca-certificates software-properties-common
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
sudo add-apt-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
sudo swapoff -a
sudo apt update -y
sudo apt install -y kubelet kubeadm kubectl

#To join cluster
kubeadm join 10.0.1.82:6443 --token kou2h9.2j7chwdfbddi0ch8 --discovery-token-ca-cert-hash sha256:699dd6c5437b2f148d734d360166d7acd7658ae341f57dc909d7c50ba3f4255e
```
> The script installs all the required components for `kubeadm` to operate just as it does on the master node. I have also included the `kubeadm join` token from the [[initialization output.png|output]] of the master's initialization process. With this token, I add this machine to the cluster as a worker node.


> [!NOTE]
> In case we want to retrieve the join token from master we run:
> ```bash
> sudo kubeadm token create --print-join-command
> ```

### Step3: Verification
After the setup, I verify the cluster by running the command `kubectl get nodes`. This command outputs the [[Installing Kubernetes (Kubeadm)#^b8d759||expected IP addresses]] of the nodes, confirming their statuses as 'Ready'. The output indicates that each node is successfully connected to the cluster and is operational, including the master node labeled as the 'control-plane'.

<br>![[Pasted image 20231113170903.png]]



%%
# OLD UNSUCSSEFUL TRY
### Kubeadm

First create and EC2 instance [[Assignment 1 – EC2_Module2_AWS Weekday BC = 2301080808]]

t2.medium

t2.micro
Ubuntu
MyVPC
Public IP
Allow all
Public Subnet

### **Master** `10.0.1.47` ^936bcc
- Kubeadm
- Kubelet
- Container runtime (e.g., Docker, containerd)
- Networking plugin (e.g., Flannel, Calico)
[Installing kubeadm | Kubernetes](https://v1-27.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
```bash
sudo apt-get update -y

#packages only for Kubernetes 1.27
sudo apt-get install -y apt-transport-https ca-certificates curl

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.27/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.27/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list


sudo apt-get update -y
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

```bash
sudo apt-get install docker.io -y
sudo systemctl enable docker
```

### **Worker** `10.0.1.214` ^33e62d
The worker nodes need to have the following software installed: 

- Kubelet
- Container runtime (e.g., Docker, containerd)
- Networking plugin (e.g., Flannel, Calico)
```bash
sudo apt-get update -y

#packages only for Kubernetes 1.27
sudo apt-get install -y apt-transport-https ca-certificates curl

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.27/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.27/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list


sudo apt-get update -y
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
sudo apt-get install -y kubelet kubeadm kubectl
```bash
sudo apt-get install docker.io -y
sudo systemctl enable docker
```

### Initializing the [[Installing Kubernetes (Kubeadm)#^936bcc|master]]
[Creating a cluster with kubeadm | Kubernetes](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
[Creating a cluster with kubeadm | Kubernetes](https://v1-27.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
```bash
sudo kubeadm init --apiserver-advertise-address=10.0.1.47 --pod-network-cidr=192.168.0.0/16
```
> [!NOTE]- Command explained
> The `kubeadm init` command is used to initialize a Kubernetes cluster. It sets up the necessary components to get a cluster up and running. The output in the image you've uploaded shows this command being run with specific flags and the resulting messages from the command. Here's a breakdown:
> 
> 1. `kubeadm init`: This is the base command to initialize the master node.
> 2. `--apiserver-advertise-address=72.31.17.194`: This flag specifies the IP address on which the API server will advertise itself. This IP is used by members of the cluster to access the Kubernetes API Server.
> 3. `--pod-network-cidr=192.168.0.0/16`: This flag specifies the range of IP addresses for the pod network. If you’re using a networking plugin which requires a CIDR notation IP range (like Calico, for example), you must set this parameter.

The output
<br>![[Pasted image 20231112202131.png]]
```bash
kubeadm join 10.0.1.47:6443 --token 3h2db8.nqirfc3sjwx4klum \
        --discovery-token-ca-cert-hash sha256:111cde88e502e0ef12563cf60b765763467718bff8b1e1383da9fabdb97fb95a 
```


### Executed it in [[Installing Kubernetes (Kubeadm)#^33e62d|worker]]
```bash

```
<br>![[Pasted image 20231112183738.png|300]]


### In [[Installing Kubernetes (Kubeadm)#^936bcc|master]]

```bash
kubectl get nodes
```

ISSUE:
<br>![[Pasted image 20231112184058.png]]

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

^7c1098

Issue still present
<br>![[Pasted image 20231112184058.png]]

After the [[Installing Kubernetes (Kubeadm)#^7c1098|step above]] i restart and <mark style="background: #BBFABBA6;">works</mark>
```bash
sudo systemctl restart kubelet
```

<br>![[Pasted image 20231112185933.png]]

The status shows Not Ready cuse we dont have a network plugin. I'll install Calico the reason we used the specific CDIR earlier

He installs them with the `apply` command given in [[calico.png|previous calico picture]]

[Install Calico](https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart#install-calico)


%%
