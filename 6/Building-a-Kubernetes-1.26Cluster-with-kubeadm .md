# Building a Kubernetes 1.24 Cluster with kubeadm

## Introduction
This lab will allow you to practice the process of building a new Kubernetes cluster. You will need 2 Ubuntu 20.04 instances. If on EC2, choose t3a.small
This will help you build the skills necessary to create your own Kubernetes clusters in the real world.

### Solution

- Create master and worker Security Groups


- Create two security groups. Name the first security group as master-sec-group and apply the following Control-plane (Master) Node(s) table to your master node.

- Name the second security group as worker-sec-group, and apply the following Worker Node(s) table to your worker nodes.

### Control-plane (Master) Node(s)

|Protocol|Direction|Port Range|Purpose|Used By|
|---|---|---|---|---|
|TCP|Inbound|6443|Kubernetes API server|All|
|TCP|Inbound|2379-2380|`etcd` server client API|kube-apiserver, etcd|
|TCP|Inbound|10250|Kubelet API|Self, Control plane|
|TCP|Inbound|10259|kube-scheduler|Self|
|TCP|Inbound|10257|kube-controller-manager|Self|
|TCP|Inbound|22|remote access with ssh|Self|
|TCP|Inbound|5473|Cluster-Wide Network Comm. - Calico VXLAN|Self|
|TCP|Inbound|80|HTTP|
|TCP|Inbound|443|HTTPS|
|TCP|Inbound|8080|common|

### Worker Node(s)

|Protocol|Direction|Port Range|Purpose|Used By|
|---|---|---|---|---|
|TCP|Inbound|10250|Kubelet API|Self, Control plane|
|TCP|Inbound|30000-32767|NodePort Services†|All|
|TCP|Inbound|22|remote access with ssh|Self|
|TCP|Inbound|5473|Cluster-Wide Network Comm. - Calico VXLAN|Self|
|TCP|Inbound|80|HTTP|
|TCP|Inbound|443|HTTPS|
|TCP|Inbound|8080|common|


- Create 2 EC2 instances of t3a.small using Ubuntu 20.04 AMI


- Install Packages
Log into the control plane node. (Note: The following steps must be performed on all three nodes.)

```bash
ssh ec2-user@<PUBLIC_IP_ADDRESS>
```

- change the name of host/prompt

```bash
PS1="master : "
```


Create configuration file for containerd:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
```

Load modules:

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

Set system configurations for Kubernetes networking:

```bash
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
```

Apply new settings:

```bash
sudo sysctl --system
```

Install containerd:

```bash
sudo apt-get update && sudo apt-get install -y containerd
```

Create default configuration file for containerd:

```bash
sudo mkdir -p /etc/containerd
```

Generate default containerd configuration and save to the newly created default file:

```bash
sudo containerd config default | sudo tee /etc/containerd/config.toml
```

Restart containerd to ensure new configuration file usage:

```bash
sudo systemctl restart containerd
```

Verify that containerd is running:

```bash
sudo systemctl status containerd
```

Disable swap:
(Amazon EC2 instances do not need this setting)

```bash
sudo swapoff -a
```

Install dependency packages:

```bash
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
```

Download and add GPG key:

```bash
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
```

Add Kubernetes to repository list:

```bash
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
```

Update package listings:

```bash
sudo apt-get update
```

Install Kubernetes packages (Note: If you get a dpkg lock message, just wait a minute or two before trying the command again):

```bash
sudo apt-get install -y kubelet=1.26.0-00 kubeadm=1.26.0-00 kubectl=1.26.0-00
```

Turn off automatic updates:

```bash
sudo apt-mark hold kubelet kubeadm kubectl
```

Log into both worker nodes to perform previous steps.
- change the name of host/prompt

```bash
PS1="master : "
```


### Initialize the Cluster

Initialize the Kubernetes cluster on the control plane node using kubeadm (Note: This is only performed on the Control Plane Node):

```bash
sudo kubeadm init --pod-network-cidr 192.168.0.0/16 --kubernetes-version 1.26.0
```

Set kubectl access:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Test access to cluster:

```bash
kubectl get nodes
```

- Install the Calico Network Add-On

On the control plane node, install Calico Networking:

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml
```

Check status of the control plane node:

```bash
kubectl get nodes
```

- Join the Worker Nodes to the Cluster

In the control plane node, create the token and copy the kubeadm join command (NOTE:The join command can also be found in the output from kubeadm init command):

```bash
kubeadm token create --print-join-command
```

In both worker nodes, paste the kubeadm join command to join the cluster. Use sudo to run it as root:

```bash
sudo kubeadm join ...
```

In the control plane node, view cluster status (Note: You may have to wait a few moments to allow all nodes to become ready):

```bash
kubectl get nodes
```

- Change the role/label of the worker that reads <none> to `worker`

```bash
kubectl label node kube-worker-1 node-role.kubernetes.io/worker=worker
```


### Deploy a Simple Nginx Server on Kubernetes

- Check the readiness of nodes at the cluster on master node.

```bash
kubectl get nodes
```

- Show the list of existing pods in default namespace on master. Since we haven't created any pods, list should be empty.

```bash
kubectl get pods
```

- Get the details of pods in all namespaces on master. Note that pods of Kubernetes service are running on the master node and also additional pods are running on the worker nodes to provide communication and management for Kubernetes service.

```bash
kubectl get pods -o wide --all-namespaces
```

- Create and run a simple `Nginx` Server image.

```bash
kubectl run nginx-server --image=nginx  --port=80
```

- Get the list of pods in default namespace on master and check the status and readyness of `nginx-server`

```bash
kubectl get pods -o wide
```

- Expose the nginx-server pod as a new Kubernetes service on master.

```bash
kubectl expose pod nginx-server --port=80 --type=NodePort
```

- Get the list of services and show the newly created service of `nginx-server`

```bash
kubectl get service -o wide
```

- You will get an output like this.

```text
kubernetes     ClusterIP   10.96.0.1       <none>        443/TCP        13m    <none>
nginx-server   NodePort    10.110.144.60   <none>        80:32276/TCP   113s   run=nginx-server
```

- Open a browser and check the `public ip:<NodePort>` of worker node to see Nginx Server is running. In this example, NodePort is 32276.

- Clean the service and pod from the cluster.

```bash
kubectl delete service nginx-server
kubectl delete pods nginx-server
```

- Check there is no pod left in default namespace.

```bash
kubectl get pods
```


# References

- https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

- https://kubernetes.io/docs/concepts/cluster-administration/addons/

- https://kubernetes.io/docs/reference/

- https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#-strong-getting-started-strong-


Conclusion
Congratulations — you've completed this hands-on lab!