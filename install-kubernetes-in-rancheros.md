###setup etcd
```shell
$ sudo system-docker  run \
--net=host \
-v /var/etcd/data:/var/etcd/data \
-d gcr.io/google_containers/etcd-amd64:2.2.5 \
/usr/local/bin/etcd \
--addr=127.0.0.1:4001 \
--bind-addr=0.0.0.0:4001 \
--data-dir=/var/etcd/data

$  sudo system-docker run --net=host gcr.io/google_containers/etcd-amd64:2.2.5 etcdctl set /coreos.com/network/config '{ "Network": "10.1.0.0/16" }'

```
###setup flannel

```shell
$ sudo system-docker run -v /run/flannel:/run/flannel -d --net=host --privileged -v /dev/net:/dev/net quay.io/coreos/flannel:0.5.5 /opt/bin/flanneld [-iface=eth1]

```
###modify the docker options


```
$sudo ros set rancher.docker.args "['daemon','--log-opt','max-size=25m','--log-opt','max-file=2','-s','overlay','-G','docker','-H','unix:///var/run/docker.sock','--bip=10.1.30.1/24','--mtu=1472']"

```
###setup k8s master

```
$ sudo docker run -d \
--net=host \
--pid=host \
--restart=always \
gcr.io/google_containers/hyperkube-amd64:v1.2.6 \
/hyperkube apiserver \
--advertise-address=10.246.1.101 \
--insecure-bind-address=10.246.1.101 \
--insecure-port=8080 \
--etcd_servers=http://10.246.1.101:4001 \
--logtostderr=true \
--service-cluster-ip-range=192.168.0.0/16

$ sudo docker run -d \
--net=host \
--pid=host \
--restart=always \
gcr.io/google_containers/hyperkube-amd64:v1.2.6 \
/hyperkube controller-manager \
--master=10.246.1.101:8080 \
--logtostderr=true


$ sudo docker run -d \
--net=host \
--pid=host \
--restart=always \
gcr.io/google_containers/hyperkube-amd64:v1.2.6 \
/hyperkube scheduler \
--master=10.246.1.101:8080 \
--logtostderr=true
```
###set up k8s minion
```shell
sudo docker run -d \
--volume=/:/rootfs:ro \
--volume=/sys:/sys:rw \
--volume=/var/lib/docker/:/var/lib/docker:rw \
--volume=/var/lib/kubelet/:/var/lib/kubelet:rw \
--volume=/var/run:/var/run:rw \
--net=host \
--pid=host \
--privileged \
gcr.io/google_containers/hyperkube-amd64:v1.2.6 \
/hyperkube kubelet \
--address=0.0.0.0 --port=10250 \
--api_servers=http://10.246.1.101:8080 \
--logtostderr=true


sudo docker run -d \
--volume=/:/rootfs:ro \
--volume=/sys:/sys:rw \
--volume=/var/lib/docker/:/var/lib/docker:rw \
--volume=/var/lib/kubelet/:/var/lib/kubelet:rw \
--volume=/var/run:/var/run:rw \
--net=host \
--pid=host \
--privileged \
gcr.io/google_containers/hyperkube-amd64:v1.2.6 \
/hyperkube proxy \
--master=http://10.246.1.101:8080 \
--logtostderr=true
```

###install the kubectl

```shell
$ wget http://storage.googleapis.com/kubernetes-release/release/v1.2.6/bin/linux/amd64/kubectl
$ cp kubectl /usr/bin/kubectl  
$ chmod +x /usr/bin/kubectl
$ kubectl config set-cluster test-doc --server=http://10.246.1.101:8080
$ kubectl config set-context test-doc --cluster=test-doc
$ kubectl config use-context test-doc
$ kubectl get nodes
```

