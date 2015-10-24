#test the kubernetes 1.0.1 with shoreman

##prepare

###1, download the kubernetes 1.0.1

The kubernetes 1.0.1 can be downloaded from https://github.com/kubernetes/kubernetes/releases/download/v1.0.1/kubernetes.tar.gz

```shell
$ wget https://github.com/kubernetes/kubernetes/releases/download/v1.0.1/kubernetes.tar.gz
$ tar -zxvf kubernetes.tar.gz
$ cd kubernetes/server
$ tar -zxvf kubernetes-server-linux-amd64.tar.gz
```


###2, download the etcd 2.2.1

```shell
$ wget https://github.com/coreos/etcd/releases/download/v2.2.1/etcd-v2.2.1-linux-amd64.tar.gz
$ tar xzvf etcd-v2.2.1-linux-amd64.tar.gz
$ cd etcd-v2.2.1-linux-amd64
```


###3, download and build the flannel 0.5.4

```shell
$ git clone https://github.com/coreos/flannel.git
$ cd flannel
$ ./build
```

###4, setup kubernetes master node with shoreman

find a machine or create a virtual machine as the master node

####a)create kubernetes-master directory on master node
create kubernetes-master directory in the master node

```shell
$ mkdir kubernetes-master
```

#####copy kubernetes files to master node
copy the following kubernetes binary to the master node kubernetes-master directory, these binaries are located at kubernetes/server/kubernetes/server/bin directory ( get at step 1)

```
kube-apiserver
kube-control-manager
kube-scheduler
```

#####copy etcd & ectdctl to master node
copy the etcd & etcdctl to the master node kubernetes-master directory, there binaries are located at etcd-v2.2.1-linux-amd64 direcoty ( get at step 2)

#####copy flanneld to master node

copy the flanneld from flannel/bin directory (down&build at step 3) to kubernetes-master

#####list files under kubernetes-master directory
After copying the above files to the kubernetes-master, list the files under kubernetes-master should like:

```
$ cd kubernetes-master
$ ls -l
-rwxr-x--- 1 root root 33256080 Oct 24 10:46 kube-controller-manager
-rwxr-x--- 1 root root 39872928 Oct 24 10:46 kube-apiserver
-rwxr-x--- 1 root root 20341304 Oct 24 11:16 kubectl
-rwxr-x--- 1 root root 17877376 Oct 24 10:46 kube-scheduler
-rwxr-xr-x 1 root root  8687648 Oct 24 10:47 etcd
-rwxr-xr-x 1 root root 12202592 Oct 24 10:47 etcdctl
-rwxr-xr-x 1 root root  8903712 Oct 24 10:47 flanneld
```

#####create a Profile for shoreman to start kubernetes master

go to the kubernetes-master directory in master node, create Procfile and .env for shoreman

```shell
$ cd kubernetes-master
$ touch Procfile
$ vi Procfile
$ cat Procfile
etcd: ./etcd -name etcd1 -initial-advertise-peer-urls http://$MASTER_IP:2380 -listen-peer-urls http://$MASTER_IP:2380 -listen-client-urls http://$MASTER_IP:2379,http://127.0.0.1:2379 -advertise-client-urls http://$MASTER_IP:2379 -initial-cluster-token etcd-cluster-1 -initial-cluster etcd1=http://$MASTER_IP:2380 -initial-cluster-state new

flannel: sleep 1; ./etcdctl --peers=http://127.0.0.1:2379 set /coreos.com/network/config '{"Network": "10.0.0.0/8", "SubnetLen": 20,  "SubnetMin": "10.10.0.0", "SubnetMax": "10.99.0.0","Backend": {"Type": "udp","Port": 7890}}'; ./flanneld -etcd-endpoints="http://127.0.0.1:2379"

kube-apiserver: sleep 2; ./kube-apiserver --address=0.0.0.0 --port=8080 --etcd_servers=http://$MASTER_IP:2379 --logtostderr=true --service-cluster-ip-range=10.1.0.0/16 --admission_control=NamespaceLifecycle,NamespaceExists,LimitRanger,ResourceQuota --service_account_key_file=/tmp/kube-serviceaccount.key --service_account_lookup=false

kube-controller-manager: sleep 4; ./kube-controller-manager --master=127.0.0.1:8080  --service_account_private_key_file=/tmp/kube-serviceaccount.key --logtostderr=true

kube-scheduler: sleep 4; ./kube-scheduler --master=127.0.0.1:8080 --logtostderr=true 


```

for first time start the kubernetes-master, /tmp/kube-serviceaccount.key should be created with following command:
```
$ openssl genrsa -out /tmp/kube-serviceaccount.key 2048 2
```

create & edit .env file under kubernetes-master, the content of this .env file should be:

```
MASTER_IP=<your master IP address>
```

Then start the kubernetes-master simply with shoreman

```shell
$ cd kubernetes-master
$ sudo shoreman
```
####5, setup kubernetes minions

find some node or virtual machine as kubernetes minions

##### create kubernetes-minion directory

```shell
$ mkdir kubernetes-minion
```

##### copy the following files to kubernetes-minions

```
flanneld
kubelet
kube-proxy
kubectl
```

#####create&edit Procfile under kubernetes-minion directory

The content of Procfile should be:

```
flannel: ./flanneld -etcd-endpoints="http://$MASTER_IP:2379"

kubelet: ./kubelet --address=0.0.0.0 --port=10250 --api_servers=http://$MASTER_IP:8080 --logtostderr=true

kube-proxy: ./kube-proxy --master=http://$MASTER_IP:8080 --logtostderr=true
```

#####create ".env" file under kubernetes-minion

the .env file under kubernetes-minion should include the MASTER_IP environment setting, its content looks like:

```
MASTER_IP=<your master IP address>
```
#####start the kubernetes-minion

```shell
$ cd kubernetes-minion
$ sudo shoreman
```

####6, create more kubernetes minion

repeat step 5 to create more kubernetes minions and start it

####7, test the kubernetes cluster with example


