### download minikube
Go yo page https://github.com/kubernetes/minikube/releases to get the latest minikube. For example, to get the latest v0.14.0, in the linux the following commands can be executed to get the minikube:

```shell
$ curl -o /usr/local/bin/minikube https://storage.googleapis.com/minikube/releases/v0.14.0/minikube-linux-amd64
$ chmod a+x /usr/local/bin/minikube
```

### start the minikube

simply run the command "minikube start"

```shell
$ minikube start
```
the minikube will start to download OS, the kubernetes from network and start it.

### start minikube with proxy

If your machine is behind the http_proxy, the minikube should be srated with following command:
```shell
$ minikube start --docker-env HTTP_PROXY=http://$YOURPROXY:PORT \
                 --docker-env HTTPS_PROXY=https://$YOURPROXY:PORT
```

### install kubectl

### start a service

```shell
$ kubectl run hello-minikube --image=gcr.io/google_containers/echoserver:1.4 --port=8080
```

### ssh into the VM

if you want to login to the minikube VM, call the following command:

```shell
$ minikube ssh
```

After login to the minikube VM, you can run any command in the minikube.

### stop minikube

To stop the minikube VM ( not delete the VM), the following command should be called:

```shell

$ minikube stop

```

### delete the started minikube

to delete the created minikube VM, run the following commands:

```shell
$ minikube delete
$ rm -rf ~/.minikube/
```

The "rm -rf ~/.minikube" command must be executed in order to restart a new minikube VM.
