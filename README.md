## GitOps: CI/CD using GitHub Actions and ArgoCD on Kubernetes 

**Objective:** 
Implementing GitOps with GitHub Actions (GitOps CI) and ArgoCD (GitOps CD) to deploy Helm Charts on Kubernetes. One key ingredient to enable GitOps is to have the CI separate from CD. Once CI execution is done, the artifact will be pushed to the repository and ArgoCD will be taking care of the CD. 

<img src="pictures/gitops-demo-all.webp?raw=true" width="1000">

## Demo1 (simple, monorepo, KIND: single cluster)

**Note**: Very simple monorepo for CI & CD (no separate app/s:CI and config:CD repo/s:ArgoCD apps manifests). See **[CI/CD GitOps Notes](./README-Notes.md)** for Production-Like Deployment Strategy. 

In this simple demo we use KIND "default" k8s namespace for DEV environment and "prod" namespaces for PRODUCTION environment (no separate Staging/Production k8s clusters and no Production-Like Deployment Strategy). 

- DEV environment: Continuous Deployment is ideal for lower environments (i.e. Development) and can be triggered by a PR merge, push or even a simple commit to the application source code repository. We will using GitHub Actions to build Docker Image of the application and then push the image to DockerHub repository (a new docker image with "git_hash" tag will be created when we PR merge/push/commit to "main" branch), and then update the version of the new image in the Helm Chart present in the Git repo (values.yaml). As soon as there is some change in the Helm Chart, ArgoCD detects it and starts rolling out and deploying the new Helm chart in the Kubernetes cluster ("default" k8s ns). 

- PRODUCTON environment: On "git tag", GitHub Actions build Docker Image of the application and then push the image to DockerHub repository (a new docker image with tag = "git tag <tagname>" will be created when we create tag to "main" branch, example: 1.0.0), and then update the version of the new image in the Helm Chart present in the Git repo (values-prod.yaml). As soon as there is some change in the Helm Chart, ArgoCD detects it and starts rolling out and deploying the new Helm chart in the Kubernetes cluster ("prod" k8s ns).

**Note**: KIND (k8s in docker) is a tool for running local Kubernetes clusters using Docker container “nodes”. KIND was primarily designed for testing Kubernetes itself, but may be used for local development or CI. Ref: **[KIND](./KIND/README.md)** 
 
### GitHub Actions & ArgoCD pipeline flow (GitOps pipeline flow):

<img src="pictures/gitops-workflow-KIND.png?raw=true" width="1000">

### Create DockerHub TOKEN & Setup GitHub Actions repository secrets

Add DOCKERHUB_USER and DOCKERHUB_TOKEN secrets to application repository. You need to create a token in DockerHub first to use for the DOCKERHUB_TOKEN. Also create a GitHub token called ARGOCD and tick repo scope (dedicated for ArgoCD).

### Install docker, kubectl, helm, etc.

### Instal KIND & Create cluster (CNI=Calico, Enable ingress)
```
$ curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.14.0/kind-linux-amd64 && chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind
$ cd KIND/
$ kind create cluster --name gitops --config cluster-config.yaml
$ kind get kubeconfig --name="gitops" > admin.conf
$ export KUBECONFIG=./admin.conf 
$ kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
$ kubectl -n kube-system set env daemonset/calico-node FELIX_IGNORELOOSERPF=true
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

### Install argocd (in-cluster) and argocd CLI
```
$ helm repo add argo-cd https://argoproj.github.io/argo-helm
$ helm dep update argocd/argocd/
$ kubectl create ns argocd
$ helm install argo-cd argocd/argocd -n argocd
$ curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64 && chmod +x ./argocd-linux-amd64 && sudo mv argocd-linux-amd64 /usr/local/bin/argocd
```

### ArgoCD ingress 
```
Note: --insecure in argocd/argocd/values.yaml 

Setup /etc/hosts file 

$ grep argocd /etc/hosts
127.0.0.1	localhost argocd.local

$ kubectl apply -f argocd/argocd/manifests/argocd-ingress.yaml
```

### Create argocd app (DEV)
```
$ kubectl apply -f argocd/apps/demo.yaml -n argocd
```

### Create argocd app (PROD)
```
Note Pre: Create git tag on "main" branch (example: 1.0.0) to buid docker image and update helm chart via GitHub Action (prod.yaml workflow)  
$ kubectl create ns prod
$ kubectl apply -f argocd/apps/prod.yaml -n argocd

