---
layout: post
title:  "Setup Kong as Ingress Controller"
date:   2018-10-20 11:30:00 +0100
categories: [kubernetes, kong]
---

## We will use Minikube for a Quick Demo

### We deploy complete NGINX Plus Ingress Controller with a full deployment file
~~~
curl https://raw.githubusercontent.com/Codegazers/k8s-vagrant/master/demo/nginx-plus-ingress_full-deployment.yml | kubectl apply -f -
~~~

### We have created 'nginx-ingress' namespace so we will wait until all components are ready.
~~~
kubectl get all --namespace nginx-ingress
~~~
### We deploy now two deployments with their services (red-app and blue-app with red-svc and blue-svc) 
~~~
curl https://raw.githubusercontent.com/Codegazers/k8s-vagrant/master/examples/colors/colors.yml | kubectl create -f -
~~~

### We review the deployments
~~~
kubectl get all --namespace default
~~~

### For quick access we prepare some environment variables
~~~


export INGRESS_IP=$(minikube   service -n nginx-ingress nginx-ingress --url --format "{{ .IP }}" | head -1)

export INGRESS_HTTP_PORT=$(minikube   service -n nginx-ingress nginx-ingress --url --format "{{ .Port }}" | head -1)

export INGRESS_HTTPS_PORT=$(minikube   service -n nginx-ingress nginx-ingress --url --format "{{ .Port }}" | tail -1)

~~~

### Now we deploy colors-ingress ingress resource for accesing both applications
~~~
curl https://raw.githubusercontent.com/Codegazers/k8s-vagrant/master/examples/colors/colors-ingress.yml | kubectl create -f -
~~~

### We just set up host routing
~~~
kubectl get ingress colors-ingress -o yaml
~~~

### And now we can access our applications setting host headers
~~~
http ${INGRESS_IP}:${INGRESS_HTTP_PORT}/text Host:red.example.com

http ${INGRESS_IP}:${INGRESS_HTTP_PORT}/text Host:blue.example.com
~~~

### We review backends because red-app has 3 replicas
~~~
ab -n 10000 -v 2 -k  -H "host: red.example.com" ${INGRESS_IP}:${INGRESS_HTTP_PORT}/text
~~~

-------

## Ingress Controller with HealthChecks

### We will deploy now application with active health checks
~~~
curl https://raw.githubusercontent.com/Codegazers/k8s-vagrant/master/examples/colors/colors-with-health.yml | kubectl apply -f -


~~~


### And now we update ingress resource 
~~~
kubectl delete ingress colors-ingress # Or just update using apply

curl https://raw.githubusercontent.com/Codegazers/k8s-vagrant/master/examples/colors/colors-ingress-with-health.yml | kubectl apply -f -
~~~

### Checking health for blue.example.com we notice that the 2 replicas are working fine (healthcheck at /health)
~~~
http ${INGRESS_IP}:${INGRESS_HTTP_PORT}/text Host:blue.example.com
~~~

#### _Sample output_
~~~
${INGRESS_IP}:${INGRESS_HTTP_PORT}/text Host:blue.example.com
HTTP/1.1 200 OK
Connection: keep-alive
Date: Thu, 06 Sep 2018 16:57:40 GMT
Server: nginx/1.13.10
Transfer-Encoding: chunked

APP_VERSION: 1.0
COLOR: blue
CONTAINER_NAME: blue-app-66f8cc9cf9-zgwqv
CONTAINER_IP: 172.17.0.7


{INGRESS_IP}:${INGRESS_HTTP_PORT}/text Host:blue.example.com
HTTP/1.1 200 OK
Connection: keep-alive
Date: Thu, 06 Sep 2018 16:57:41 GMT
Server: nginx/1.13.10
Transfer-Encoding: chunked

APP_VERSION: 1.0
COLOR: blue
CONTAINER_NAME: blue-app-66f8cc9cf9-twhr5
CONTAINER_IP: 172.17.0.8

~~~

### Now we set one of the application pods to a failure state
~~~

