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
KUBE_API_ADDRESS="--address=10.245.1.101"
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
KUBELET_ADDRESS="--address=10.245.1.102"
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

### Kubernetes master HA

There are lots of document describe how to setup multiple kubernetes master for HA purpose. For the offical setup, please refer the document: https://kubernetes.io/docs/admin/high-availability.

For small-size k8s cluster, if there is no enough node to create a separate k8s master cluster, we can setup a proxy on all nodes with following method:

#### start-up multiple k8s master

##### Change KUBE_API_ADDRESS in /etc/kubernetes/apiserver file in master nodes

On each k8s master nodes, change the parameter KUBE_API_ADDRESS:

On node 10.245.1.101, change KUBE_API_ADDRESS to:
```shell
KUBE_API_ADDRESS="--insecure-bind-address=10.245.1.101"
```

On node 10.245.1.102, change the KUBE_API_ADDRESS to:
```shell
KUBE_API_ADDRESS="--insecure-bind-address=10.245.1.102"
```

##### Change KUBE_CONTROLLER_MANAGER_ARGS in /etc/kubernetes/controller-manager in master nodes

For each node that running the k8s master, change the KUBE_CONTROLLER_MANAGER_ARGS to:

```shell
KUBE_CONTROLLER_MANAGER_ARGS="--leader-elect=true"
```

##### Change KUBE_MASTER in /etc/kubernetes/config in all nodes

For all the nodes, including the master nodes and missions nodes, change the KUBE_MASTER in /etc/kubernetes/config as:

```shell
KUBE_MASTER="--master=http://localhost:8080"
```

##### change KUBELET_ARGS in /etc/kubernetes/kubelet in all nodes

For all the nodes, including the master nodes and missions nodes, change the KUBELET_ARGS in /etc/kubernetes/kubelet in all nodes as:

```shell
KUBELET_ARGS="--cluster_dns=10.254.0.100 --cluster_domain=cluster.local --pod-manifest-path=/etc/kubelet.d/"
```

##### put nginx.conf to /etc/nginx

put the nginx.conf with following contents to /etc/nginx:

```shell
events {
        worker_connections  1024;
}

http {
  upstream backend_hosts {
    server 192.168.122.20:8080;
    server 192.168.122.63:8080;
  }

  server {
    listen 127.0.0.1:8080;
    server_name 127.0.0.1;

    location / {
        proxy_pass http://backend_hosts;
    }
  }
}

```

##### put the following static-pods configure /etc/kubelet.d/ to all the nodes:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: k8s-master-proxy
spec:
  hostNetwork: true
  containers:
    - name: k8s-master-proxy
      image: nginx:1.13.7-alpine
      volumeMounts:
        - name: nginx-cfg
          mountPath: /etc/nginx
  volumes:
    - name: nginx-cfg
      hostPath:
        path: /etc/nginx
```


#### start a proxy for the k8s master

After all the configurations are ready, we can start-up the master and kubelet:

On master node, for example, 10.245.1.101 and 10.245.1.102, start the kube-apiserver, kube-controller-manager and kube-scheduler as below:

```shell
$ sudo systemctl start kube-apiserver
$ sudo systemctl start kube-controllere-manager
$ sudo systemctl start kube-scheduler
```

and on all the nodes to start the kube-proxy and the kubelet

```shell
$ sudo systemctl start kubelet
$ sudo systemctl start kube-proxy
```

After all the nodes startup, login to any one of node to check the k8s node status:

```shell
$ kubectl get node
NAME           LABELS                                STATUS
10.245.1.102   kubernetes.io/hostname=10.245.1.102   Ready
10.245.1.103   kubernetes.io/hostname=10.245.1.103   Ready
```

Then we can deploy applications to the k8s cluster.

### skyDNS deployment

the skydns-rc.yml and skydns-svc.yml are from template in https://github.com/kubernetes/kubernetes/tree/v1.5.8/cluster/addons/dns

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
        - --kube-master-url=http://10.245.1.101:8080
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
skydns-svc.yml

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
