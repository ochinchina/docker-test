####install centos atomic
####install docker&kubernetes
####configure etcd
####configure flanneld
####start docker
remove ip address
```shell

$ ip a del (ip a docker0 | xargs grep inet | awk '{print $1}') dev docker0
```
####configure kubernetes master
####configure kubernetes minions
