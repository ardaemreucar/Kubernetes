# Install Kubernetes on Amazon Linux 2


## Install Docker

```bash
sudo yum update -y
sudo amazon-linux-extras install -y docker
sudo service docker start
sudo systemctl enable docker
sudo usermod -a -G docker ec2-user
newgrp -
docker info
sudo reboot now  # if needed
```

## Install Kubernetes

cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

- Set SELinux in permissive mode (effectively disabling it)
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

- install on controller and worker
sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

sudo systemctl enable --now kubelet


- iptables

cat << EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system

- cgroup driver

```bash
echo '{"exec-opts": ["native.cgroupdriver=systemd"]}' | sudo tee /etc/docker/daemon.json
sudo systemctl daemon-reload
sudo systemctl restart docker
sudo systemctl restart kubelet
```

- prepare env

```bash
sudo kubeadm init --apiserver-advertise-address=172.31.76.33 --pod-network-cidr=10.244.0.0/16 --ignore-preflight-errors=NumCPU
```

> :warning: **Note**: If you are working on `t2.micro` or `t2.small` instances,  use the command with `--ignore-preflight-errors=NumCPU` as shown below to ignore the errors.

>```bash
>sudo kubeadm init --apiserver-advertise-address=172.31.76.33 --pod-network-cidr=10.244.0.0/16 --ignore-preflight-errors=NumCPU
>```

> **Note**: There are a bunch of pod network providers and some of them use pre-defined `--pod-network-cidr` block. Check the documentation at the References part. We will use Flannel for pod network and Flannel uses 10.244.0.0/16 CIDR block. 

- In case of problems, use following command to reset the initialization and restart from Part 2 (Setting Up Master Node for Kubernetes).

```bash
sudo kubeadm reset
```

- Solution for kubelet error:
- create a yaml file in home directory
```yaml
# kubeadm-config.yaml
kind: ClusterConfiguration
apiVersion: kubeadm.k8s.io/v1beta3
kubernetesVersion: v1.27.0
---
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: systemd
```
- run kubeadm using this file as config file

```bash
sudo kubeadm init --config kubeadm-config.yaml
```

- After successful initialization, you should see something similar to the following output (shortened version).

```bash
...
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

  kubeadm join 172.31.3.109:6443 --token 1aiej0.kf0t4on7c7bm2hpa \
      --discovery-token-ca-cert-hash sha256:0e2abfb56733665c0e6204217fef34be2a4f3c4b8d1ea44dff85666ddf722c02
```

> Note to the Instructor: Note down the `kubeadm join ...` part in order to connect your worker nodes to the master node. Remember to run this command with `sudo`.

- Run following commands to set up local `kubeconfig` on master node.

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

- Activate the `Calico` pod networking and explain briefly the about network add-ons on `https://kubernetes.io/docs/concepts/cluster-administration/addons/`.

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml
```