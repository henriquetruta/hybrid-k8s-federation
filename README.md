# Hybrid Kubernetes Federation

Here, we'll setup a Kubernetes federation having the control plane running on a Google Cloud Engine instance and using Google DNS.
After that, we'll create a local on-prem cluster (in a VM, in my case) and make it join the Federation.

If you don't know the basics of Kubernetes Federation, you can take a look at this official [doc](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/federation.md).

NOTE: We use here an insecure method of accessing the local cluster, and it's just meant for test and dev purposes.

## What you need to have

This tutorial assumes you have:

* A running k8s local cluster. If you don't, create one following these steps [here](https://kubernetes.io/docs/getting-started-guides/kubeadm/)
* Your local API server has a public IP
* A federation running on Google Cloud Engine. Follow these steps [here](http://blog.kubernetes.io/2016/12/cluster-federation-in-kubernetes-1.5.html). You just need to create the clusters. Running an application is optional at this point
* `kubectl` and `kubefed` properly installed

## How to setup it

At this point, you should be able to access the federation control plane API by doing commands like:
`kubectl --context=federation get clusters`

We will need access to this Federation context to create most of the resources. This is adapted from the official [documentation](https://kubernetes.io/docs/admin/federation/#registering-kubernetes-clusters-with-federation).

From now on, assume the cluster we'll add is going to be called `local` and its api-server is located can be accessed through public IP `123.456.789.1` and local IP `10.0.0.1`.

### Create secret

Federation needs to have access to your local cluster. To do so, we need to create a secret that will give access to the api-server. In the api-server machine, there is a file `/etc/kubernetes/admin.conf` which should do the work. Then:

* Copy it to the a machine that has access to the federation. An example of this file can be found here at this repo
* Name it `kubeconfig`
* Change the private IP address at port 6443 to the public address at 8001, using normal HTTP, e.g. `http://123.456.789.1:8001`
* Get the context which the federation is running, i.e. the cluster that runs the control plane. In my case, it's the US-east GCE cluster
* Take a look at the federation namespace. In my case, it's `federation-system`, and if you followed the GCE tutorial, yours should be too
* To create the secret, run:

```bash
kubectl create secret generic local --context=gke_kubetesting-us-east1-b_gce-us-east1-b --namespace=federation-system --from-file=kubeconfig
```

Check if it was created

```bash
$ kubectl get secret --context=gke_kubetesting-us-east1-b_gce-us-east1-b --namespace=federation-system
NAME                                       TYPE                                  DATA      AGE
...
local                                      Opaque                                1         1m
...
```

### Proxy your local API

By default, kubernetes API server is not accessible externally. The easier way to do so, is by proxying it. To do so, `ssh` into the machine that runs your local `api-server` and run:

```bash
$ kubectl proxy --address="10.0.0.1" --disable-filter --port=8001 &
Starting to serve on 10.0.0.1:8001
```

You should see a warning telling you that the request filter was disabled, which is only for development purposes. If you intend to use this as a production environment, you must *always* enable secure authentication.

At this point, you should be able to simply curl your `api-server` using the public IP and see something like:

```bash
$ curl 123.456.789.1:8001
{
  "paths": [
    "/api",
    "/api/v1",
    "/apis",
    "/apis/apps",
    "/apis/apps/v1beta1",
    "/apis/authentication.k8s.io",
    "/apis/authentication.k8s.io/v1beta1",
    "/apis/authorization.k8s.io",
    "/apis/authorization.k8s.io/v1beta1",
    "/apis/autoscaling",
    "/apis/autoscaling/v1",
    "/apis/batch",
    "/apis/batch/v1",
    "/apis/batch/v2alpha1",
    "/apis/certificates.k8s.io",
    "/apis/certificates.k8s.io/v1alpha1",
    "/apis/extensions",
    "/apis/extensions/v1beta1",
    "/apis/policy",
    "/apis/policy/v1beta1",
    "/apis/rbac.authorization.k8s.io",
    "/apis/rbac.authorization.k8s.io/v1alpha1",
    "/apis/storage.k8s.io",
    "/apis/storage.k8s.io/v1beta1",
    "/healthz",
    "/healthz/poststarthook/bootstrap-controller",
    "/healthz/poststarthook/extensions/third-party-resources",
    "/healthz/poststarthook/rbac/bootstrap-roles",
    "/logs",
    "/metrics",
    "/swaggerapi/",
    "/ui/",
    "/version"
  ]
```


### Create cluster

Next step is to create a cluster in the federation, i.e. let the federation know your local cluster exists. To do so, we need to create a `Cluster` resource.

Take a look at the `fed_cluster.yaml` file. All you need to do is:

* Change the `name` attributes to `local` (or whatever name you've chosen)
* Change the `serverAdress` value to the same you've put on the `kubeconfig` (`123.456.789.1:8001` in our case)
* Run:

```bash
kubectl --context=federation create -f fed_cluster.yaml
# Wait a few seconds and then, see if it was created
$ kubectl --context=federation get clusters
NAME                   STATUS    AGE
...
local                  Ready     42s
```

If it's not showing the status as `Ready`, you need to see the `federation-controller-manager` logs. See Debugging section.


### Create ingress configmap in local

Looks like this is a bug from federation, that this property should've been replicated across all new clusters, but it's not, so we need to manually create a ConfigMap that refers to the Ingress used in the federation. To do so, access any context available in the federation and do:

```bash
kubectl get cm -n  kube-system ingress-uid -o yaml > ingress_uid_cm.yaml
```

This file should look like the one available in this repo. After you generate it, copy the `ingress_uid_cm.yaml` to your new local cluster and create this resource there:

```bash
kubectl create -f ingress_uid_cm.yaml
```

### Label zones and region

Now, we should label every node on the local cluster to have a proper region and zone name. In my case, I'm just labeling them as `br-northeast`. To do so, execute it for every node in your cluster:

```bash
kubectl label nodes <node_name> failure-domain.beta.kubernetes.io/zone=<region>
kubectl label nodes <node_name> failure-domain.beta.kubernetes.io/region=<region>
```

Get your nodes list by doing `kubectl get nodes` at the local cluster.

### Updating kube-DNS

Once you’ve registered your cluster with the federation, you’ll need to update KubeDNS so that your cluster can route federation service requests. In Kubernetes 1.5 or later, you must pass the --federations flag to kube-dns via the kube-dns config map. In my case, my federation is called `federation` and the domain I've put in the beginning of Google cloud configuration is `kubetest.com`. So, my configmap looks like `dns_configmap.yaml`.
Copy it to your local cluster and just run:

```bash
kubectl create -f dns_configmap.yaml
```

### Setting up the context

The last step is to create a context, from which the federation will be able to access the new cluster, as well as you would be able to do it by using the CLI with `--context=local`. To do so, go to the machine you've been running this whole process and run `kubectl config view`. If you see something, it means that you have a file in `~/kube/config` that has the info to access each cluster. You must edit this file and add two attributes: a cluster and a context.

Your file should look like this:

```YAML
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0t---
    server: https://one_ip
  name: federation
- cluster:
    certificate-authority-data: LS0t---
    server: https://some_other_ip
  name: gke_kubetesting-asia-east1-a_gce-asia-east1-a
...
contexts:
- context:
    cluster: federation
    user: federation
  name: federation
- context:
    cluster: gke_kubetesting-asia-east1-a_gce-asia-east1-a
    user: gke_kubetesting-asia-east1-a_gce-asia-east1-a
  name: gke_kubetesting-asia-east1-a_gce-asia-east1-a
...
current-context: gke_kubetesting-us-east1-b_gce-us-east1-b
kind: Config
preferences: {}
users:
- name: federation
  user:
    client-certificate-data: LS0---
    client-key-data: LS0t--
    username: admin
...
```

Just do the following:

#### Cluster

Add this at the `clusters` section, replacing for your public IP:

```YAML
- cluster:
    server: http://123.456.789.1:8001
  name: local
```

It's important to notice that when using secure connections, the certificate or other auth method would be necessary.

#### Context

Add this at the `contexts` section:

```YAML
- context:
    cluster: local
    user: admin
  name: local
```

NOTE: You can also update this using `kubectl config`.

## Debugging

When some error happens, the best way to see what's going on is to search the `federation-controller-manager` logs. It is run in a pod inside the master node of the cluster that runs the `federation control plane`. In my case, it's on the us-east region.

You can get the pod by doing replacing the context of your control plane:
`kubectl --context=gke_kubetesting_us-east1-b_gce-us-east1-b get pods -n federation-system`.

Then, run `kubectl --context=gke_kubetesting_us-east1-b_gce-us-east1-b logs federation-controller-manager-<ID> -n federation-system -f`, and the log will be streamed.

## Remarks

Again, this should not be used in a production envorinment, as it disables auth at local clusters. The creation of context is pretty simple too.

Any thoughts, feel free to submit a PR or email me at henrique@lsd.ufcg.edu.br or talk to htruta at k8s slack.