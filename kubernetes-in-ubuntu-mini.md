###add proxy if running behind firewall

edit the file /lib/systemd/system/docker.service, add proxy setting before the ExecStart

```shell
[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
Environment="HTTP_PROXY=http://example.com:8080" "HTTPS_PROXY=https://example.com:8080" "NO_PROXY=localhost,127.0.0.1,10.246.1.0/24"
ExecStart=/usr/bin/docker daemon -H fd://
MountFlags=slave
LimitNOFILE=1048576
LimitNPROC=1048576
LimitCORE=infinity
TimeoutStartSec=0
# set delegate yes so that systemd does not reset the cgroups of docker containers
Delegate=yes
```

###start bootstrap docker daemon for flannel and etcd

create file /lib/systemd/system/docker-bootstrap.socket and put following contents to this file

```
[Unit]
Description=Docker Socket for the API
PartOf=docker-bootstrap.service

[Socket]
ListenStream=/var/run/docker-bootstrap.sock
SocketMode=0660
SocketUser=root
SocketGroup=docker

[Install]
WantedBy=sockets.target
```

create file /lib/systemd/system/docker-bootstrap.service and put following contents to this file

```
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network.target docker-bootstrap.socket
Requires=docker-bootstrap.socket

[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
Environment="HTTP_PROXY=http://example.com:8080" "HTTPS_PROXY=https://example.com:8080" "NO_PROXY=localhost,127.0.0.1,your_private_ip_network"
ExecStart=/usr/bin/docker daemon -H unix:///var/run/docker-bootstrap.sock -p /var/run/docker-bootstrap.pid --iptables=false --ip-masq=false --bridge=none --graph=/var/lib/docker-bootstrap --exec-root=/var/run/docker-bootstrap
MountFlags=slave
LimitNOFILE=1048576
LimitNPROC=1048576
LimitCORE=infinity
TimeoutStartSec=0
# set delegate yes so that systemd does not reset the cgroups of docker containers
Delegate=yes

[Install]
WantedBy=multi-user.target

```

start the docker-bootstrap service

```shell
$ sudo systemctl daemon-reload
$ sudo systemctl start docker-bootstrap
```
###setup etcd
```shell
$ sudo docker -H unix:///var/run/docker-bootstrap.sock pull gcr.io/google_containers/etcd-amd64:2.2.5
$ sudo docker -H unix:///var/run/docker-bootstrap.sock run \
--net=host \
-v /var/etcd/data:/var/etcd/data \
-d gcr.io/google_containers/etcd-amd64:2.2.5 \
/usr/local/bin/etcd \
--addr=127.0.0.1:4001 \
--bind-addr=0.0.0.0:4001 \
--data-dir=/var/etcd/data
$ sudo docker -H unix:///var/run/docker-bootstrap.sock ps
CONTAINER ID        IMAGE                                       COMMAND                  CREATED             STATUS              PORTS               NAMES
5f9ed5673bc2        gcr.io/google_containers/etcd-amd64:2.2.5   "/usr/local/bin/etcd "   8 seconds ago       Up 7 seconds                            reverent_jepsen

$  sudo docker -H unix:///var/run/docker-bootstrap.sock run --net=host gcr.io/google_containers/etcd-amd64:2.2.5 etcdctl set /coreos.com/network/config '{ "Network": "10.1.0.0/16" }'

```
###setup flannel

```shell
$ sudo docker -H unix:///var/run/docker-bootstrap.sock run -v /run/flannel:/run/flannel -d --net=host --privileged -v /dev/net:/dev/net quay.io/coreos/flannel:0.5.5 /opt/bin/flanneld [-iface=enp0s8]

$ sudo docker -H unix:///var/run/docker-bootstrap.sock exec <flannel_container_id> cat /run/flannel/subnet.env
```

###modify the docker options

add following line to the Service section in file /lib/systemd/system/docker.service

```
EnvironmentFile=/run/flannel/subnet.env
```

change the line in file /lib/systemd/system/docker.service from

```
ExecStart=/usr/bin/docker daemon -H fd://
```

to

```
ExecStart=/usr/bin/docker daemon -H fd:// --bip=${FLANNEL_SUBNET} --mtu=${FLANNEL_MTU}
```
After doing above modification, the file /lib/systemd/system/docker.service looks like:
```
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network.target docker.socket docker-bootstrap.service
Requires=docker.socket

[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
Environment="HTTP_PROXY=http://example.com:8080" "HTTPS_PROXY=https://example.com:8080" "NO_PROXY=localhost,127.0.0.1,your_private_network"
EnvironmentFile=/run/flannel/subnet.env
ExecStart=/usr/bin/docker daemon -H fd:// --bip=${FLANNEL_SUBNET} --mtu=${FLANNEL_MTU}
MountFlags=slave
LimitNOFILE=1048576
LimitNPROC=1048576
LimitCORE=infinity
TimeoutStartSec=0
# set delegate yes so that systemd does not reset the cgroups of docker containers
Delegate=yes

[Install]
WantedBy=multi-user.target

```

restart the docker

```shell
$ sudo systemctl daemon-reload
$ sudo systemctl restart docker
```
###setup k8s master

```
$ sudo docker run -d \
--net=host \
--pid=host \
--restart=always \
gcr.io/google_containers/hyperkube-amd64:v1.3.0 \
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
gcr.io/google_containers/hyperkube-amd64:v1.3.0 \
/hyperkube controller-manager \
--master=10.246.1.101:8080 \
--logtostderr=true


$ sudo docker run -d \
--net=host \
--pid=host \
--restart=always \
gcr.io/google_containers/hyperkube-amd64:v1.3.0 \
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
gcr.io/google_containers/hyperkube-amd64:v1.3.0 \
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
gcr.io/google_containers/hyperkube-amd64:v1.3.0 \
/hyperkube proxy \
--master=http://10.246.1.101:8080 \
--logtostderr=true
```

###install the kubectl

```shell
$ wget http://storage.googleapis.com/kubernetes-release/release/v1.3.0/bin/linux/amd64/kubectl
$ cp kubectl /usr/bin/kubectl  
$ chmod +x /usr/bin/kubectl
$ kubectl config set-cluster test-doc --server=http://10.246.1.101:8080
$ kubectl config set-context test-doc --cluster=test-doc
$ kubectl config use-context test-doc
$ kubectl get nodes
```

###External IPs(copied from http://kubernetes.io/docs/user-guide/services/#publishing-services---service-types)

If there are external IPs that route to one or more cluster nodes, Kubernetes services can be exposed on those externalIPs. Traffic that ingresses into the cluster with the external IP (as destination IP), on the service port, will be routed to one of the service endpoints. externalIPs are not managed by Kubernetes and are the responsibility of the cluster administrator.

In the ServiceSpec, externalIPs can be specified along with any of the ServiceTypes. In the example below, my-service can be accessed by clients on 80.11.12.10:80 (externalIP:port)
```shell
{
    "kind": "Service",
    "apiVersion": "v1",
    "metadata": {
        "name": "my-service"
    },
    "spec": {
        "selector": {
            "app": "MyApp"
        },
        "ports": [
            {
                "name": "http",
                "protocol": "TCP",
                "port": 80,
                "targetPort": 9376
            }
        ],
        "externalIPs" : [
            "80.11.12.10"
        ]
    }
}
```
