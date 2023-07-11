## k3d istio Canary and Blue/Green deployments (POC)

In this playground, we are going to review ways in which we can deploy two versions of our application in production-like environments in Kubernetes and apply two different availability approaches. The first is to distribute network traffic evenly between the two versions (canary) and the second is for 100% of the traffic to either one of them (blue/green) with zero downtime.

### WHAT IS BLUE/GREEN DEPLOYMENT?
Blue-green deployment is a technique that reduces downtime and risk by running two identical production environments called Blue and Green.

At any time, only one of the environments is live, with the live environment serving all production traffic. For this example, Blue is currently live and Green is idle.
As you prepare a new version of your software, deployment and the final stage of testing takes place in the environment that is not live: in this example, Green. Once you have deployed and fully tested the software in Green, you switch the router so all incoming requests now go to Green instead of Blue. Green is now live, and Blue is idle.

<img src="screenshots/k8s-istio-blue-green.png?raw=true" width="1000">

### WHAT IS CANARY DEPLOYMENT THEN?
Canary deployments are a pattern for rolling out releases to a subset of users or servers. The idea is to first deploy the change to a small subset of servers, test it, and then roll the change out to the rest of the servers. The canary deployment serves as an early warning indicator with less impact on downtime: if the canary deployment fails, the rest of the servers aren't impacted.

<img src="screenshots/k8s-istio-canary.png?raw=true" width="1000">

### WHAT IS HELM ALL ABOUT?
Helm is the first application package manager running on top of Kubernetes. It allows you to describe the application structure through convenient helm-charts and to manage it with simple commands.

Why is Helm important? Because it’s a huge shift in the way the server-side applications are defined, stored, and managed. Adoption of Helm might well be the key to the mass adoption of microservices, as using this package manager simplifies their management greatly.

Why are microservices so important? They have quite a few uses:

When there are several microservices instead of a monolithic application, each microservice can be managed, updated and scaled individually
Issues with one microservice do not affect the functionality of other components of the application
The new application can be easily composed of existing loosely-coupled microservices

### WHAT IS ISTIO?
Istio addresses the challenges developers and operators face as monolithic applications transition towards a distributed microservice architecture. To see how it helps to take a more detailed look at Istio’s service mesh.

The term service mesh is used to describe the network of microservices that make up such applications and the interactions between them. As a service mesh grows in size and complexity, it can become harder to understand and manage. Its requirements can include discovery, load balancing, failure recovery, metrics, and monitoring. A service mesh also often has more complex operational requirements, like A/B testing, canary rollouts, rate limiting, access control, and end-to-end authentication.

Istio provides behavioral insights and operational control over the service mesh as a whole, offering a complete solution to satisfy the diverse requirements of microservice applications.

Now let us dive into the solution:

If you want to follow along with this article, you will need to install all necessary resources:

- k3d cluster
- kubectl
- istio
- istioctl
- helm3

As you can see from the repository, I have wrapped my application into a Helm chart so that it can be installed as a package. Thanks to Helm I am able to dynamically install different versions of the application on runtime.

### Create k3s cluster 
```
k3d cluster create istio  --k3s-arg "--disable=traefik@server:0" \
       --port 9443:443@loadbalancer \
       --port 8080:80@loadbalancer \
       --api-port 6443 \
       --wait

Note: We can use k3s also (not tested)

```
### Install Istio
```
ISTIO_VERSION=1.18.0
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=${ISTIO_VERSION} TARGET_ARCH=x86_64 sh
./istio-${ISTIO_VERSION}/bin/istioctl install  --set profile=default --skip-confirmation
✔ Istio core installed
✔ Istiod installed
✔ Ingress gateways installed
✔ Installation complete                                                                                                                                                                                                                                                                     Making this installation the default for injection and validation.

./istio-${ISTIO_VERSION}/bin/istioctl analyze

kubectl apply -f ./istio-${ISTIO_VERSION}/samples/addons/kiali.yaml
kubectl apply -f ./istio-${ISTIO_VERSION}/samples/addons/jaeger.yaml
kubectl apply -f ./istio-${ISTIO_VERSION}/samples/addons/prometheus.yaml
kubectl apply -f ./istio-${ISTIO_VERSION}/samples/addons/grafana.yaml

kubectl apply -f ./ingresses/  -n istio-system
```

