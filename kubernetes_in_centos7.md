## install centos

get the centos atomic from the http://cloud.centos.org/centos/7/atomic/images/CentOS-Atomic-Host-7-Installer.iso, install this .iso to your machine

### configure the network

if the DHCP is used but no network address is got, edit the file /etc/sysconfig/network-scripts/ifcfg-enp0s3 and change the item "ONBOOT" from "no" to "yes"

### configure proxy for atomic upgrade

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
### more space for root directory

edit the file /etc/sysconfig/docker-storage-setup file and add the ROOT_SIZE to this file

```shell

$ cat /etc/sysconfig/docker-storage-setup | grep ROOT_SIZE
ROOT_SIZE=6G

```

### proxy setting for docker

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

## install docker&kubernetes


## configure and start etcd

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

### set the network configuration for kubernetes cluster nodes

From one etcd node, use etcdctl command to set the /coreos.com/network/config

```
$ etcdctl set /coreos.com/network/config '{"Network": "10.0.0.0/8", "SubnetLen": 20,  "SubnetMin": "10.10.0.0", "SubnetMax": "10.99.0.0","Backend": {"Type": "udp","Port": 7890}}'
```

### configure and start flanneld on all the kubernetes cluster nodes

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


### start docker

remove the ip address of bridge 'docker0' if a ip address is assigned to docker0 before

```shell

$ sudo ip a del $(ip a show docker0 |  grep inet | awk '{print $2}') dev docker0
```

Start the docker on all nodes in the cluster

```
$ systemctl start docker
```

### configure kubernetes master

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

### configure kubernetes minions

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

### check the nodes managed by master

```shell
$ cubectl get nodes
NAME           LABELS                                STATUS
10.245.1.102   kubernetes.io/hostname=10.245.1.102   Ready
10.245.1.103   kubernetes.io/hostname=10.245.1.103   Ready
```

### skyDNS deployment

skydns-rc.yml:
```yaml
# Copyright 2016 The Kubernetes Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# TODO - At some point, we need to rename all skydns-*.yaml.* files to kubedns-*.yaml.*
# Should keep target in cluster/addons/dns-horizontal-autoscaler/dns-horizontal-autoscaler.yaml
# in sync with this file.

# __MACHINE_GENERATED_WARNING__

apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: kube-dns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
spec:
  replicas: 1
  # 1. In order to make Addon Manager do not reconcile this replicas parameter.
  # 2. Default is 1.
  # 3. Will be tuned in real time if DNS horizontal auto-scaling is turned on.
  strategy:
    rollingUpdate:
      maxSurge: 10%
      maxUnavailable: 0
  selector:
    matchLabels:
      k8s-app: kube-dns
  template:
    metadata:
      labels:
        k8s-app: kube-dns
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ''
        scheduler.alpha.kubernetes.io/tolerations: '[{"key":"CriticalAddonsOnly", "operator":"Exists"}]'
    spec:
      containers:
      - name: kubedns
        image: gcr.io/google_containers/kubedns-amd64:1.9
        resources:
          # TODO: Set memory limits when we've profiled the container for large
          # clusters, then set request = limit to keep this container in
          # guaranteed class. Currently, this container falls into the
          # "burstable" category so the kubelet doesn't backoff from restarting it.
          limits:
            memory: 170Mi
          requests:
            cpu: 100m
            memory: 70Mi
        livenessProbe:
          httpGet:
            path: /healthz-kubedns
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 60
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 5
        readinessProbe:
          httpGet:
            path: /readiness
            port: 8081
            scheme: HTTP
          # we poll on pod startup for the Kubernetes master service and
          # only setup the /readiness HTTP server once that's available.
          initialDelaySeconds: 3
          timeoutSeconds: 5
        args:
        - --domain=cluster.local.
        - --dns-port=10053
        - --config-map=kube-dns
        - --kube-master-url=http://192.168.122.20:8080
        # This should be set to v=2 only after the new image (cut from 1.5) has
        # been released, otherwise we will flood the logs.
        - --v=0
        #__PILLAR__FEDERATIONS__DOMAIN__MAP__
        env:
        - name: PROMETHEUS_PORT
          value: "10055"
        ports:
        - containerPort: 10053
          name: dns-local
          protocol: UDP
        - containerPort: 10053
          name: dns-tcp-local
          protocol: TCP
        - containerPort: 10055
          name: metrics
          protocol: TCP
      - name: dnsmasq
        image: gcr.io/google_containers/k8s-dns-dnsmasq-amd64:1.14.5
        livenessProbe:
          httpGet:
            path: /healthz-dnsmasq
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 60
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 5
        args:
        - --cache-size=1000
        - --no-resolv
        - --server=127.0.0.1#10053
        #- --log-facility=-
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
        # see: https://github.com/kubernetes/kubernetes/issues/29055 for details
        resources:
          requests:
            cpu: 150m
            memory: 10Mi
      - name: dnsmasq-metrics
        image: gcr.io/google_containers/dnsmasq-metrics-amd64:1.0.1
        livenessProbe:
          httpGet:
            path: /metrics
            port: 10054
            scheme: HTTP
          initialDelaySeconds: 60
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 5
        args:
        - --v=2
        - --logtostderr
        ports:
        - containerPort: 10054
          name: metrics
          protocol: TCP
        resources:
          requests:
            memory: 10Mi
      - name: healthz
        image: gcr.io/google_containers/exechealthz-amd64:1.2
        resources:
          limits:
            memory: 50Mi
          requests:
            cpu: 10m
            # Note that this container shouldn't really need 50Mi of memory. The
            # limits are set higher than expected pending investigation on #29688.
            # The extra memory was stolen from the kubedns container to keep the
            # net memory requested by the pod constant.
            memory: 50Mi
        args:
        - --cmd=nslookup kubernetes.default.svc.cluster.local 127.0.0.1 >/dev/null
        - --url=/healthz-dnsmasq
        - --cmd=nslookup kubernetes.default.svc.cluster.local 127.0.0.1:10053 >/dev/null
        - --url=/healthz-kubedns
        - --port=8080
        - --quiet
        ports:
        - containerPort: 8080
          protocol: TCP
      dnsPolicy: Default # Don't use cluster DNS.

```
skydns-service.yaml

```yaml
# Copyright 2016 The Kubernetes Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# TODO - At some point, we need to rename all skydns-*.yaml.* files to kubedns-*.yaml.*

# Warning: This is a file generated from the base underscore template file: skydns-svc.yaml.base

apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: "KubeDNS"
spec:
  selector:
    k8s-app: kube-dns
  clusterIP: 10.254.0.100
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP

```

```shell
$ kubectl create -f skydns-rc.yaml
$ kubectl create -f skydns-service.yaml
```