```

**Note**: **[GitHub Private Repos](./README-private-repos.md)**

**Note**: **[Docker Private Registry](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/)** (Helm Charts: imagePullSecrets) 

**Note**: Helm 3 supports OCI for package distribution. **Using Helm OCI charts and registries:**
 

- [Use OCI-based registries](https://helm.sh/docs/topics/registries/)
- [Deploy Helm OCI charts with ArgoCD](https://drake0103.medium.com/deploy-helm-oci-charts-with-argocd-583699c7d739)
- [How to deploy with ArgoCD when Helm values and Chart are in different repositories](https://mixi-developers.mixi.co.jp/argocd-with-helm-fee954d1003c)


### Log to argocd via ArgoCD UI 
```
Browser: http://argocd.local

Log as admin
To get password:
$ kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

Note: port-forward example 
$ kubectl -n argocd port-forward svc/argo-cd-argocd-server 8080:443
Browser: http://localhost:8080
```

### Log to argocd via ArgoCD CLI
```
$ kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
$ argocd login --insecure argocd.local --grpc-web
Username: admin
Password: 
'admin:login' logged in successfully
Context 'argocd.local' updated 
 
$ argocd version
argocd: v2.5.2+148d8da
  BuildDate: 2022-11-07T17:06:04Z
  GitCommit: 148d8da7a996f6c9f4d102fdd8e688c2ff3fd8c7
  GitTreeState: clean
  GoVersion: go1.18.7
  Compiler: gc
  Platform: linux/amd64
argocd-server: v2.3.2+ecc2af9
  BuildDate: 2022-03-23T00:40:57Z
  GitCommit: ecc2af9dcaa12975e654cde8cbbeaffbb315f75c
  GitTreeState: clean
  GoVersion: go1.17.6
  Compiler: gc
  Platform: linux/amd64
  Kustomize Version: v4.4.1 2021-11-11T23:36:27Z
  Helm Version: v3.8.0+gd141386
  Kubectl Version: v0.23.1
  Jsonnet Version: v0.18.0

```

### Check apps via Argo UI & ArgoCD CLI & kubectl
 
<img src="pictures/ArgoCD-apps-dev-and-prod.png?raw=true" width="900">
<img src="pictures/ArgoCD-app-dev.png?raw=true" width="900">
<img src="pictures/ArgoCD-app-prod.png?raw=true" width="900">

```
$ argocd app get test
Name:               argocd/test
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          default
URL:                https://argocd.example.com/applications/test
Repo:               https://github.com/adavarski/ArgoCD-GitOps-playground
Target:             HEAD
Path:               helm
SyncWindow:         Sync Allowed
Sync Policy:        Automated (Prune)
Sync Status:        Synced to HEAD (5ee3f51)
Health Status:      Healthy

GROUP  KIND        NAMESPACE  NAME               STATUS  HEALTH   HOOK  MESSAGE
       Service     default    test-helm-example  Synced  Healthy        service/test-helm-example unchanged
apps   Deployment  default    test-helm-example  Synced  Healthy        deployment.apps/test-helm-example configured

$ argocd app get test-prod
Name:               argocd/test-prod
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          prod
URL:                https://argocd.example.com/applications/test-prod
Repo:               https://github.com/adavarski/ArgoCD-GitOps-playground
Target:             HEAD
Path:               helm
Helm Values:        values-prod.yaml
SyncWindow:         Sync Allowed
Sync Policy:        Automated (Prune)
Sync Status:        Synced to HEAD (5ee3f51)
Health Status:      Healthy

GROUP  KIND        NAMESPACE  NAME                    STATUS  HEALTH   HOOK  MESSAGE
       Service     prod       test-prod-helm-example  Synced  Healthy        service/test-prod-helm-example created
apps   Deployment  prod       test-prod-helm-example  Synced  Healthy        deployment.apps/test-prod-helm-example created

$ kubectl get po -n default
NAME                                 READY   STATUS    RESTARTS   AGE
test-helm-example-777ff787cf-kqv4g   1/1     Running   0          12m
$ kubectl get po -n prod
NAME                                     READY   STATUS    RESTARTS   AGE
test-prod-helm-example-6d875dd6c-rrz2f   1/1     Running   0          41m


$ kubectl get pods -n prod -o jsonpath="{.items[*].spec.containers[*].image}" |\
> tr -s '[[:space:]]' '\n' |\
> sort |\
> uniq -c
      1 davarski/gitops-demo:1.0.0

$ kubectl get pods -n default -o jsonpath="{.items[*].spec.containers[*].image}" |\
> tr -s '[[:space:]]' '\n' |\
> sort |\
> uniq -c
      1 davarski/gitops-demo:main-f7f0db3


