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

###setup kubernetes master node with shoreman

find a machine as the master node and create kubernetes-master directory in the master node

```shell
$ mkdir kubernetes-master
```

copy the following kubernetes binary to the master node kubernetes-master directory, these binaries are located at kubernetes/server/kubernetes/server/bin directory ( get at step 1)

```
kube-apiserver
kube-control-manager
kube-scheduler
```
copy the etcd & etcdctl to the master node kubernetes-master directory, there binaries are located at etcd-v2.2.1-linux-amd64 direcoty ( get at step 2)

copy the flanneld from flannel/bin directory (down&build at step 3)

After copying the above files to the kubernetes-master, list the files under kubernetes-master should like:

```
$ cd kubernetes-master
$ ls -l
-rwxr-x--- 1 root root 33256080 Oct 24 10:46 kube-controller-manager
-rwxr-x--- 1 root root 39872928 Oct 24 10:46 kube-apiserver
-rwxr-x--- 1 root root 17877376 Oct 24 10:46 kube-scheduler
-rwxr-xr-x 1 root root  8687648 Oct 24 10:47 etcd
-rwxr-xr-x 1 root root 12202592 Oct 24 10:47 etcdctl
-rwxr-xr-x 1 root root  8903712 Oct 24 10:47 flanneld
```




