## k3d istio Canary and Blue/Green deployments (POC)

### Create k3d cluster 
```
k3d cluster create istio  --k3s-arg "--disable=traefik@server:0" \
       --port 9443:443@loadbalancer \
       --port 8080:80@loadbalancer \
       --api-port 6443 \
       --wait 
```

### Install Istio
```
$ ISTIO_VERSION=1.18.0
$ curl -L https://istio.io/downloadIstio | ISTIO_VERSION=${ISTIO_VERSION} TARGET_ARCH=x86_64 sh
$ ./istio-${ISTIO_VERSION}/bin/istioctl install  --set profile=default --skip-confirmation
✔ Istio core installed
✔ Istiod installed
✔ Ingress gateways installed
✔ Installation complete                                                                                                                                                                                                                                                                     Making this installation the default for injection and validation.

$ ./istio-${ISTIO_VERSION}/bin/istioctl analyze

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

Kiali:

istio-system: 
<img src="screenshots/k3d-istio-kiali.png?raw=true" width="1000">

Prometheus:

<img src="screenshots/k3d-istio-prometheus.png?raw=true" width="1000">

Grafana:

<img src="screenshots/k3d-istio-grafana.png?raw=true" width="1000">

Jaeger (Tracing):

<img src="screenshots/k3d-istio-jaeger.png?raw=true" width="1000">


```
## istio-helm-deployment

Pre: Create docker images
```
$ cd demoapp
$ docker build -t davarski/testing-app:v1 .
Change index.html: v1 -> v2
$ docker build -t davarski/testing-app:v2 .
$ docker push davarski/testing-app:v1
$ docker push davarski/testing-app:v2
```

Create canary and blue/green deployment of a demo application with Istio and Helm on k3d

**We will use the following resources**

- demo application helm chart
- basic istio configuration


**Installation**
```shell
$ kubectl create namespace prod
$ kubectl create namespace stage
$ kubectl label namespace prod istio-injection=enabled
$ kubectl label namespace stage istio-injection=enabled
$ helm install demoappv1 helm-chart/demoapp/ --wait --set deployment.tag=v1 --namespace prod
$ helm install demoappv2 helm-chart/demoapp/ --wait --set deployment.tag=v2 --namespace stage
$ kubectl create -f istio-config/gateway.yaml
$ kubectl create -f istio-config/vsvc.yaml
```

If everything went good, you should be able to see in your kiali versioned graph the following:
![Screenshot](screenshot.png)

---

## How it works 

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
---
Gateway file applies a listening policy to the istio ingress-controller whereas virtualservice maps that gateway with the services we would like to destribute the traffic to.

