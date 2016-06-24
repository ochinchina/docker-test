####install centos atomic

get the centos atomic from the http://cloud.centos.org/centos/7/atomic/images/CentOS-Atomic-Host-7-Installer.iso, install this .iso to your machine

#####configure the network

if the DHCP is used but no network address is got, edit the file /etc/sysconfig/network-scripts/ifcfg-enp0s3 and change the item "ONBOOT" from "no" to "yes"

#####configure proxy for atomic upgrade

if your atomic is installed behind proxy server, before upgrading the atomic host, you need to add proxy in the file /etc/ostree/remotes.d/centos-atomic-host.conf

```shell
$ cd /etc/ostree/remotes.d
$ cat centos-atomic-host.conf
[remote "centos-atomic-host"]
url=http://mirror.centos.org/centos/7/atomic/x86_64/rep
branches=centos-atomic-host/7/x86_64/standard;
gpg-verify=true
proxy=http://example.com:8080
```
#####more space for root directory

edit the file /etc/sysconfig/docker-storage-setup file and add the ROOT_SIZE to this file

```shell

$ cat /etc/sysconfig/docker-storage-setup | grep ROOT_SIZE
ROOT_SIZE=6G

```

#####proxy setting for docker

if docker runs behind proxy, the file /etc/sysconfig/docker should be edit and following lines should be append to the end of the file /etc/sysconfig/docker:

```
HTTP_PROXY=http://example.com:8080
HTTPS_PROXY=https://example.com:8080
```

and then restart the docker:

```shell
$ sudo vi /etc/sysconfig/docker
$ sudo systemctl restart docker
```

####install docker&kubernetes


####configure and start etcd

assume there are three nodes for etcd cluster, they are 10.245.1.101, 10.245.1.102 and 10.245.1.103.

So in the node 10.245.1.101, the /etc/etcd/etcd.conf should be:
```
ETCD_NAME=infra0
ETCD_DATA_DIR=/var/lib/etcd/default.etcd
ETCD_LISTEN_PEER_URLS=http://10.245.1.101:2380
ETCD_LISTEN_CLIENT_URLS=http://localhost:2379,http://10.245.1.101:2379
ETCD_INITIAL_ADVERTISE_PEER_URLS=http://10.245.1.101:2380
ETCD_INITIAL_CLUSTER=infra0=http://10.245.1.101:2380,infra1=http://10.245.1.102:2380,infra2=http://10.245.1.103:2380
ETCD_INITIAL_CLUSTER_STATE=new
ETCD_INITIAL_CLUSTER_TOKEN=etcd-cluster
ETCD_ADVERTISE_CLIENT_URLS=http://10.245.1.101:2379
```
in the node 10.245.1.102, the /etc/etcd/etcd.conf should be:
```
ETCD_NAME=infra1
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="http://10.245.1.102:2380"
ETCD_LISTEN_CLIENT_URLS="http://localhost:2379,http://10.245.1.102:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.245.1.102:2380"
ETCD_INITIAL_CLUSTER="infra0=http://10.245.1.101:2380,infra1=http://10.245.1.102:2380,infra2=http://10.245.1.103:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_ADVERTISE_CLIENT_URLS="http://10.245.1.102:2379"
```

in the node 10.245.1.103, the /etc/etcd/etcd.conf should be:

```
ETCD_NAME=infra2
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="http://10.245.1.103:2380"
ETCD_LISTEN_CLIENT_URLS="http://10.245.1.103:2379,http://localhost:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.245.1.103:2380"
ETCD_INITIAL_CLUSTER="infra0=http://10.245.1.101:2380,infra1=http://10.245.1.102:2380,infra2=http://10.245.1.103:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_ADVERTISE_CLIENT_URLS="http://10.245.1.103:2379"
```

After all the aboving configuration are done, start the etcd on each node

```shell
$ sudo systemctl start etcd
```

####set the network configuration for kubernetes cluster nodes

From one etcd node, use etcdctl command to set the /coreos.com/network/config

```
$ etcdctl set /coreos.com/network/config '{"Network": "10.0.0.0/8", "SubnetLen": 20,  "SubnetMin": "10.10.0.0", "SubnetMax": "10.99.0.0","Backend": {"Type": "udp","Port": 7890}}'
```

####configure and start flanneld on all the kubernetes cluster nodes

edit file /etc/sysconfig/flanneld on all the kubernetes cluster nodes.

The parameter FLANNEL_ETCD in /etc/sysconfig/flanneld file  should be:

```
FLANNEL_ETCD="http://10.245.1.101:2379,http://10.245.1.102:2379,http://10.245.1.103:2379"
```

If multiple network cards are available in your centos, you'd better bind the flannel to a specific network interface:

```
FLANNEL_OPTIONS="-iface=enp0s8"
```
Then start the flanneld on all the nodes in the cluster using systemctl

```shell
$ systemctl start flanneld
```


####start docker

remove the ip address of bridge 'docker0' if a ip address is assigned to docker0 before

```shell

$ sudo ip a del $(ip a show docker0 |  grep inet | awk '{print $2}') dev docker0
```

Start the docker on all nodes in the cluster

```
$ systemctl start docker
```

####configure kubernetes master

Choose one node for running kubernetes master, in this example, the 10.245.1.101 is choosed as the kubernetes master

change KUBE_API_ADDRESS, KUBE_API_PORT, KUBE_ETCD_SERVERS, KUBE_SERVICE_ADDRESSES and KUBE_SERVICE_ADDRESSES parameters in /etc/kubernetes/apiserver

```
KUBE_API_ADDRESS="--address=0.0.0.0"
KUBE_API_PORT="--port=8080"
KUBE_ETCD_SERVERS="--etcd_servers=http://10.245.1.101:2379,http://10.245.1.102:2379,http://10.245.1.103:2379"
//the KUBE_SERVICE_ADDRESSES should be in network 10.0.0.0/8 but should be not in range 10.10.0.0 to 10.99.0.0
//because this range is already allocated for flannel use
KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.254.0.0/16"
KUBE_ADMISSION_CONTROL="--admission_control=NamespaceLifecycle,NamespaceExists,LimitRanger,SecurityContextDeny,ResourceQuota"
```

Then start the kube-apiserver, kube-controller-manager and kube-scheduler

```shell
$ sudo systemctl start kube-apiserver
$ sudo systemctl start kube-controller-manager
$ sudo systemctl start kube-scheduler
```

####configure kubernetes minions

Edit the /etc/kubernetes/kubelet and change the KUBELET_ADDRESS, KUBELET_HOSTNAME, and KUBELET_API_SERVER

An example of minion on node 10.245.1.102 is
```
KUBELET_ADDRESS="--address=0.0.0.0"
KUBELET_HOSTNAME="--hostname_override=10.245.1.102"
KUBELET_API_SERVER="--api_servers=http://10.245.1.101:8080"
```

start the kubelet and the kube-proxy on all minion nodes

```shell
$ sudo systemctl start kubelet
$ sudo systemctl start kube-proxy
```

####check the nodes managed by master

```shell
$ cubectl get nodes
NAME           LABELS                                STATUS
10.245.1.102   kubernetes.io/hostname=10.245.1.102   Ready
10.245.1.103   kubernetes.io/hostname=10.245.1.103   Ready
```
