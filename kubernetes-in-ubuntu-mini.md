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
ExecStart=/usr/bin/docker daemon -H unix:///var/run/docker_bootstrap.sock -p /var/run/docker_bootstrap.pid --iptables=false --ip-masq=false --bridge=none --graph=/var/lib/docker_bootstrap
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
