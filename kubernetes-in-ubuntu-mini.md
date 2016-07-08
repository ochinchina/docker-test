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

create file /lib/systemd/system/docker_bootstrap.socket and put following contents to this file

```
[Unit]
Description=Docker Socket for the API
PartOf=docker_bootstrap.service

[Socket]
ListenStream=/var/run/docker_bootstrap.sock
SocketMode=0660
SocketUser=root
SocketGroup=docker

[Install]
WantedBy=sockets.target
```

create file /lib/systemd/system/docker_bootstrap.service and put following contents to this file

```
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network.target docker_bootstrap.socket
Requires=docker_bootstrap.socket

[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
Environment="HTTP_PROXY=http://example.com:8080" "HTTPS_PROXY=https://example.com:8080" "NO_PROXY=localhost,127.0.0.1,your_private_ip_network"
ExecStart=/usr/bin/docker daemon -H unix:///var/run/docker_bootstrap.sock -p /var/run/docker_bootstrap.pid --iptables=false --ip-masq=false --bridge=none --graph=/var/lib/docker_bootstrap --exec-root=/var/run/docker_bootstrap
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

start the docker_bootstrap service

```shell
$ sudo systemctl daemon-reload
$ sudo systemctl start docker_bootstrap
```
###setup etcd
```shell
$ sudo docker -H unix:///var/run/docker_bootstrap.sock pull gcr.io/google_containers/etcd-amd64:2.2.5
$ sudo docker -H unix:///var/run/docker_bootstrap.sock run \
--net=host \
-v /var/etcd/data:/var/etcd/data \
-d gcr.io/google_containers/etcd-amd64:2.2.5 \
/usr/local/bin/etcd \
--addr=127.0.0.1:4001 \
--bind-addr=0.0.0.0:4001 \
--data-dir=/var/etcd/data
$ sudo docker -H unix:///var/run/docker_bootstrap.sock ps
CONTAINER ID        IMAGE                                       COMMAND                  CREATED             STATUS              PORTS               NAMES
5f9ed5673bc2        gcr.io/google_containers/etcd-amd64:2.2.5   "/usr/local/bin/etcd "   8 seconds ago       Up 7 seconds                            reverent_jepsen

$  sudo docker -H unix:///var/run/docker_bootstrap.sock run --net=host gcr.io/google_containers/etcd-amd64:2.2.5 etcdctl set /coreos.com/network/config '{ "Network": "10.1.0.0/16" }'

```
###setup flannel

```shell
$ sudo docker -H unix:///var/run/docker_bootstrap.sock run -v /run/flannel:/run/flannel -d --net=host --privileged -v /dev/net:/dev/net quay.io/coreos/flannel:0.5.5

$ sudo docker -H unix:///var/run/docker_bootstrap.sock exec <flannel_container_id> cat /run/flannel/subnet.env
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
After=network.target docker.socket docker_bootstrap.service
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

```shell
$ docker run -d \
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
--containerized \
--hostname-override=127.0.0.1 \
--api-servers=http://localhost:8080 \
--config=/etc/kubernetes/manifests \
--cluster-dns=10.0.0.10 \
--cluster-domain=cluster.local \
--allow-privileged --v=2
```
set the kubectl

```shell
$ wget http://storage.googleapis.com/kubernetes-release/release/v1.3.0/bin/linux/amd64/kubectl
$ cp kubectl /usr/bin/kubectl  
$ chmod +x /usr/bin/kubectl
$ kubectl config set-cluster test-doc --server=http://localhost:8080
$ kubectl config set-context test-doc --cluster=test-doc
$ kubectl config use-context test-doc
$ kubectl get nodes
```