```

### Check app
```
$ kubectl -n default port-forward svc/test-helm-example 9999:80
```
<img src="pictures/GitOps-app.png?raw=true" width="900">


### Clean environment

```
$ kind delete cluster --name=gitops
```
 
## Demo2 (simple, monorepo, KIND: multiple cluster)

Deployment Strategy (Production-Like):

<img src="pictures/Deployment-Strategy-KIND.png?raw=true" width="1000">

```
$ kind create cluster --name gitops
$ kind get kubeconfig --name="prod" > kind-prod.conf
$ kind get kubeconfig --name="gitops" > kind-giops.conf
$ export KUBECONFIG="./kind-prod.conf:./kind-giops.conf"
$ kubectl config view --flatten > ./kind-clusters.conf

$ export KUBECONFIG=./kind-clusters.conf
$ kubectl config set current-context kind-prod
$ kubectl get endpoints
NAME         ENDPOINTS         AGE
kubernetes   172.18.0.4:6443   129m
$ cp kind-clusters.conf kind-clusters.conf.ORIG
$ vi kind-clusters.conf (Getting ArgoCD working: argocd cluster add)
$ diff kind-clusters.conf kind-clusters.conf.ORIG
9c9
<     server: https://172.18.0.4:6443
---
>     server: https://127.0.0.1:37295
$ kubectl config set current-context kind-gitops
Property "current-context" set.
$ kubectl config current-context
kind-gitops
$ kubectl config get-contexts
CURRENT   NAME          CLUSTER       AUTHINFO      NAMESPACE
*         kind-gitops   kind-gitops   kind-gitops   
          kind-prod     kind-prod     kind-prod 
$ kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://127.0.0.1:37213
  name: kind-gitops
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://172.18.0.4:6443
  name: kind-prod
contexts:
- context:
    cluster: kind-gitops
    user: kind-gitops
  name: kind-gitops
- context:
    cluster: kind-prod
    user: kind-prod
  name: kind-prod
current-context: kind-gitops
kind: Config
preferences: {}
users:
- name: kind-gitops
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
- name: kind-prod
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED

 $ argocd cluster add kind-prod
WARNING: This will create a service account `argocd-manager` on the cluster referenced by context `kind-prod` with full cluster level privileges. Do you want to continue [y/N]? y
INFO[0010] ServiceAccount "argocd-manager" created in namespace "kube-system" 
INFO[0010] ClusterRole "argocd-manager-role" created    
INFO[0010] ClusterRoleBinding "argocd-manager-role-binding" created 
INFO[0015] Created bearer token secret for ServiceAccount "argocd-manager" 
Cluster 'https://172.18.0.4:6443' added

$ kubectl apply -f argocd/apps/prod-cluster.yaml -n argocd
application.argoproj.io/test-prod-cluster created
$ kubectl config set current-context kind-prod
Property "current-context" set.
$ kubectl get po
NAME                                              READY   STATUS    RESTARTS   AGE
test-prod-cluster-helm-example-84b4bbc848-xvs2q   1/1     Running   0          4m39s

```

### Check app via Argo UI & ArgoCD CLI & kubectl on production cluster

<img src="pictures/ArgoCD-multi-cluster-setup-Clusters.png?raw=true" width="900">
<img src="pictures/ArgoCD-multi-cluster-setup-apps.png?raw=true" width="900">
<img src="pictures/ArgoCD-multi-cluster-setup-prod-app.png?raw=true" width="900">

```
$ argocd app get test-prod-cluster
Name:               argocd/test-prod-cluster
Project:            default
Server:             kind-prod
Namespace:          default
URL:                https://argocd.example.com/applications/test-prod-cluster
Repo:               https://github.com/adavarski/ArgoCD-GitOps-playground
Target:             HEAD
Path:               helm
Helm Values:        values-prod.yaml
SyncWindow:         Sync Allowed
Sync Policy:        Automated (Prune)
Sync Status:        Synced to HEAD (04660d6)
Health Status:      Healthy

GROUP  KIND        NAMESPACE  NAME                            STATUS  HEALTH   HOOK  MESSAGE
       Service     default    test-prod-cluster-helm-example  Synced  Healthy        service/test-prod-cluster-helm-example created
apps   Deployment  default    test-prod-cluster-helm-example  Synced  Healthy        deployment.apps/test-prod-cluster-helm-example created


$ kubectl get pods -n default -o jsonpath="{.items[*].spec.containers[*].image}" |tr -s '[[:space:]]' '\n' |sort |uniq -c
      1 davarski/gitops-demo:1.0.0
```
### Clean environment
```
$ kind delete cluster --name=gitops
$ kind delete cluster --name=prod
```
