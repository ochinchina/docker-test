## start dockerd with --cluster-store

To create network with "docker network create" command, the docker daemon should be started with "--cluster-store" option. After adding the --cluster-store, the docker daemon start command looks like (the etcd is used as the cluster store):

```shell

/usr/bin/dockerd -H fd:// --cluster-store=etcd://localhost:2379

```

## install & start etcd

The etcd can be started in a docker container or be started by systemctl, for how to install & start the etcd cluster, please refer to the etcd documents.

## install calico

### download calico

The calico binary can be get with following commands
```shell
sudo wget -O /usr/local/bin/calicoctl https://github.com/projectcalico/calico-containers/releases/download/v1.0.0/calicoctl
sudo chmod +x calicoctl
```

### configure calicoctl.cfg

Edit file /etc/calico/calicoctl.cfg, the calicoctl.cfg file looks like:

```yml
apiVersion: v1
kind: calicoApiConfig
metadata:
spec:
  datastoreType: "etcdv2"
  etcdEndpoints: "http://etcd1:2379,http://etcd2:2379"
```
for how to configuew file calicoctl.cfg, please refer to the calico document

### start the calico

After the calico is downloaded and the configuration file /etc/calico/calicoctl.cfg is ready, the calico can be started by:

```shell
sudo calicoctl node run
```
The above command will trigger the docker daemon to dowload the "calico/node:v1.0.0" image from the docker registry and start it. This may takes several minutes depends on your network environment.

## create calico network
After the calico container with image "calico/node:v1.0.0" is started and the docker daemon is started with "--cluster-store" correctly, we can run "docker network create" command to create calico network with following:

```shell
$ docker network create --driver calico --ipam-driver calico-ipam net1
$ docker network create --driver calico --ipam-driver calico-ipam net2
```
The above commands created two calico network.

## start container on the calico network

```shell
$ docker run --net net1 --name workload-A -tid busybox
$ docker run --net net2 --name workload-B -tid busybox
$ docker run --net net1 --name workload-C -tid busybox
```
## checking the reachability

after the above containers are started, we can ping if these containers can ping each other.

```
$ docker exec workload-A ping workload-C
PING workload-C (192.168.146.129): 56 data bytes
64 bytes from 192.168.146.129: seq=0 ttl=63 time=0.106 ms
64 bytes from 192.168.146.129: seq=1 ttl=63 time=0.134 ms
$ docker exec workload-A ping workload-B
ping: bad address 'workload-B'
$ docker exec workload-A ping -c 2  `docker inspect --format "{{ .NetworkSettings.Networks.net2.IPAddress }}" workload-B`
PING 192.168.146.130 (192.168.146.130): 56 data bytes

--- 192.168.146.130 ping statistics ---
2 packets transmitted, 0 packets received, 100% packet loss
```
So for any two cotainers, if they are in same network, they can ping each other, otherwise they can't ping each other.





