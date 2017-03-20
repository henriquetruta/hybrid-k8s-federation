# Hybrid Kubernetes Federation

Here, we'll setup a Kubernetes federation having the control plane running on a Google Cloud Engine instance and using Google DNS.
After that, we'll create a local on-prem cluster (in a VM, in my case) and make it join the Federation.

NOTE: We use here an insecure method of accessing the local cluster, and it's just meant for test and dev purposes.

## What you need to have

This tutorial assumes you have:

* A running k8s local cluster. If you don't, create one following these steps [here](https://kubernetes.io/docs/getting-started-guides/kubeadm/)
* Your local API server has a public IP
* A federation running on Google Cloud Engine. Follow these steps [here](http://blog.kubernetes.io/2016/12/cluster-federation-in-kubernetes-1.5.html). Here, you just need to create the clusters. Running an application is optional
* `kubectl` and `kubefed` properly installed

## How to setup it

### Create secret

### Create service

### Proxy (insecure)

kubectl proxy --address="SERVER PRIVATE IP" --disable-filter

### Create ingress configmap in local

save result from kubectl get cm -n  kube-system ingress-uid -o yaml
kubectl create -f ingress_uid_cm.yaml

### Label zones and region

kubectl label nodes henrique-k8s-master2 failure-domain.beta.kubernetes.io/zone=br-northeast
kubectl label nodes henrique-k8s-master2 failure-domain.beta.kubernetes.io/region=br-northeast