### Check istio installation: 

```
$ kubectl get po -n istio-system
NAME                                    READY   STATUS    RESTARTS   AGE
istiod-786fddf775-x5vdk                 1/1     Running   0          32m
istio-ingressgateway-644bdd7769-jt86m   1/1     Running   0          31m
jaeger-cc4688b98-42rnp                  1/1     Running   0          8m26s
prometheus-67f6764db9-5x8mz             2/2     Running   0          8m24s
kiali-75b4b9df64-fxsb4                  1/1     Running   0          8m26s
grafana-6cb5b7fbb8-fh2lm                1/1     Running   0          7m19s

$ kubectl get svc -n istio-system
NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                                      AGE
istiod                 ClusterIP      10.43.226.173   <none>        15010/TCP,15012/TCP,443/TCP,15014/TCP        32m
istio-ingressgateway   LoadBalancer   10.43.190.112   172.24.0.2    15021:31905/TCP,80:31413/TCP,443:31757/TCP   32m
kiali                  ClusterIP      10.43.30.161    <none>        20001/TCP,9090/TCP                           9m1s
tracing                ClusterIP      10.43.45.101    <none>        80/TCP,16685/TCP                             9m1s
zipkin                 ClusterIP      10.43.7.106     <none>        9411/TCP                                     9m1s
jaeger-collector       ClusterIP      10.43.159.96    <none>        14268/TCP,14250/TCP,9411/TCP                 9m1s
prometheus             ClusterIP      10.43.115.20    <none>        9090/TCP                                     9m
grafana                ClusterIP      10.43.162.18    <none>        3000/TCP                                     7m54s

$ kubectl get ing -n istio-system
NAME      CLASS    HOSTS                         ADDRESS      PORTS   AGE
grafana   <none>   grafana.192.168.1.99.nip.io   172.24.0.2   80      24m
kiali     <none>   kiali.192.168.1.99.nip.io     172.24.0.2   80      24m
prom      <none>   prom.192.168.1.99.nip.io      172.24.0.2   80      24m
tracing   <none>   tracing.192.168.1.99.nip.io   172.24.0.2   80      24m
```

### Browser:

- Kiali: http://kiali.192.168.1.99.nip.io:8080
- Prometheus: http://prom.192.168.1.99.nip.io:8080
- Grafana: http://grafana.192.168.1.99.nip.io:8080
- Jaeger: http://tracing.192.168.1.99.nip.io:8080

### Screenshots:

Kiali (istio-system namespace)

<img src="screenshots/istio-kiali-istio-system-namespace.png?raw=true" width="1000">

Prometheus:

<img src="screenshots/k3d-istio-prometheus.png?raw=true" width="1000">

Grafana:

<img src="screenshots/k3d-istio-grafana.png?raw=true" width="1000">

Jaeger (Tracing):

<img src="screenshots/k3d-istio-jaeger.png?raw=true" width="1000">


###  istio-helm-deployment

Pre: Create docker images
```
cd demoapp
docker build -t davarski/testing-app:v1 .
Change index.html: v1 -> v2
docker build -t davarski/testing-app:v2 .
docker push davarski/testing-app:v1
docker push davarski/testing-app:v2
```
I have created two versions of the application that was built in Docker images. 


Create canary and blue/green deployment of a demo application with Istio and Helm on k3d:

**We will use the following resources**

- demo application helm chart
- basic istio configuration


**Installation**

#### We need to create two namespaces (prod and stage) in which we will put the different versions of our application - let us create them:

```shell
kubectl create namespace prod
kubectl create namespace stage
```
#### Then we need to tell Istio to monitor these namespaces by labeling them so it can keep track of the resources inside them:
```
kubectl label namespace prod istio-injection=enabled
kubectl label namespace stage istio-injection=enabled
```
#### Now we can deploy the two versions of our application in each individual namespace by:
```
helm install demoappv1 helm-chart/demoapp/ --wait --set deployment.tag=v1 --namespace prod
helm install demoappv2 helm-chart/demoapp/ --wait --set deployment.tag=v2 --namespace stage
```
#### Once deployed, we have to install the two Istio files in the istio-config directory that will basically do the magic:
```
kubectl create -f istio-config/gateway.yaml
kubectl create -f istio-config/vsvc.yaml
```

