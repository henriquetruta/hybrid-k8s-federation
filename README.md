# Hybrid Kubernetes Federation

### Create secret

### Create service

### Proxy (insecure)
kubectl proxy --address="SERVER PRIVATE IP" --disable-filter

### Create ingress configmap in local
save result from kubectl get cm -n  kube-system ingress-uid -o yaml
kubectl create -f ingress_uid_cm.yaml

### Label zones and region:
kubectl label nodes henrique-k8s-master2 failure-domain.beta.kubernetes.io/zone=br-northeast
kubectl label nodes henrique-k8s-master2 failure-domain.beta.kubernetes.io/region=br-northeast

