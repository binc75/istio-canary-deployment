# ISTIO tutorial -- Canary Deployment
We are going to setup a *minikube* cluster, install *istio* and install an application to demonstrate the ability of istio with **canary deployments**.  


![Setup schema](img/istio-app-schema.png?raw=true "Schema")


## Environment setup
Setup minikube with the appropriate resources:
```bash
minikube start -p istio-mk --memory=8192 --cpus=3 \
  --kubernetes-version=v1.17.0 \
  --vm-driver=virtualbox --disk-size=20g
```

## ISTIO setup
```bash
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.4.6/
export PATH=$PWD/bin:$PATH

## The demo configuration profile is not suitable for performance evaluation. 
## It is designed to showcase Istio functionality with high levels of tracing and access logging
## For a production setup consider: $ istioctl manifest apply

istioctl manifest apply --set profile=demo
```
### Display the list of available profiles (optional)
```bash
$ istioctl profile list
Istio configuration profiles:
    minimal
    remote
    sds
    default
    demo
```
### Display the configuration of a profile (optional)
You can view the configuration settings of a profile.  
For example, to view the setting for the demo profile run the following command:
```bash
$ istioctl profile dump demo
autoInjection:
  components:
    injector:
      enabled: true
      k8s:
        replicaCount: 1
        strategy:
          rollingUpdate:
            maxSurge: 100%
            maxUnavailable: 25%
  enabled: true
cni:
  components:
    cni:
      enabled: false
  enabled: false
...
```

### Uninstall (optional)
```bash
istioctl manifest generate --set profile=demo | kubectl delete -f -
```

## Configure sidecar injector 
Setup the automatic sidecar injection feature of istio for the *default* namespace:  
```bash
kubectl label namespace default istio-injection=enabled
```
Check
```bash
kubectl describe ns default
```
### Check if sidecar injection works (optional)
```bash
kubectl create deployment --image=nginx nginxdeploy
kubectl describe pods nginxdeploy-57c4d97988-zvl56
```

## Deploy example app
```bash
kubectl apply -f deployment/frontend.yaml
kubectl apply -f deployment/backend.yaml
kubectl apply -f deployment/istio-gw.yaml
kubectl apply -f deployment/istio-frontend.yaml
kubectl apply -f deployment/istio-backend.yaml
```
Quick explanation:  
* frontend.yaml
  * nginx reverse proxy (2 versions: frontend-v1 & frontend-v2)
  * configmap (nginx-config) for the nginx reverse proxy
  * service (ClusterIP) to expose the nginx pool (label: app: frontend)
* backend.yaml
  * custom python flask API application (2 versions: backend-v1 & backend-v2)  
    This application read the env variable *VERSION* and return it when called.
  * service (ClusterIP) to expose the application (label: app: backend)
* istio-gw.yaml
  * define an istio gateway named *http-gateway* that route traffic from the external world to the inside of k8s
* istio-frontend.yaml
  * defines an istio VirtualService that rely on the istio gateway to route traffic towards the frontend nginx instances
  * defines an istio DestinationRule to distinguish between the 2 versions of the frontend nginx  
    (labels: version: v1 & v2)
* istio-backend.yaml
  * defines an istio VirtualService that route traffic to the instances of the python app
  * defines an istio DestinationRule to distinguish between the 2 versions of the python app  
    (labels: version: v1 & v2)
## Checking out features
Get and set env variables
```bash
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
export INGRESS_HOST=$(minikube -p istio-mk ip)
```
Check if the app respond:
```bash
while true; do curl http://$INGRESS_HOST:$INGRESS_PORT -HHost:www.example.com; sleep .2;done
```
or
```bash
while true; do curl http://www.example.com:$INGRESS_PORT --resolve www.example.com:$INGRESS_PORT:$INGRESS_HOST -HHost:www.example.com; sleep .2;done
```
### Query the app with specific http header
Istio conf (see *deployment/istio-backend.yaml*) force http connections with this header **canary: canary-tester** to only reach the V2, test it out:
```bash
while true; do curl http://www.example.com:$INGRESS_PORT --resolve www.example.com:$INGRESS_PORT:$INGRESS_HOST -HHost:www.example.com -H "canary: canary-tester"; sleep .2;done
```

### Modify traffic
You can now play with the weights for the backend service in the *deployment/istio-xxx.yaml* file and the apply the new configuration
```bash
kubectl apply -f deployment/istio-xxx.yaml
```

## Kiali
To have a realtime graphical representation of the situation you can look at *kiali*
```bash
istioctl dashboard kiali
```
Here an example of the traffic balance:
![Kiali View](img/istio-traffic-shaping.png?raw=true "Kiali View")

## Cleanup 
```bash
minikube -p istio-mk stop
minikube -p istio-mk delete
```

