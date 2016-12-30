## start dockerd with --cluster-store

/usr/bin/dockerd -H fd:// --cluster-store=etcd://localhost:2381

## install & start etcd
## install calico

### download calico

sudo wget -O /usr/local/bin/calicoctl https://github.com/projectcalico/calico-containers/releases/download/v1.0.0/calicoctl
sudo chmod +x calicoctl

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

## create calico network

```shell
$ docker network create --driver calico --ipam-driver calico-ipam net1
$ docker network create --driver calico --ipam-driver calico-ipam net2
```

## start container on the calico network
```shell
$ docker run --net net1 --name workload-A -tid busybox
$ docker run --net net2 --name workload-B -tid busybox
$ docker run --net net1 --name workload-C -tid busybox
```



