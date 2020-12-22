# K8S POD with Multi-Container for Web page serving (Over Minikube)

In this project we see an example how to use a Pod with multiple containers (side-cars).
Here we also can see how we can create a Kong Routing for these multiple services in order to serve Web sites, and specificaly multiple web sites.

The main concept here is to use Kong ingress CRD with "host" keyword, making it a hostname base route.

The Base Architecture is using a Pod with 2 containers:

1. Application
2. Nginx

The Route from outside via Kong should be received by the Nginx.
From there the Nginx will route the rest to the application container.

When using the hostname in the ingress CRD we would have the web page internal resources relative to the same hostname (which would be different as defined in the main use case).

meaning a page requested by my.example1.com would return resource reference to my.example1.com/faicon.png etc. and not to the kong Base URL.

### Note: We will use Minikube to simulate the flow.


### Stage 0 - Creating the Docker Images inside Minikube VM:
First we should define the minikube VM as the Docker Proxy :
```bash
eval $(minikube docker-env)
```

### now we build the images
1. Build the nginx-proxy:
```bash
docker build -t nginx-web-1:v1.0.0 -f ./nginx-proxy/Dockerfile ./nginx--proxy/
```
2. Build the nginx-web-1:
```bash
docker build -t nginx-web-1:v1.0.0 -f ./nginx-web-1/Dockerfile ./nginx-web-1/
```
We can do the same for the Web-2 via:
```bash
docker build -t nginx-web-2:v1.0.0 -f ./nginx-web-2/Dockerfile ./nginx-web-2/
```
3. Now we check for the images in Minikube:
```bash
docker images
```
Note: In order to make the K8S pull the images from the VM host we set the Container with the property:
```
imagePullPolicy: Never
``` 

### Create a namespace

```bash
kubectl create namespace multi-test
```
### Set the namespace as default

```bash
kubectl config set-context --current --namespace= multi-test
```

### Stage 1 - Creating the Deployment:

**Deploying the Application Containers and Cluster IP Service** 
```bash
kubectl apply -f service-pod-1.yaml
```
**Deploying the Ingress CRD for routing**

```bash
kubectl apply -f ingress-resource-web-1.yaml
```

Notice the Ingress:
```bash
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: multi-test-web-1
  annotations:
    kubernetes.io/ingress.class: kong
    konghq.com/strip-path: "true"
spec:
  rules:
  - host: myapp.example1.com
    http:
      paths:
      - path: /
        backend:
          serviceName: multi-test-service-1
          servicePort: 5035


```

### Stage 2 - Add kong to the cluster:
**install kong**
[kong deployment eks](https://github.com/Kong/kubernetes-ingress-controller/blob/master/docs/deployment/eks.md)
```bash
kubectl create -f https://bit.ly/k4k8s
```

### Stage 3 - Creating hostname:
**Create host name in the hosts file**

first get the minikube kong url for the service:
```bash
minikube service kong-proxy --url  -nkong
```
the response would be something like:
```bash
http://192.168.64.13:32092
http://192.168.64.13:31639
```
in the hosts file we will add:

```bash
192.168.64.13 myapp.example1.com
```


Now we can open the browser with:
```bash
myapp.example1.com:32092
```
and we will see that we get the "Web 1" page



## Utils:
### looking into a container in a multi caontainer Pod

```bash
kubectl get pod

NAME                                      READY   STATUS    RESTARTS   AGE
multi-web-deployment-1-54fb5d6884-89pbh   2/2     Running   0          37m
multi-web-deployment-2-77d67c4d8c-8rzv2   2/2     Running   0          37m
```

Getting Exec into a container:
we should define the container name when executing into a container.
```bash
kubectl exec -it multi-web-deployment-1-54fb5d6884-89pbh nginx-proxy -- sh

Defaulting container name to nginx-web-1.
Use 'kubectl describe pod/multi-web-deployment-1-54fb5d6884-89pbh -n multi-test' to see all of the containers in this pod.
#
```

Getting Logs into a container:
```bash
kubectl logs -f multi-web-deployment-1-54fb5d6884-89pbh

error: a container name must be specified for pod multi-web-deployment-1-54fb5d6884-89pbh, choose one of: [nginx-web-1 nginx-proxy]
```
So we should use the container name:

```bash
kubectl logs -f multi-web-deployment-1-54fb5d6884-89pbh nginx-proxy

/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: info: /etc/nginx/conf.d/default.conf differs from the packaged version
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
```
