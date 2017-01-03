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

### create weave network from the docker

use the "docker network create" with "weave" driver, you can create weave network.

```shell
$ docker network create --driver weave net1
```

Note: to make the "docker network" series commands work, the docker daemon should be started with argument "--cluster-store".

