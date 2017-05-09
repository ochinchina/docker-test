### start the nginx-ingress-controller

in k8s 1.6.1, if create the nginx-ingress-controller with file https://rawgit.com/kubernetes/ingress/master/examples/deployment/nginx/kubeadm/nginx-ingress-controller.yaml like:

```shell
$ kubectl apply -f https://rawgit.com/kubernetes/ingress/master/examples/deployment/nginx/kubeadm/nginx-ingress-controller.yaml
```

the nginx ingress controller can't find the kube-system/default-http-backend becauses of user account information, an issue is reported in https://github.com/kubernetes/ingress/issues/575. 

The https://github.com/kubernetes/ingress/issues/575 gives an solution (create correct user account before creating the nginx-ingress-controller), its contents is pasted below:

```yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: ingress
rules:
- apiGroups:
  - ""
  - "extensions"
  resources:
  - configmaps
  - secrets
  - services
  - endpoints
  - ingresses
  - nodes
  - pods
  verbs:
  - list
  - watch
- apiGroups:
  - "extensions"
  resources:
  - ingresses
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - events
  - services
  verbs:
  - create
  - list
  - update
  - get
- apiGroups:
  - "extensions"
  resources:
  - ingresses/status
  - ingresses
  verbs:
  - update
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: Role
metadata:
  name: ingress-ns
  namespace: kube-system
rules:
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - list
- apiGroups:
  - ""
  resources:
  - services
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - endpoints
  verbs:
  - get
  - create
  - update  
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: ingress-ns-binding
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: ingress-ns
subjects:
  - kind: ServiceAccount
    name: ingress
    namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: ingress-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: ingress
subjects:
  - kind: ServiceAccount
    name: ingress
    namespace: kube-system
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: default-http-backend
  labels:
    k8s-app: default-http-backend
  namespace: kube-system
spec:
  replicas: 1
  template:
    metadata:
      labels:
        k8s-app: default-http-backend
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - name: default-http-backend
        # Any image is permissable as long as:
        # 1. It serves a 404 page at /
        # 2. It serves 200 on a /healthz endpoint
        image: gcr.io/google_containers/defaultbackend:1.0
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 30
          timeoutSeconds: 5
        ports:
        - containerPort: 8080
        resources:
          limits:
            cpu: 10m
            memory: 20Mi
          requests:
            cpu: 10m
            memory: 20Mi
---
apiVersion: v1
kind: Service
metadata:
  name: default-http-backend
  namespace: kube-system
  labels:
    k8s-app: default-http-backend
spec:
  ports:
  - port: 80
    targetPort: 8080
  selector:
    k8s-app: default-http-backend
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ingress
  namespace: kube-system
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  labels:
    k8s-app: nginx-ingress-controller
  namespace: kube-system
spec:
  replicas: 1
  template:
    metadata:
      labels:
        k8s-app: nginx-ingress-controller
    spec:
      # hostNetwork makes it possible to use ipv6 and to preserve the source IP correctly regardless of docker configuration
      # however, it is not a hard dependency of the nginx-ingress-controller itself and it may cause issues if port 10254 already is taken on the host
      # that said, since hostPort is broken on CNI (https://github.com/kubernetes/kubernetes/issues/31307) we have to use hostNetwork where CNI is used
      # like with kubeadm
      hostNetwork: true
      terminationGracePeriodSeconds: 60
      serviceAccountName: ingress
      containers:
      - image: gcr.io/google_containers/nginx-ingress-controller:0.9.0-beta.5
        name: nginx-ingress-controller
        readinessProbe:
          httpGet:
            path: /healthz
            port: 10254
            scheme: HTTP
        livenessProbe:
          httpGet:
            path: /healthz
            port: 10254
            scheme: HTTP
          initialDelaySeconds: 10
          timeoutSeconds: 1
        ports:
        - containerPort: 80
          hostPort: 80
        - containerPort: 443
          hostPort: 443
        env:
          - name: POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: POD_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
        args:
        - /nginx-ingress-controller
        - --default-backend-service=$(POD_NAMESPACE)/default-http-backend
```

The image is changed from gcr.io/google_containers/nginx-ingress-controller:0.9.0-beta.3 to gcr.io/google_containers/nginx-ingress-controller:0.9.0-beta.5.

copy above content to file nginx-ingress-controller.yml file, then execute:

```shell
$ kubectl apply -f nginx-ingress-controller.yml
```

### create ingress resource

create a file named test-ingress.yml with following contents:

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test-ingress
spec:
  rules:
  - host: "test.io"
    http:
      paths:
      - path: /hello
        backend:
          serviceName: hello
          servicePort: 8080
```

Then create the ingress:

```shell
$ kubectl create -f test-ingress.yml
$ kubectl get ing
NAME           HOSTS     ADDRESS   PORTS     AGE
test-ingress   test.io             80        47m
$ kubectl describe ing test-ingress
Name:                   test-ingress
Namespace:              default
Address:
Default backend:        default-http-backend:80 (10.10.4.10:8080)
Rules:
  Host          Path    Backends
  ----          ----    --------
  test.io
                /hello  hello:8080 (<none>)
Annotations:
Events:
  FirstSeen     LastSeen        Count   From                    SubObjectPath   Type            Reason  Message
  ---------     --------        -----   ----                    -------------   --------        ------  -------
  47m           47m             1       ingress-controller                      Normal          CREATE  Ingress default/test-ingress
  47m           47m             1       ingress-controller                      Normal          UPDATE  Ingress default/test-ingress

```

### create service


```shell
$ kubectl run hello --image=gcr.io/google_containers/echoserver:1.4 --port=8080
$ kubectl expose deployment hello
```

### check the /etc/nginx/nginx.conf file

```shell
$ kubectl get pod --namespace=kube-system
$ kubectl exec --namespace=kube-system nginx-ingress-controller-3023662530-g7b5m -- cat /etc/nginx/nginx.conf

http {
   upstream default-hello-8080 {
      least_conn;
      server 10.10.1.57:8080 max_fails=0 fail_timeout=0;
   }
   
   server {
      server_name test.io;
      listen 80;
      listen [::]:80;
      
      location /hello {
         set $proxy_upstream_name "default-hello-8080";
         proxy_pass http://default-hello-8080;
      }
   }
}
```


scale the hello service and check the /etc/nginx/nginx.conf file again.


```shell
$ kubectl scale deployment/hello --replicas=3
$ kubectl exec --namespace=kube-system nginx-ingress-controller-3023662530-g7b5m -- cat /etc/nginx/nginx.conf
http {
   upstream default-hello-8080 {
      least_conn;
      server 10.10.1.57:8080 max_fails=0 fail_timeout=0;
      server 10.10.4.11:8080 max_fails=0 fail_timeout=0;
      server 10.10.4.9:8080 max_fails=0 fail_timeout=0;
   }
   
   server {
      server_name test.io;
      listen 80;
      listen [::]:80;
      
      location /hello {
         set $proxy_upstream_name "default-hello-8080";
         proxy_pass http://default-hello-8080;
      }
   }
}
```


