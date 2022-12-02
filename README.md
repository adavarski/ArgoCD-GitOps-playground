## GitOps: CI/CD using GitHub Actions and ArgoCD on Kubernetes 

**Objective:** 
Implementing GitOps with GitHub Actions (GitOps CI) and ArgoCD (GitOps CD) to deploy Helm Charts on Kubernetes. One key ingredient to enable GitOps is to have the CI separate from CD. Once CI execution is done, the artifact will be pushed to the repository and ArgoCD will be taking care of the CD. 

## Demo (simple, monorepo, KIND)

**Note**: Very simple monorepo for CI & CD. See **[CI/CD GitOps Notes](./README-Notes.md)** for Production-Like Deployment Strategy. 

In this simple demo we use KIND default k8s namespace for dev environment (no Sandbox/Production namespaces or separate Sandbox/Production k8s clusters and no Production-Like Deployment Strategy). Continuous Deployment is ideal for lower environments (i.e. Development) and can be triggered by a PR merge, push or even a simple commit to the application source code repository. We will using GitHub Actions to build Docker Image of the application and then push the image to DockerHub repository (a new docker image with "git_hash" tag will be created when we PR merge/push/commit to "main" branch), and then update the version of the new image in the Helm Chart present in the Git repo. As soon as there is some change in the Helm Chart, ArgoCD detects it and starts rolling out and deploying the new Helm chart in the Kubernetes cluster.

### GitHub Actions & ArgoCD pipeline flow (GitOps pipeline flow):

<img src="pictures/gitops-workflow-KIND.png?raw=true" width="1000">

### Create DockerHub TOKEN & Setup GitHub Actions repository secrets

Add DOCKERHUB_USER and DOCKERHUB_TOKEN secrets to application repository. You need to create a token in DockerHub first to use for the DOCKERHUB_TOKEN. Also create a GitHub token called ARGOCD and tick repo scope (dedicated for ArgoCD).

### Install docker, kubectl, helm, etc.

### Instal KIND & Create cluster (CNI=Calico, Enable ingress)
```
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.14.0/kind-linux-amd64 && chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind
cd KIND/
kind create cluster --name gitops --config cluster-config.yaml
kind get kubeconfig --name="gitops" > admin.conf
export KUBECONFIG=./admin.conf 
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
kubectl -n kube-system set env daemonset/calico-node FELIX_IGNORELOOSERPF=true
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

### Install argocd (in-cluster)
```
helm repo add argo-cd https://argoproj.github.io/argo-helm
helm dep update argocd/argocd/
kubectl create ns argocd
helm install argo-cd argocd/argocd -n argocd
```

### Create argocd app 
```
kubectl apply -f argocd/apps/demo.yaml -n argocd
```
**Note**: **[GitHub Private Repos](./README-private-repos.md)**
**Note**: **[Docker Private Registry](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/)**  

### Log to argocd
```
kubectl -n argocd port-forward svc/argo-cd-argocd-server 8080:443
Log as admin
To get password:
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### Check Argo UI
<img src="pictures/ArgoCD-app.png?raw=true" width="900">
<img src="pictures/ArgoCD-app-details.png?raw=true" width="900">

### Check app
```
kubectl -n default port-forward svc/test-helm-example 9999:80
```
<img src="pictures/GitOps-app.png?raw=true" width="900">


### Clean environment

```
kind delete cluster --name=gitops
```