kubectl get pods 

kubectl exec -ti <POD>  -- touch /tmp/down

~~~

#### _Sample output_
~~~
$ kubectl get pods 
NAME                        READY     STATUS    RESTARTS   AGE
blue-app-66f8cc9cf9-twhr5   1/1       Running   0          1m
blue-app-66f8cc9cf9-zgwqv   1/1       Running   0          1m
red-app-5969d9f7bd-dnwwz    1/1       Running   0          1m
red-app-5969d9f7bd-gszsq    1/1       Running   0          1m
red-app-5969d9f7bd-zhmh4    1/1       Running   0          1m

$kubectl exec -ti blue-app-66f8cc9cf9-twhr5  -- touch /tmp/down
~~~

### Now we have one endpoint in failure state
If we are fast enought we can reach failure endpoint before it will disable.
Review health a few times
~~~
http ${INGRESS_IP}:${INGRESS_HTTP_PORT}/health Host:blue.example.com

http ${INGRESS_IP}:${INGRESS_HTTP_PORT}/health Host:blue.example.com
~~~

Failed endpoint will be disabled and we just reach alive endpoint.
Review text page a few times
~~~
http ${INGRESS_IP}:${INGRESS_HTTP_PORT}/text Host:blue.example.com

http ${INGRESS_IP}:${INGRESS_HTTP_PORT}/text Host:blue.example.com
~~~

### Now we set failed pod to a 'alive state'
~~~

kubectl get pods 

kubectl exec -ti <FAILURE_POD_USED_BEFORE>  -- rm /tmp/down

~~~

#### _Sample output_
~~~
$kubectl exec -ti blue-app-66f8cc9cf9-twhr5  -- rm /tmp/down
~~~
Now we will review health a few times
~~~
http ${INGRESS_IP}:${INGRESS_HTTP_PORT}/health Host:blue.example.com

http ${INGRESS_IP}:${INGRESS_HTTP_PORT}/health Host:blue.example.com
~~~

And failed endpoint will be alive again and will be added to the upstream.

Review text page a few times
~~~
http ${INGRESS_IP}:${INGRESS_HTTP_PORT}/text Host:blue.example.com

http ${INGRESS_IP}:${INGRESS_HTTP_PORT}/text Host:blue.example.com
~~~

-------

## Ingress Controller rewriting 
### Update ingress resource 
~~~
kubectl delete ingress colors-ingress

curl https://raw.githubusercontent.com/Codegazers/k8s-vagrant/master/examples/colors/colors-ingress-rewrite.yml | kubectl apply -f -
~~~
### We can just test rewrite easily using /red/ path on blue deployment:
~~~
http ${INGRESS_IP}:${INGRESS_HTTP_PORT}/red/text Host:blue.example.com
~~~
Results will show a red application pod

-------

## Ingress Controller Sticky Sessions Persistence
### Update ingress resource 
~~~
curl https://raw.githubusercontent.com/Codegazers/k8s-vagrant/master/examples/colors/colors-ingress-sticky.yml | kubectl apply -f -
~~~
### We can just test rewrite easily using /red/ path on blue deployment:
~~~
http --session red ${INGRESS_IP}:${INGRESS_HTTP_PORT}/text Host:red.example.com

http --session red ${INGRESS_IP}:${INGRESS_HTTP_PORT}/text Host:red.example.com

~~~
Results will show same red application pod everytime we use same session.

## __NOTES:__

### We can use curl too passing Cookie session id
~~~
curl -vvv --resolve red.example.com:${INGRESS_HTTP_PORT}:${INGRESS_IP}-b "red_svc_id=<red_svc_id VALUE>" http://red.example.com:${INGRESS_HTTP_PORT}/text
~~~

-------


### Dashboard will be accesible on http://127.0.0.1:8080/dashboard.html after nginx-ingress pod port forwarding:
~~~
kubectl port-forward $(kubectl get pods -n nginx-ingress -o name) 8080:8080 --namespace=nginx-ingress
~~~
