# Welcome
# Steps to perform the Deployment
0. Prerequisites
1. Install Kubernetes on Master and slave.
4. Add Slave node.
5. Install the Kubernetes Dashboard.
- ## Prerequisites:
  this work must be done in both master and worker node.
  edit /etc/hosts on all nodes to add the hostname and internal ip 
  example:
  ```
     127.0.0.1 localhost
     # (New) Add these names
     [master-internal-ip]    [master-hostname]
     [node-0-internal-ip]    [node-0-hostname]
     ...
     # (new-end)
     # The following lines are desirable for IPv6 capable hosts
     ::1 ip6-localhost ip6-loopback
     fe00::0 ip6-localnet
     ff00::0 ip6-mcastprefix
     ff02::1 ip6-allnodes
     ff02::2 ip6-allrouters
     ff02::3 ip6-allhosts
  ```  
## Install CRI
In Both nodes install: 
- Docker with Containerd:
In master and slave:
* Install Pre-requisites:
 ```shell
  $ sudo apt-get update
  $ sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
  ```
* Add Docker Repo
 ```shell
 $ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
 ```
 ```shell
 # [Optional] Check the GPG key
 $ sudo apt-key fingerprint 0EBFCD88
 ```
-  Let's Select the Stable channel (we don't want any surprise)
```shell
$ sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```
* Instal the docker packages
```shell
$ sudo apt-get update
$ sudo apt-get install docker-ce docker-ce-cli containerd.io
```
* Check the correct install of Docker engine
```shell
$ sudo docker run hello-world
```
* Set up docker daemon:
```shell
# Set up the Docker daemon
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
mkdir -p /etc/systemd/system/docker.service.d
systemctl daemon-reload
systemctl start docker
systemctl daemon-reload
```
* It's convenient to create a new user for docker and kubernetes with this command:
```shell
   $ useradd [username] # It adds the user
   $ passwd [username] # It will ask for a password
   $ groupadd docker,sudo [username] # To add the group
   # Let's test Docker on username:
   $ su [username]
   $ docker run hello-world #if output is correct, you will get the next result:
```
the output:
````
Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu shell

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
````
- Configure Containerd:
Now we'll configure containerd with docker
```shell
   cat > /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

# Setup required sysctl params, these persist across reboots.
cat > /etc/sysctl.d/99-kubernetes-cri.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sysctl --system
# Configure containerd
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
# edit /etc/containerd/config.toml to change systemd_cgroup = false
# to systemd_cgroup = true
 systemctl daemon-reload
 systemctl restart containerd
 systemctl enable containerd
```
- Install and configure kubernetes servers:
# prerequisites:
```shell
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```
<emp>disable swap.</emp> Kubernetes can't work with swap on. To do so type:
```shell
   sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
   sudo swapoff -a
```

```shell
# Install kubernetes:
   sudo apt update
   sudo apt -y install curl apt-transport-https
   curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
   echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
   #Install kubelet, kubeadm, kubectl
   sudo apt -y install vim git curl wget kubelet kubeadm kubectl
   sudo apt-mark hold kubelet kubeadm kubectl
   #Check the right version:
   kubectl version --client && kubeadm version
```
- Initialize master node:
```shell
   lsmod | grep br_netfilter
   systemctl daemon-reload
   systemctl restart kubelet
   # Enable Kubelet service
   sudo systemctl enable kubelet
   # Pull init containers
   sudo kubeadm config images pull
```
- Create cluster (as [username]):
```shell
   # On master node:
kubeadm init --control-plane-endpoint=[external-cluster-ip] --api-server-advertise-address=[internal-ip-host] \ pod-network-cidr=10.244.0.0/16
```
The output should be:
```
[init] Using Kubernetes version: vX.Y.Z
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Activating the kubelet service
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [kubeadm-cp localhost] and IPs [10.138.0.4 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [kubeadm-cp localhost] and IPs [10.138.0.4 127.0.0.1 ::1]
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kubeadm-cp kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 10.138.0.4]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 31.501735 seconds
[uploadconfig] storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-X.Y" in namespace kube-system with the configuration for the kubelets in the cluster
[patchnode] Uploading the CRI Socket information "/var/run/dockershim.sock" to the Node API object "kubeadm-cp" as an annotation
[mark-control-plane] Marking the node kubeadm-cp as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node kubeadm-cp as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: <token>
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstraptoken] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstraptoken] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstraptoken] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstraptoken] creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a Pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  /docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```
- Before joining any node, Add Network drivers for kubernetes:
  <i href="https://docs.projectcalico.org/getting-started/kubernetes/flannel/flannel">follow the next instructions to install flannel network drivers</i>
  taint nodes to add the master node working:
  kubectl taint nodes --all node-role.kubernetes.io/master-
- add a Worker node:
If you want to add a worker node, you must use the join sentence before
```shell
   kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```
- to check the status of cluster run as normal user
```shell
   # In master node
   mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config
   # to check the status of nodes
   kubectl get nodes
```
This will show something like this:
<img src="./kubectl-get-nodes.png" />

Both nodes should appear in "<Ready>" state.

# Install Control panel