Both of these files and almost everything in Istio are built and run thanks to the Kubernetes Custom Resources Definitions.

Custom Resources Definition (CRD) is a powerful feature introduced in Kubernetes which enables users to add their own customized objects to the Kubernetes cluster and use it like any other native Kubernetes objects. So what is the Gateway project then? Let us look below at the official Istio explanation.

"Gateway describes a load balancer operating at the edge of the mesh receiving incoming or outgoing HTTP/TCP connections. The specification describes a set of ports that should be exposed, the type of protocol to use, SNI configuration for the load balancer, etc."

### WHAT IS VIRTUALSERVICE?
Let us look again into the official documentation:

"A VirtualService defines a set of traffic routing rules to apply when a host is addressed. Each routing rule defines matching criteria for the traffic of a specific protocol. If the traffic is matched, then it is sent to a named destination service (or subset/version of it) defined in the registry."

So basically Gateway describes the listening policy of the Istio’s ingress controller and VirtualService will “glue” this Gateway service to the services of the actual pods running in the cluster.

With VirtualService we actually tell Istio how to distribute the traffic to our services. Weight in this case refers to the traffic percentage on the said environment. It is very important that we have a formal means of redistributing traffic between the two environments (prod & stage) with different versions of the code. The goal is to have the new version of the application fully tested in the staging environment and then migrating it to production, with 100% traffic (while keeping the old code for roll-back purposes). Production migration has to happen gradually. The staging environment needs to mandatorily be tested at 0% traffic to ensure it is ready for the gradual increase of traffic. Once these tests are successful, staging is ready for a 30% traffic redistribution, leaving production with 70% of the traffic.

The traffic increase on staging needs to happen gradually until it reaches 100% and tests have to be performed at every stage. Once the entire traffic has been migrated, we will have a staging environment at 100% traffic and a production environment at 0%. The next step will be to manually update the production environment with the latest version (that is on staging) and then switch the whole traffic (100%) back to production, once we are ready.

### How it works 

The magic happens in the next two files that we applied earlier

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: app-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
```
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: demoapp
spec:
  hosts:
  - "*"
  gateways:
  - app-gateway
  http:
    - route:
      - destination:
          host: flaskapp.prod.svc.cluster.local 
        weight: 50
      - destination:
          host: flaskapp.stage.svc.cluster.local
        weight: 50
 ```

Gateway file applies a listening policy to the istio ingress-controller whereas virtualservice maps that gateway with the services we would like to destribute the traffic to.

The next step will be to get the DNS name of the load-balancer created by Istio:

```
kubectl get svc -n istio-system | grep -i LoadBalancer | awk '{print $4}'[/php]
```

Then open your browser and copy-paste the DNS name of the LB you are going to see the current version set up by the VirtualService weight distribution:

If everything went good, you should be able to see in your kiali versioned graph the following:

<img src="screenshots/screenshot.png?raw=true" width="900">

With Istio, we can do a lot of advanced configuration such as SSL mutual authentication between microservices and apply advanced routing policies with the help of DestinationRule.

```
kubectl get svc istio-ingressgateway -n istio-system
export INGRESS_HOST=$(kubectl get po -l istio=ingressgateway -n istio-system -o jsonpath='{.items[0].status.hostIP}')
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
echo "$GATEWAY_URL"
curl -v $GATEWAY_URL (10 times for example)
```

<img src="screenshots/istio-kiali-helm.png?raw=true" width="900">

---


### CONCLUSION
In this playground, we discovered together how can we apply a hybrid solution between Canary and Blue/Green deployment with the help of Helm and Istio. 

Here are some sources for further reading:

Kubernetes Fundamentals on Strive 2 Code
Canary Deployments on Octopus Deploy
Istio Docs
Blue/Green Deployments on O'Reilly
Using Canary Release Pipelines to Achieve Continuous Testing on Sauce Labs


