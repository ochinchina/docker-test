### install alpine linux to hard disk

#### install dbclient

if your alpine linux is installed with DHCP address fetch, install dhclient to update the "/etc/resolv.conf"

```shell
$ apk add dhclient
```

After rebooting the alpine linux, the /etc/resolv.conf will be updated by the "dhclient".

### install docker

append the following line to /etc/apk/repositories:

```text
http://dl-4.alpinelinux.org/alpine/edge/community
```

Then update & install the docker

```shell
$ apk update
$ apk add docker
```

start the docker:

```shell
$ service docker start
```

### set the PROXY for docker

if the alpine linux is installed behind the proxy, append following two lines to the /etc/config.d/docker file:

```text
export HTTP_PROXY=http://proxy-host:proxy-port/
export HTTPs_PROXY=http://proxy-host:proxy-port/
```

