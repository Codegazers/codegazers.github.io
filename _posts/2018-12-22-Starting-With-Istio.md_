---
layout: post
title:  "Starting With Istio"
date:   NOT_PUBLISHED_2018-12-22 11:30:00 +0100
categories: [kubernetes]
---

minikube start --memory=6000 --cpus=2 --kubernetes-version=v1.11.5 --vm-driver kvm2

curl -L https://git.io/getLatestIstio | sh -

export PATH="$PATH:/home/zero/Source/istio-1.0.5/bin"

cd istio-1.0.5/

kubectl apply -f install/kubernetes/helm/istio/templates/crds.yaml

kubectl apply -f install/kubernetes/istio-demo.yaml

kubectl api-versions | grep admissionregistration

kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml

kubectl get services

kubectl get pods

kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml

export INGRESS_HOST=$(minikube ip)

export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')

export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT

curl -o /dev/null -s -w "%{http_code}\n" http://${GATEWAY_URL}/productpage

kubectl apply -f samples/bookinfo/networking/destination-rule-all.yaml

kubectl get destinationrules -o yaml

kubectl port-forward -n istio-system $(kubectl get pod -n istio-system -l app=jaeger -o jsonpath='{.items[0].metadata.name}') 16686:16686 &

http://localhost:16686/
