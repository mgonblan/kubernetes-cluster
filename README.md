# Welcome
  In this guide you will learn about creating a kubernetes cluster with virtual machines.
  There is two ways:
  1. The soft Way.
  2. The Hard way.

# Steps to perform the Deployment
0. Prerequisites
1. Install Kubernetes on Master and slave.
4. Add Slave node.
5. Install the Kubernetes Dashboard.
- ## Prerequisites:
  To deploy a kubernetes cluster on premise, first, have to define several things:
 
  this work must be done in both master and worker node.
  edit /etc/hosts on all nodes to add the hostname and internal ip 
  ```/etc/hosts
     127.0.0.1 localhost
     # (New) Add these names
     [master-internal-ip]    [master-hostname]
     [node-0-internal-ip]    [node-0-hostname]
     [access-external-ip]    [access-domain-name] [access-hostname]
     ...
     # (new-end)
  ```  
## Install CRI
In Both nodes install Docker with Containerd:
1. Install Pre-requisites:
 * Copy and paste this code: 
  ```shell
    $ sudo apt-get update
    $ sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common -y
  ```
* Add Docker Repo
 ```shell
 $ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
 ```
 ```shell
 # [Check the GPG key
 $ sudo apt-key fingerprint 0EBFCD88
 ```
* Let's Select the Stable channel: (we don't want any surprise)
```shell
$ sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```
* Instal the docker packages
```shell
$ sudo apt-get update
$ sudo apt-get install docker-ce docker-ce-cli containerd.io -y
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
*  Create a new user for docker and kubernetes with this command/modify an user to work with docker:
```shell
   $ useradd [username] # It adds the user
   $ passwd [username] # It will ask for a password
   $ groupadd docker,sudo [username] # To add the group
   # Let's test Docker on username:
   $ su [username]
   $ docker run hello-world #if output is correct, you will get the next result:
```
     - modify an user to work with docker:
     ```shell
        usermod -aG docker [username]
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
* Configure Containerd:
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
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
```
edit /etc/containerd/config.toml to cange:
```toml
[plugins.cri]
systemd_cgroup = false
```
for:
```toml
[plugins.cri]
systemd_cgroup = true
```
and
```shell
systemctl start containerd
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
- install kubernetes packages:
```shell

   sudo apt update
   sudo apt -y install curl apt-transport-https
   curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
   echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
   #Install kubelet, kubeadm, kubectl
   sudo apt-get update
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
   # On master node for flannel:
kubeadm init --control-plane-endpoint=[access-external-ip] --pod-network-cidr=[network-zone] --apiserver-advertise-address=master-internal-ip
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
In our case we'll use weave net, an open source tool that provide us networking.
  to do so simply type:
  ```shell
     kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
  ```
  and perform a 
  ```shell 
  kubectl get pods --all-namespaces 
  ```
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
# Isolate control plane node:
```bash
   kubectl taint nodes --all node-role.kubernetes.io/master-
```
# Add new working nodes:
  To add new working nodes, the working nodes must be added to /etc/hosts file
  and they must be in the same network.
  To perform this option type:
  ```bash
     kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
  ```
Both nodes should appear in "<Ready>" state.

# Install Control panel
once the network is ok and both nodes are working fine, we can install Web-UI.
to do so:
On your kubectl client:
1. copy kubeconfig file to your workstation:

```bash
   scp root@[control-plane-cluster]:/etc/kubernetes/admin.conf [local-path-of-kubeconfig]
   EXPORT KUBECONFIG=local-path-of-kubeconfig]:$KUBECONFIG
   # to check it is ok:
   kubectl config current-context
   # and you will have the following output:
   # kubernetes-admin@kubernetes
```
2. apply the web-UI file:
```shell
kubectl create secret generic kubernetes-dashboard-certs --from-file=$HOME/certs -n kubernetes-dashboard
   kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.4/aio/deploy/recommended.yaml

```
   when finished creating all services, pods and ingress, you have to create a user
   that access to dashboard through a token on this way:
   - creating a service account:
```shell
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
EOF
```
- Creating a ClusterRoleBinding:
```shell
cat <<EOF | kubectl apply -f -
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
EOF
```
- Getting a Bearer Token:
``` bash
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
```

```PowerShell
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | sls admin-user | ForEach-Object { $_ -Split '\s+' } | Select -First 1)
```
It should print something like:
```
Name:         admin-user-token-v57nw
Namespace:    kubernetes-dashboard
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin-user
              kubernetes.io/service-account.uid: [account-id]

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1066 bytes
namespace:  20 bytes
token:     [sha-256-page]
```
- enter on kubernetes web UI page:
in kubernetes client terminal:
```shell
kubectl proxy
```
this command will enable the proxy access to the cluster from your local machine.
put the next in your web browser:
<i href="http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/" >http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/.</i>
and paste the token that you got in the previous step.

now you can see the kubernetes web UI

Also, if you want a better alternative, you want to test weave net. To do so, simply use:
```shell
kubectl apply -f "https://cloud.weave.works/k8s/scope.yaml?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```
then do a port-forward to access the weave.net service:
```bash
kubectl port-forward -n weave "$(kubectl get -n weave pod --selector=weave-scope-component=app -o jsonpath='{.items..metadata.name}')" 4040
```
and put in your browser:
```url
 https://localhost:4040
```
### Installing Helm
Helm is a package manager that helps to install applications on the kubernetes cluster.
It is installed on local (Download the desired version from [this site](https://github.com/helm/helm/releases) and you have to select the binary that is compatibe with your client.

## Configure Longhorn
Longhorn is a store service that help you to manage the storage of your cluster.
## Installing Longhorn
Once you installed helm in your local workstation, proceed to install longhorn:
```shell
kubectl create namespace longhorn-system
helm install longhorn longhorn/longhorn --namespace longhorn-system
```
to check if all longhorn deployment is installed run this command:
```shell
   kubectl -n longhorn-system get pods
   # and all longhorn pods must be in "Running" state.
```
let's create an ingress to the cluster service:
1. Create a basic auth file auth. It’s important the file generated is named auth (actually - that the secret has a key data.auth), otherwise the Ingress returns a 503.
```shell
   USER=<USERNAME_HERE>; PASSWORD=<PASSWORD_HERE>; echo "${USER}:$(openssl passwd -stdin -apr1 <<< ${PASSWORD})" >> auth
```
2. Create a secret:
```shell
   kubectl -n longhorn-system create secret generic basic-auth --from-file=auth
```
3. Create a file called ingress.yaml with this content:
```YAML
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: longhorn-ingress
  namespace: longhorn-system
  annotations:
    # type of authentication
    nginx.ingress.kubernetes.io/auth-type: basic
    # prevent the controller from redirecting (308) to HTTPS
    nginx.ingress.kubernetes.io/ssl-redirect: 'false'
    # name of the secret that contains the user/password definitions
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    # message to display with an appropriate context why the authentication is required
    nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required '
spec:
  rules:
  - http:
      paths:
      - path: /
        backend:
          serviceName: longhorn-frontend
          servicePort: 80
```
and 
## Testing the cluster
```shell
helm repo add bitnami https://charts.bitnami.com/bitnami
helm search repo bitnami
helm install my-release bitnami/[chart]
```
in our case we want to test the cluster with wordpress chart, so we type:
```shell
helm install my-release bitnami/wordpress
```
##Create ELK cluster:
to create an ELK cluster, simply type:
```shell
   helm install elasticsearch elastic/elasticsearch
```
and wait for the post are all in running state.
You must put port forwarding of the kibana service.
