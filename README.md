# kubernetes-kubeadm

## Install Kubernetes Cluster using kubeadm
In this documentation we will see how to setup a Kubernetes cluster on __CentOS 8__.

## Assumptions

All VMs are provisioned with Vagrant.
See Vagrantfile for details

master-node : 

IP : 172.16.1.50

RAM : 2G

CPU : 2

node-1 :

IP : 172.16.1.51

RAM : 2G

CPU : 2

node-2 :

IP : 172.16.1.52

RAM : 2G

CPU : 2

## On all VMs

Perform all the commands as root user

##### Config Host file

```
# For master
hostnamectl set-hostname master-node
# For Worker1
hostnamectl set-hostname 'node-1'
# For Worker1
hostnamectl set-hostname 'node-2'

cat <<EOF>> /etc/hosts
172.16.1.50 master-node
172.16.1.51 node-1 worker-node-1
172.16.1.52 node-2 worker-node-2
EOF
```

##### Disable SELinux
```
setenforce 0
sed -i --follow-symlinks 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/sysconfig/selinux
```

##### Disable Firewall

On Dev mode
```
systemctl disable firewalld; systemctl stop firewalld
```

On Prod mode
```
firewall-cmd --permanent --add-port=6783/tcp
firewall-cmd --permanent --add-port=10250/tcp
firewall-cmd --permanent --add-port=10255/tcp
firewall-cmd --permanent --add-port=30000-32767/tcp
firewall-cmd --reload
# this option "br_netfilter" only for master
modprobe br_netfilter
echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables
```


##### Disable swap
```
swapoff -a; sed -i '/swap/d' /etc/fstab
```

##### Update sysctl settings for Kubernetes networking
```
cat >>/etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```
##### Install Docker
```
dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
dnf install docker-ce --nobest -y
systemctl enable docker
systemctl start docker
```
### Kubernetes Cluster Setup
##### Add yum repository
```
cat >>/etc/yum.repos.d/kubernetes.repo<<EOF
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
        https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
```
##### Install Kubernetes components
```
dnf install -y kubeadm-1.21.2-0 kubelet-1.21.2-0 kubectl-1.21.2-0
```
##### Enable and Start kubelet service
```
systemctl enable --now kubelet
```
## On kmaster
##### Initialize Kubernetes Cluster
```
kubeadm init --apiserver-advertise-address=172.16.1.50 --pod-network-cidr=192.168.0.0/16
```
##### Config root user Or another user 

```
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

Alternatively, if you are the root user, you can run:

```
export KUBECONFIG=/etc/kubernetes/admin.conf
```


##### Deploy Flannel network

In this step we need to modify the yml file before use it because flannel by default use eth0 interface. We need to configure it with the right network interface.

In my case the network interface should be used is : eth1

```
wget https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

```
kubectl apply -f kube-flannel.yml
```

##### Cluster join command
```
kubeadm token create --print-join-command
```

## On Workers
##### Join the cluster
Use the output from __kubeadm token create__ command in previous step and run it here.

## Verifying the cluster
##### On Master get Nodes status
```
kubectl get nodes

NAME          STATUS   ROLES                  AGE   VERSION
master-node   Ready    control-plane,master   38m   v1.21.2
node-1        Ready    <none>                 25m   v1.21.2
node-2        Ready    <none>                 25m   v1.21.2


```
##### Get component status
```
kubectl get cs
# if you got an error scheduler and controller-manager "connection refused" like below

NAME                 STATUS      MESSAGE                                                                  ERROR
controller-manager   Unhealthy   Get "http://127.0.0.1:10252/healthz": dial tcp 127.0.0.1:10252: connect: connection refused
scheduler            Unhealthy   Get "http://127.0.0.1:10251/healthz": dial tcp 127.0.0.1:10251: connect: connection refused
etcd-0               Healthy     {"health":"true"}
```

follow those steps 

```
vi /etc/kubernetes/manifests/kube-scheduler.yaml
# delete the line (spec->containers->command) containing this args: - --port=0

vi /etc/kubernetes/manifests/kube-controller-manager.yaml
# delete the line (spec->containers->command) containing this args: - --port=0

systemctl restart kubelet.service

# and should be resolved ;) 

NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   {"health":"true"}


```

### Hope this will be usefull for you :)