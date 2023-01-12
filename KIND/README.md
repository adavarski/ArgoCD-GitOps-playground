## KIND local environment

References: 
- Calico : https://alexbrand.dev/post/creating-a-kind-cluster-with-calico-networking/ 
- Ingress: https://kind.sigs.k8s.io/docs/user/ingress/ 
- LoadBalancer:  https://kind.sigs.k8s.io/docs/user/loadbalancer/
- Local Registry: https://kind.sigs.k8s.io/docs/user/local-registry/
- Private Registry: https://kind.sigs.k8s.io/docs/user/private-registries/
- Auditing: https://kind.sigs.k8s.io/docs/user/auditing/
- Kind in CI: https://github.com/kind-ci/examples

<img src="./KIND-diagram.png?raw=true" width="800">

KIND Source: https://github.com/kubernetes-sigs/kind (https://github.com/kubernetes-sigs/kind/tree/main/images/base)


### Install KIND 
```
### Install docker, kubectl, etc.

### Instal KIND 

$ curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.14.0/kind-linux-amd64 && chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind (Note: k8s v1.24.0)

$ curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.17.0/kind-linux-amd64 && chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind (Note: k8s v1.25.3)
```
### Create cluster (CNI=Calico, Enable ingress)

```
$ cat cluster-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
- role: worker
networking:
  disableDefaultCNI: true
  podSubnet: 192.168.0.0/16


$ kind create cluster --name gitops --config cluster-config.yaml
```
Example Output (kind:v0.14.0/k8s:1.24.0 & kind:v0.17.0/k8s:v1.25.3):

```
$ kind create cluster --name gitops --config cluster-config.yaml
Creating cluster "gitops" ...
 ✓ Ensuring node image (kindest/node:v1.24.0) 🖼 
 ✓ Preparing nodes 📦 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing StorageClass 💾 
 ✓ Joining worker nodes 🚜 
Set kubectl context to "kind-gitops"
You can now use your cluster with:

kubectl cluster-info --context kind-gitops

Have a nice day! 👋

$ kind create cluster --name gitops --config cluster-config.yaml
Creating cluster "gitops" ...
 ✓ Ensuring node image (kindest/node:v1.25.3) 🖼 
 ✓ Preparing nodes 📦 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing StorageClass 💾 
 ✓ Joining worker nodes 🚜 
Set kubectl context to "kind-gitops"
You can now use your cluster with:

kubectl cluster-info --context kind-gitops

Have a question, bug, or feature request? Let us know! https://kind.sigs.k8s.io/#community 🙂


$ docker ps -a
CONTAINER ID        IMAGE                   COMMAND                  CREATED             STATUS              PORTS                                                                 NAMES
b1f0d34833c0        kindest/node:v1.24.0    "/usr/local/bin/entr…"   24 minutes ago      Up 24 minutes       0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp, 127.0.0.1:36409->6443/tcp   devsecops-control-plane
b2dfe90a04cb        kindest/node:v1.24.0    "/usr/local/bin/entr…"   24 minutes ago      Up 24 minutes                                                                             devsecops-worker
baea6d91feb6        kindest/node:v1.24.0    "/usr/local/bin/entr…"   27 minutes ago      Created                                                                                   kind-control-plane
ff1679781bbc        kindest/node:v1.24.0    "/usr/local/bin/entr…"   27 minutes ago      Created                                                                                   kind-worker

$ docker ps -a
CONTAINER ID        IMAGE                   COMMAND                  CREATED             STATUS              PORTS                                                                 NAMES
7b1116751270        kindest/node:v1.25.3    "/usr/local/bin/entr…"   32 minutes ago      Up 32 minutes       0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp, 127.0.0.1:40769->6443/tcp   gitops-control-plane
a63e1717ef2c        kindest/node:v1.25.3    "/usr/local/bin/entr…"   32 minutes ago      Up 32 minutes                                                                             gitops-worker

$ kind get kubeconfig --name="gitops" > admin.conf
$ export KUBECONFIG=./admin.conf 
$ kubectl get node
NAME                      STATUS     ROLES           AGE     VERSION
gitops-control-plane   NotReady   control-plane   10m     v1.24.0
gitops-worker          NotReady   <none>          9m51s   v1.24.0

$ kubectl get pods -n kube-system
NAME                                              READY   STATUS    RESTARTS   AGE
coredns-6d4b75cb6d-8ltl7                          0/1     Pending   0          5m4s
coredns-6d4b75cb6d-crfxv                          0/1     Pending   0          5m4s
etcd-devsecops-control-plane                      1/1     Running   0          5m18s
kube-apiserver-devsecops-control-plane            1/1     Running   0          5m19s
kube-controller-manager-devsecops-control-plane   1/1     Running   0          5m19s
kube-proxy-4kcgw                                  1/1     Running   0          5m5s
kube-proxy-phc5j                                  1/1     Running   0          5m2s
kube-scheduler-devsecops-control-plane            1/1     Running   0          5m19s

$ kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
configmap/calico-config created
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/caliconodestatuses.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipreservations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/kubecontrollersconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org created
clusterrole.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrolebinding.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrole.rbac.authorization.k8s.io/calico-node created
clusterrolebinding.rbac.authorization.k8s.io/calico-node created
daemonset.apps/calico-node created
serviceaccount/calico-node created
deployment.apps/calico-kube-controllers created
serviceaccount/calico-kube-controllers created
poddisruptionbudget.policy/calico-kube-controllers created

$ kubectl -n kube-system set env daemonset/calico-node FELIX_IGNORELOOSERPF=true (kind:v0.14.0/k8s:1.24.0 only)
daemonset.apps/calico-node env updated

$ kubectl get pods -n kube-system
NAME                                              READY   STATUS    RESTARTS   AGE
calico-kube-controllers-6766647d54-5jpkc          1/1     Running   0          2m48s
calico-node-7457x                                 1/1     Running   0          41s
calico-node-ggrfn                                 1/1     Running   0          21s
coredns-6d4b75cb6d-8ltl7                          1/1     Running   0          14m
coredns-6d4b75cb6d-crfxv                          1/1     Running   0          14m
etcd-devsecops-control-plane                      1/1     Running   0          14m
kube-apiserver-devsecops-control-plane            1/1     Running   0          14m
kube-controller-manager-devsecops-control-plane   1/1     Running   0          14m
kube-proxy-4kcgw                                  1/1     Running   0          14m
kube-proxy-phc5j                                  1/1     Running   0          14m
kube-scheduler-devsecops-control-plane            1/1     Running   0          14m

$ kubectl -n kube-system get pods | grep calico-node
calico-node-7457x                                 1/1     Running   0          63s
calico-node-ggrfn                                 1/1     Running   0          43s

$ kubectl api-resources
$ kubectl api-resources|head

$ kubectl get node -o wide  (kind:v0.14.0/k8s:1.24.0)
NAME                      STATUS   ROLES           AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE       KERNEL-VERSION     CONTAINER-RUNTIME
devsecops-control-plane   Ready    control-plane   4h56m   v1.24.0   172.17.0.3    <none>        Ubuntu 21.10   5.0.0-32-generic   containerd://1.6.4
devsecops-worker          Ready    <none>          4h56m   v1.24.0   172.17.0.2    <none>        Ubuntu 21.10   5.0.0-32-generic   containerd://1.6.4

$ kubectl get node -o wide (kind:v0.17.0/k8s:v1.25.3)
NAME                   STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
gitops-control-plane   Ready    control-plane   27m   v1.25.3   172.17.0.3    <none>        Ubuntu 22.04.1 LTS   5.0.0-32-generic   containerd://1.6.9
gitops-worker          Ready    <none>          26m   v1.25.3   172.17.0.2    <none>        Ubuntu 22.04.1 LTS   5.0.0-32-generic   containerd://1.6.9


```

