### Download the weave script

Download the weave script with following commands:

```shell
$ sudo curl -L git.io/weave -o /usr/local/bin/weave
$ sudo chmod a+x /usr/local/bin/weave
```


### launch the weave network

To launch the weave network, simply run command:

```shell

$ sudo weave launch

```

Note: if your machine is running behind a firewall, please set the environment variable "http_proxy" and "https_proxy".

The above command will download the weave related docker image and start it.

### launch the weave cluster network

### create weave network from the docker

use the "docker network create" with "weave" driver, you can create weave network.

```shell
$ docker network create --driver weave net1
```

Note: to make the "docker network" series commands work, the docker daemon should be started with argument "--cluster-store".
### start containers on the created weave network

After the weave network is created with command "docker network create", we can start container like:
```shell

$ docker run --host=host1 --net=net1 -it busybox /bin/sh

```

the parameter "--net=net1" specifies that the created weave network is used， the hostname is host1.

Then we can start another container with name host2 on the net1

```shell
$ docker run --net=net1 --host=host2 busybox /bin/sh
```

In the first container (hostname is host1) console, we can ping the host2 like:

```shell
$ ping host2
```

From the second container (hostname is host2) console, we can ping the host1 like:

```shell
$ ping host1
```

the above "ping" command will show that the host1 and host2 are reachable each other.


