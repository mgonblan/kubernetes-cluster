## The easy way: Rancher
Rancher is a docker solution that make easyer the cluster implementation.
Deploying a cluster with rancher is easy, simply following these steps:
- First edit /etc/hosts with private ip of cluster node's.
  ```/etc/hosts
     127.0.0.1 localhost
     # (New) Add these names
     [master-internal-ip]    [master-hostname]
     [node-0-internal-ip]    [node-0-hostname]
     [access-external-ip]    [access-domain-name] [access-hostname]
     ...
     # (new-end)
  ```
- Then install rke on this way:
  run Rancher-master.sh in master and Rancher-worker.sh in the worker node, these scripts will configure the servers to work with rancher and configure a one-node kubernetes cluster.
  