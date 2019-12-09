---
layout:     post
title:      Configure Memory and CPU Quotas for a Namespace
subtitle:   Hi Kubernetes
date:       2019-12-06
author:     JL
header-img: img/post-bg-debug.png
catalog: true
tags:
    - Kubernetes
    
---


# Create Namespace 
    kubectl create namespace quota-mem-cpu-example
# Create a ResourceQuota
quota.yaml:
```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: mem-cpu-demo
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
```
kubectl apply -f quota.yaml --namespace=quota-mem-cpu-example
#Create a pod
pod1.yaml 
```
apiVersion: v1
kind: Pod
metadata:
  name: quota-mem-cpu-demo
spec:
  containers:
  - name: quota-mem-cpu-demo-ctr
    image: nginx
    resources:
      limits:
        memory: "800Mi"
        cpu: "800m" 
      requests:
        memory: "600Mi"
        cpu: "400m"
```
kubectl apply -f pod1.yaml  --namespace=default-mem-example 

success

# Attempt to create a second Pod
pod2.yaml 
```
apiVersion: v1
kind: Pod
metadata:
  name: quota-mem-cpu-demo-2
spec:
  containers:
  - name: quota-mem-cpu-demo-ctr
    image: nginx
    resources:
      limits:
        memory: "700Mi"
        cpu: "800m" 
      requests:
        memory: "600Mi"
        cpu: "400m"
```
kubectl apply -f pod2.yaml  --namespace=default-mem-example

pod1 800 + pod2 700 > 1G  : failed
```
Error from server (Forbidden): error when creating "https://k8s.io/examples/admin/resource/quota-mem-cpu-pod-2.yaml": pods "quota-mem-cpu-demo-2" is forbidden: exceeded quota: mem-cpu-demo, requested: limits.memory=1Gi,requests.memory=700Mi, used: limits.memory=800Mi,requests.memory=600Mi, limited: limits.memory=1Gi,requests.memory=1Gi
```
#Update  ResourceQuota
1G -> 2G
quota.yaml:
```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: mem-cpu-demo
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 2Gi
    limits.cpu: "2"
    limits.memory: 2Gi
```
kubectl edit -f quota.yaml --namespace=quota-mem-cpu-example

#Attempt to create a second Pod

kubectl apply -f pod2.yaml  --namespace=default-mem-example

success