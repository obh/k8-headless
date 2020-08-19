# Setting up NGINX INGRESS in Google Cloud

This setup helps you create a NGINX load balancer for your service. I created this for load balancing a websocket service. 
Most of this process is fairly well-documented. However, just for posteriority's sake, I'll just list down the specific steps.

## Which NGINX?
So there are two different setups out there. One of them is provided by Kubernetes where they have used the nginx image and the other one is officially provided by NGINX.
The one provided by NGINX has two versions - a community version and a paid one. The gotcha is that if you use one and add annotations from the other one, you'll end up in a soup.
[Here](https://www.nginx.com/blog/wait-which-nginx-ingress-controller-kubernetes-am-i-using/) is a blog post about it. 

I used the official nginx version.

## Installation
The installation steps mentioned [here](https://docs.nginx.com/nginx-ingress-controller/installation/installation-with-manifests/) are the ones to follow. I'll list these down.
```
git clone https://github.com/nginxinc/kubernetes-ingress/
cd kubernetes-ingress/deployments
git checkout v1.8.1
```

<b>Note</b>: If you don't use the 1.8.1 (or latest stable version), you'll end up using the edge release (which is the latest one). 

## Basics (NS, Roles etc)
Next we need to create the namespace, the roles for nginx, access etc.

`kubectl apply -f common/ns-and-sa.yaml`

`kubectl apply -f rbac/rbac.yaml`

## Config and certificates
Ideally you should have your own certificate (you can use Letsencrypt), however for installation there is a default certificate provided. We need to add this as a secret.
We can also add a config map (we can add more configs later)

`kubectl apply -f common/default-server-secret.yaml`
`kubectl apply -f common/nginx-config.yaml`

## Custom Resources (Maybe?)
We can skip installing any custom resources. It is required if you want to do advanced routing. 

## Deploy
To deploy the ingress controller, just run.

`kubectl apply -f deployment/nginx-ingress.yaml`

Then we need to expose this as a deployment. For GCP you need to create a load balancer service (there is a different resource for AWS)

`kubectl apply -f service/loadbalancer.yaml`

This will generate the load balancer IP. 

## Ingress
Hoping that you already have a service which is deployed and needs to be exposed. We just now need to create a new ingress resource and point it to this service.

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  namespace: headless
  name: browserless-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.org/websocket-services: "browserless"
    nginx.org/lb-method: "least_conn"
    nginx.org/client-max-body-size: "4m"
    nginx.org/proxy-connect-timeout: "86400s"
    nginx.org/proxy-read-timeout: "86400s"
    nginx.org/proxy-send-timeout: "86400s"

spec:
  backend:
    serviceName: browserless
    servicePort: 80
  rules:
  - host: test.gce.com
    http:
      paths:
      - path: /browserless
        backend:
          serviceName: browserless
          servicePort: 80
```          
          
## Gotchas
Ensure that you give the nginx user appropriate permisssions to access the kubernetes api
`kubectl create clusterrolebinding nginx-ingress-admin -n nginx-ingress  --clusterrole=cluster-admin  --serviceaccount=nginx-ingress:nginx-ingres`

<b>Note</b>: If you ever use the GCP load balancer, it has a default connection timeout of 30 seconds, which doesn't work for websockets. To avoid that you'll have to create a new backend config and add it to your service.

```backendconfig.yaml

apiVersion: cloud.google.com/v1beta1
kind: BackendConfig
metadata:
  name: browserless-backendconfig
  namespace: headless
spec:
  timeoutSec: 1500
  connectionDraining:
    drainingTimeoutSec: 1500
```
and add this to your service as 

```
kind: Service
metadata:
  name: headless
  annotations:
    beta.cloud.google.com/backend-config: '{"ports": {"8080":"mediaproxy-backend-config"}}'

...
```
    
