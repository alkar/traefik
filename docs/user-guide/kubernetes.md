# Kubernetes Ingress Controller

This guide explains how to use Træfɪk as an Ingress controller in a Kubernetes cluster. 
If you are not familiar with Ingresses in Kubernetes you might want to read the [Kubernetes user guide](http://kubernetes.io/docs/user-guide/ingress/)

The config files used in this guide can be found in the [examples directory](https://github.com/containous/traefik/tree/master/examples/k8s)

## Prerequisites

1. A working Kubernetes cluster. If you want to follow along with this guide, you should setup [minikube](http://kubernetes.io/docs/getting-started-guides/minikube/)
on your machine, as it is the quickest way to get a local Kubernetes cluster setup for experimentation and development.

2. The `kubectl` binary should be [installed on your workstation](http://kubernetes.io/docs/getting-started-guides/minikube/#download-kubectl).

## Deploy Træfɪk

We are going to deploy Træfɪk with a
[Deployment](http://kubernetes.io/docs/user-guide/deployments/), as this will
allow you to easily roll out config changes or update the image.

```yaml
apiVersion: v1
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
  labels:
    k8s-app: traefik-ingress-lb
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: traefik-ingress-lb
  template:
    metadata:
      labels:
        k8s-app: traefik-ingress-lb
        name: traefik-ingress-lb
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - image: traefik
        name: traefik-ingress-lb
        resources:
          limits:
            cpu: 200m
            memory: 30Mi
          requests:
            cpu: 100m
            memory: 20Mi
        ports:
        - containerPort: 80
          hostPort: 80
        - containerPort: 8080
        args:
        - --web
        - --kubernetes
```
[examples/k8s/traefik.yaml](https://github.com/containous/traefik/tree/master/examples/k8s/traefik.yaml)

> notice that we binding port 80 on the Træfɪk container to port 80 on the host.
> With a multi node cluster we might expose Træfɪk with a NodePort or LoadBalancer service
> and run more than 1 replica of Træfɪk for high availability.

To deploy Træfɪk to your cluster start by submitting the deployment to the cluster with `kubectl`:

```sh
kubectl apply -f examples/k8s/traefik.yaml
```

### Check the deployment

Now lets check if our deployment was successful.

Start by listing the pods in the `kube-system` namespace:

```sh
$kubectl --namespace=kube-system get pods

NAME                                         READY     STATUS    RESTARTS   AGE
kube-addon-manager-minikubevm                1/1       Running   0          4h
kubernetes-dashboard-s8krj                   1/1       Running   0          4h
traefik-ingress-controller-678226159-eqseo   1/1       Running   0          7m
```

You should see that after submitting the Deployment to Kubernetes it has launched
a pod, and it is now running. _It might take a few moments for kubenetes to pull
the Træfɪk image and start the container._

> You could also check the deployment with the Kubernetes dashboard, run
> `minikube dashboard` to open it in your browser, then choose the `kube-system`
> namespace from the menu at the top right of the screen.

You should now be able to access Træfɪk on port 80 of your minikube instance.

```sh
curl $(minikube ip)
404 page not found
```

> We expect to see a 404 response here as we haven't yet given Træfɪk any configuration.

## Submitting An Ingress to the cluster.

Lets start by creating a Service and an Ingress that will expose the
[Træfɪk Web UI](https://github.com/containous/traefik#web-ui).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: traefik-web-ui
  namespace: kube-system
spec:
  selector:
    k8s-app: traefik-ingress-lb 
  ports:
  - port: 80
    targetPort: 8080
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: traefik-web-ui
  namespace: kube-system
spec:
  rules:
  - host: traefik-ui.local
    http:
      paths:
      - backend:
          serviceName: traefik-web-ui
          servicePort: 80
```
[examples/k8s/ui.yaml](https://github.com/containous/traefik/tree/master/examples/k8s/ui.yaml)

```sh
kubectl apply -f examples/k8s/ui.yaml
```

Now lets setup an entry in our /etc/hosts file to route `traefik-ui.local`
to our cluster. 

> In production you would want to set up real dns entries.

> You can get the ip address of your minikube instance by running `minikube ip`

```
echo "$(minikube ip) traefik-ui.local" | sudo tee -a /etc/hosts
```

We should now be able to visit [traefik-ui.local](http://traefik-ui.local) in the browser and view the Træfɪk Web UI.

## Name based routing

In this example we are going to setup websites for 3 of the United Kingdoms
best loved cheeses, Cheddar, Stilton and Wensleydale.

First lets start by launching the 3 pods for the cheese websites.

```yaml
---
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: stilton
  labels:
    app: cheese
    cheese: stilton
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cheese
      task: stilton
  template:
    metadata:
      labels:
        app: cheese
        task: stilton
        version: v0.0.1
    spec:
      containers:
      - name: cheese
        image: errm/cheese:stilton
        resources:
          requests:
            cpu: 100m
            memory: 50Mi
          limits:
            cpu: 100m
            memory: 50Mi
        ports:
        - containerPort: 80
---
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: cheddar
  labels:
    app: cheese
    cheese: cheddar
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cheese
      task: cheddar
  template:
    metadata:
      labels:
        app: cheese
        task: cheddar
        version: v0.0.1
    spec:
      containers:
      - name: cheese
        image: errm/cheese:cheddar
        resources:
          requests:
            cpu: 100m
            memory: 50Mi
          limits:
            cpu: 100m
            memory: 50Mi
        ports:
        - containerPort: 80
---
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: wensleydale
  labels:
    app: cheese
    cheese: wensleydale
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cheese
      task: wensleydale
  template:
    metadata:
      labels:
        app: cheese
        task: wensleydale
        version: v0.0.1
    spec:
      containers:
      - name: cheese
        image: errm/cheese:wensleydale
        resources:
          requests:
            cpu: 100m
            memory: 50Mi
          limits:
            cpu: 100m
            memory: 50Mi
        ports:
        - containerPort: 80
```
[examples/k8s/cheese-deployments.yaml](https://github.com/containous/traefik/tree/master/examples/k8s/cheese-deployments.yaml)

```sh
kubectl apply -f examples/k8s/cheese-deployments.yaml
```

Next we need to setup a service for each of the cheese pods.

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: stilton
spec:
  ports:
  - name: http
    targetPort: 80
    port: 80
  selector:
    app: cheese
    task: stilton
---
apiVersion: v1
kind: Service
metadata:
  name: cheddar
spec:
  ports:
  - name: http
    targetPort: 80
    port: 80
  selector:
    app: cheese
    task: cheddar
---
apiVersion: v1
kind: Service
metadata:
  name: wensleydale
spec:
  ports:
  - name: http
    targetPort: 80
    port: 80
  selector:
    app: cheese
    task: wensleydale
```
[examples/k8s/cheese-services.yaml](https://github.com/containous/traefik/tree/master/examples/k8s/cheese-services.yaml)

```sh
kubectl apply -f examples/k8s/cheese-services.yaml
```

Now we can submit an ingress for the cheese websites.

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: cheese
spec:
  rules:
  - host: stilton.local
    http:
      paths:
      - path: /
        backend:
          serviceName: stilton
          servicePort: http
  - host: cheddar.local
    http:
      paths:
      - path: /
        backend:
          serviceName: cheddar
          servicePort: http
  - host: wensleydale.local
    http:
      paths:
      - path: /
        backend:
          serviceName: wensleydale
          servicePort: http
```
[examples/k8s/cheese-ingress.yaml](https://github.com/containous/traefik/tree/master/examples/k8s/cheese-ingress.yaml)

> Notice that we list each hostname, and add a backend service.

```sh
kubectl apply -f examples/k8s/cheese-ingress.yaml
```

Now visit the [Træfɪk dashboard](http://traefik-ui.local/) and you should
see a frontend for each host. Along with a backend listing for each service
with a Server set up for each pod.

If you edit your `/etc/hosts` again you should be able to access the cheese
websites in your browser.

```sh
echo "$(minikube ip) stilton.local cheddar.local wensleydale.local" | sudo tee -a /etc/hosts
```

* [Stilton](http://stilton.local/)
* [Cheddar](http://cheddar.local/)
* [Wensleydale](http://wensleydale.local/)

## Path based routing

Now lets suppose that our fictional client has decided that while they are
super happy about our cheesy web design, when they asked for 3 websites
they had not really bargained on having to buy 3 domain names.

No problem, we say, why don't we reconfigure the sites to host all 3 under one domain.


```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: cheeses
  annotations:
    traefik.frontend.rule.type: pathprefixstrip
spec:
  rules:
  - host: cheeses.local
    http:
      paths:
      - path: /stilton
        backend:
          serviceName: stilton
          servicePort: http
      - path: /cheddar
        backend:
          serviceName: cheddar
          servicePort: http
      - path: /wensleydale
        backend:
          serviceName: wensleydale
          servicePort: http
```
[examples/k8s/cheeses-ingress.yaml](https://github.com/containous/traefik/tree/master/examples/k8s/cheeses-ingress.yaml)

> Notice that we are configuring Træfɪk to strip the prefix from the url path
> with the `traefik.frontend.rule.type` annotation so that we can use
> the containers from the previous example without modification.

```sh
kubectl apply -f examples/k8s/cheeses-ingress.yaml
```

```sh
echo "$(minikube ip) cheeses.local" | sudo tee -a /etc/hosts
```

You should now be able to visit the websites in your browser.

* [cheeses.local/stilton](http://cheeses.local/stilton/)
* [cheeses.local/cheddar](http://cheeses.local/cheddar/)
* [cheeses.local/wensleydale](http://cheeses.local/wensleydale/)