### Ingress NGINX (other options: Ingress Kong/Contour/Ambassador)

```
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
namespace/ingress-nginx created
serviceaccount/ingress-nginx created
serviceaccount/ingress-nginx-admission created
role.rbac.authorization.k8s.io/ingress-nginx created
role.rbac.authorization.k8s.io/ingress-nginx-admission created
clusterrole.rbac.authorization.k8s.io/ingress-nginx created
clusterrole.rbac.authorization.k8s.io/ingress-nginx-admission created
rolebinding.rbac.authorization.k8s.io/ingress-nginx created
rolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
configmap/ingress-nginx-controller created
service/ingress-nginx-controller created
service/ingress-nginx-controller-admission created
deployment.apps/ingress-nginx-controller created
job.batch/ingress-nginx-admission-create created
job.batch/ingress-nginx-admission-patch created
ingressclass.networking.k8s.io/nginx created
validatingwebhookconfiguration.admissionregistration.k8s.io/ingress-nginx-admission created

$ kubectl wait --namespace ingress-nginx --for=condition=ready pod --selector=app.kubernetes.io/component=controller --timeout=90s
pod/ingress-nginx-controller-5458c46d7d-7mblv condition met
$ kubectl get ns
NAME                 STATUS   AGE
default              Active   48m
ingress-nginx        Active   65s
kube-node-lease      Active   49m
kube-public          Active   49m
kube-system          Active   49m
local-path-storage   Active   48m

$ kubectl get po -n ingress-nginx
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-5slj4        0/1     Completed   0          97s
ingress-nginx-admission-patch-fc2rz         0/1     Completed   1          97s
ingress-nginx-controller-5458c46d7d-7mblv   1/1     Running     0          97s
```
### Clean environment

```
$ kind delete cluster --name=gitops

$ kind delete cluster --name=gitops
Deleting cluster "devsecops" ...
[1]+  Killed                  linkerd viz dashboard
```

