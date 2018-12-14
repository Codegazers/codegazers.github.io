---
layout: post
title:  "Setup NGINX Plus as Ingress Controller"
date:   2018-10-15 11:30:00 +0100
categories: [kubernetes, NGINX]
---


## We will use Minikube for a Quick Demo

### We deploy complete Kong Environment with Postgresql.
~~~
curl https://raw.githubusercontent.com/Kong/kubernetes-ingress-controller/master/deploy/single/all-in-one-postgres.yaml | kubectl create -f -
~~~

### We have created 'kong' namespace so we will wait until all Kong components are ready.
~~~
kubectl get all --namespace kong
~~~
### We deploy now two deployments with their services (red-app and blue-app with red-svc and blue-svc) 
~~~
curl https://raw.githubusercontent.com/Codegazers/k8s-vagrant/master/examples/colors/colors.yml | kubectl create -f -
~~~

### We review the deployments
~~~
kubectl get all --namespace default
~~~

### We review Kong
~~~
kubectl get all --namespace kong
~~~

### For quick access we prepare some envirnment variables (for Kong admin and Kong proxy access)
~~~
export KONG_ADMIN_PORT=$(minikube service -n kong kong-ingress-controller --url --format "{{ .Port }}")
export KONG_ADMIN_IP=$(minikube service   -n kong kong-ingress-controller --url --format "{{ .IP }}")
export PROXY_IP=$(minikube   service -n kong kong-proxy --url --format "{{ .IP }}" | head -1)
export HTTP_PORT=$(minikube  service -n kong kong-proxy --url --format "{{ .Port }}" | head -1)
export HTTPS_PORT=$(minikube service -n kong kong-proxy --url --format "{{ .Port }}" | tail -1)
~~~

### Now we deploy colors-ingress ingress resource for accesing both applications
~~~
curl https://raw.githubusercontent.com/Codegazers/k8s-vagrant/master/examples/colors/colors-ingress.yml | kubectl apply -f -
~~~

### We just set up host routing
~~~
kubectl get ingress colors-ingress -o yaml
~~~

### And now we can access our applications setting host headers
~~~
http ${PROXY_IP}:${HTTP_PORT}/text Host:red.example.com

http ${PROXY_IP}:${HTTP_PORT}/text Host:blue.example.com
~~~

### We review targets because red-app has 3 replicas
~~~
http ${KONG_ADMIN_IP}:${KONG_ADMIN_PORT}/upstreams/default.red-svc.80/targets
~~~

### We now add a KongPluggin resource for reate limitting to route
~~~
echo "
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: add-ratelimiting-to-route
config:
  minute: 20
  limit_by: ip
  second: 5
" | kubectl create -f -
~~~

### We review deployed Kong Plugins
~~~
kubectl get kongplugins
~~~

### And now we apply rate limit to our deployed ingress resource
~~~
kubectl patch ingress colors-ingress \
  -p '{"metadata":{"annotations":{"rate-limiting.plugin.konghq.com":"add-ratelimiting-to-route\n"}}}'
~~~

### We can review the plugin, associated to route and routes deployed
~~~
http ${KONG_ADMIN_IP}:${KONG_ADMIN_PORT}/plugins

http ${KONG_ADMIN_IP}:${KONG_ADMIN_PORT}/routes
~~~

### Accessing to our applications we can review proxying requests and limits applied
~~~ 
http ${PROXY_IP}:${HTTP_PORT}/text Host:red.example.com

http ${PROXY_IP}:${HTTP_PORT}/text Host:blue.example.com
~~~

### We now delete ingress resource for creating separated ones
~~~ 
kubectl delete ingress colors-ingress
~~~

### Deploying different ingress resources
~~~
curl https://raw.githubusercontent.com/Codegazers/k8s-vagrant/master/examples/colors/red-ingress.yml | kubectl create -f -

curl https://raw.githubusercontent.com/Codegazers/k8s-vagrant/master/examples/colors/blue-ingress.yml | kubectl create -f -
~~~

### Review Ingress resources
~~~
kubectl get ingress
~~~

### We now add rate limit to both ingress resources
~~~
kubectl patch ingress red-ingress \
  -p '{"metadata":{"annotations":{"rate-limiting.plugin.konghq.com":"add-ratelimiting-to-route\n"}}}'

kubectl patch ingress blue-ingress \
  -p '{"metadata":{"annotations":{"rate-limiting.plugin.konghq.com":"add-ratelimiting-to-route\n"}}}'
~~~

### Applying pluging just for one service
~~~
kubectl patch svc red-svc \
  -p '{"metadata":{"annotations":{"rate-limiting.plugin.konghq.com": "add-ratelimiting-to-route\n"}}}'
~~~

### And we review now the 
~~~
http ${KONG_ADMIN_IP}:${KONG_ADMIN_PORT}/plugins
~~~

### And its access
~~~ 
http ${PROXY_IP}:${HTTP_PORT}/text Host:red.example.com
~~~