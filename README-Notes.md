**Definitions**:
1. Kubernetes, also known as K8s, is an open-source system for automating deployment, scaling, and management of containerized applications.
2. Argo CD is a declarative, GitOps continuous delivery tool for Kubernetes.
3. GitHub Actions Automate, customize, and execute your software development workflows right in your repository with GitHub Actions. You can discover, create, and share actions to perform any job you’d like, including CI/CD, and combine actions in a completely customized workflow.
4. Continuous Delivery vs. Continuous Deployment

This Atlassian chart best depicts the Continuous Delivery vs Continuous Deployment setup:

<img src="pictures/CI-VS-CD-Atlassian.png?raw=true" width="900">

Read more [Atlassian’s thousand words picture depicting CD vs CD](https://www.atlassian.com/continuous-delivery/principles/continuous-integration-vs-delivery-vs-deployment)

**Note (GitOps):** GitOps is one of the hottest topics in the world of DevOps as it’s an evolution of Infrastructure as Code (IaC) and a DevOps best practice that leverages Git as the single source of truth, and control mechanism for creating, updating, and deleting system architecture. More simply, GitOps is a way of implementing Continuous Deployment for cloud-native applications

## Deployment Strategy EXAMPLES : 2 OPTIONS):

### OPTION1: Recommended, because of easy maintanability** -> We are going to create two GitHub repositories. One for the application where application code and the **Helm Charts** live. The other one is for the ArgoCD's declarative configuration files (app manifests & ApplicationSets), because we don't want to use UI to manually create many resources every single time a new application pops up.

**Example Deployment strategy (CI/CD GitOps strategy):**

```
Kubernetes setup
Note: You can add more clusters or namespaces if you wish.
-------------------------------------------------------------------------------------------
CLUSTER     NAMESPACE   DESCRIPTION         DEPLOYMENT STRATEGY            IMAGE TAG
nonprod     dev         development env     fully auto CI/CD               latest
            sbox        sandbox env         fully auto CI but manual CD    semantic ver
prod        default     production env      fully auto CI but manual CD    semantic ver
-------------------------------------------------------------------------------------------
```

- dev  - Always auto deploys when a PR is merged to master branch.
- sbox - Always auto deploys when a new "tag" is released.
- prod - Never auto deploys!

We can create two GitHub repositories. One for the application where application code and the Helm Charts (I will explain why bellow!) live. The other one is for the ArgoCD's declarative configuration files because we don't want to use UI to manually create many resources every single time a new application pops up (We will use argocd command). We can use ArgoCD application manifests (or ApplicationSets), instead of creating apps on the command line (argocd cli) or ArgoCD UI and this will make us more portable in case we need to recreate them at some point.

I should make it clear that everyone would have different requirements and expectations so you are free to go your own way. You can make all stages either fully automatic (GitFlow branching would help a lot for this) or manual or partially automated/manual. You can also create a Kubernetes cluster per environment. There are pros and cons for all options.

<img src="pictures/Deployment-Strategy.png?raw=true" width="900">

GitHub Actions's responsibility: There are three actions but only two of them directly affect ArgoCD which are "merge" and "release". The "merge" action pushes a new docker image using the "latest" tag. This is for the dev CD flow. The "release" action pushes a new docker image using semantic image tag version that you have just created. This is for the sbox and prod CD flows.

**DEMO OPTION1:** 
```
### Application repository

├── .github
│   └── workflows
│       ├── merge.yaml
│       ├── pull_request.yaml
│       └── release.yaml
├── .infra
│   ├── docker
│   │   └── Dockerfile
│   └── helm
│       ├── Chart.yaml
│       ├── dev.yaml
│       ├── prod.yaml
│       ├── sbox.yaml
│       └── templates
│           ├── configmap.yaml
│           ├── deployment.yaml
│           └── service.yaml
├── .dockerignore
├── main.go
└── main_test.go

### Config repository

This is just to keep ArgoCD's declarative configurations, for now!

└── infra
    └── argocd
        └── pacman
            ├── dev.yaml
            ├── prod.yaml
            ├── project.yaml
            └── sbox.yaml
            
Prepare GitHub

Add DOCKERHUB_USER and DOCKERHUB_TOKEN secrets to application repository. You need to create a token in DockerHub first to use for the DOCKERHUB_TOKEN. e.g. 86b2f4b8-816d-4d96-9666-ac0a8066e737 Also create a GitHub token called ARGOCD and tick repo scope. e.g. ghp_816dac0a8066e737966686b2f4b84d96           
            
### Prepare Kubernetes

Create clusters

$ minikube start -p argocd --vm-driver=virtualbox --memory=2000
  Starting control plane node argocd in cluster argocd
 
$ minikube start -p nonprod --vm-driver=virtualbox
  Starting control plane node nonprod in cluster nonprod
 
$ minikube start -p prod --vm-driver=virtualbox
  Starting control plane node prod in cluster prod

Verify

$ kubectl config get-contexts
CURRENT   NAME      CLUSTER   AUTHINFO   NAMESPACE
*         argocd    argocd    argocd
          nonprod   nonprod   nonprod
          prod      prod      prod
          
$ kubectl config view

### Helm installation

Head Helm for Helm CLI installation. We use it for creating, linting and debugging our charts. You can use its commands against the charts I added to this example. As I said before, Helm is not required. You could just use Kubernetes manifests with some minor changes. However, Helm makes many things pleasant to work with.

### ArgoCD installation

Most examples install ArgoCD to application clusters but I will have a dedicated cluster for ArgoCD as I like to keep things separate. ArgoCD will then deploy applications to other clusters. 


$ kubectl config current-context
argocd
 
$ kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.2.4/manifests/install.yaml

If you wish to uninstall it you can use command below.

$ kubectl delete -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

$ kubectl get all

Let's expose it so we can access it using a browser. By the way, you can install ArgoCD as LoadBalancer and avoid port forwarding.

$ kubectl port-forward svc/argd-server 8443:443

Extract password for login using command below.

$ kubectl get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
 
// Assume you got gYzYfJ5kxzJsdZGK

You can now head https://localhost:8443/ and use admin:gYzYfJ5kxzJsdZGK to login.

Use command below to install argocd command. This will help us to avoid creating ArgoCD resources manually in UI.

$ brew tap argoproj/tap && brew install argoproj/tap/argocd

$ argocd version
argocd: v2.2.5+8f981cc.dirty
  BuildDate: 2022-02-05T05:39:12Z
  GitCommit: 8f981ccfcf942a9eb00bc466649f8499ba0455f5
  GitTreeState: dirty
  GoVersion: go1.17.6
  Compiler: gc
  Platform: darwin/amd64
FATA[0000] Argo CD server address unspecified


### Create resources

First we need to login.

$ argocd --insecure login 127.0.0.1:8443
Username: admin
Password: gYzYfJ5kxzJsdZGK
'admin:login' logged in successfully
Context '127.0.0.1:8443' updated


Add clusters. This is CLI only.
$ argocd cluster add nonprod
$ argocd cluster add prod

Add application repository.
$ argocd repo add https://github.com/you/pacman \
    --username you \
    --password ghp_816dac0a8066e737966686b2f4b84d96 \
    --type git

Create project. As you can see we are reading from the config repository.
$ argocd proj create --file ~/config/infra/argocd/pacman/project.yaml

### Create applications from the config repository . As you can see we are reading from the config repository. 

$ argocd app create --file ~/pacman/.infra/helm/dev.yaml && \
  argocd app create --file ~/pacman/.infra/helm/prod.yaml && \
  argocd app create --file ~/pacman/.infra/helm/sbox.yaml 

### OR Create applications from the application repository. As you can see we are reading from the application repository

$ argocd app create --file ~/config/infra/argocd/pacman/dev.yaml && \
  argocd app create --file ~/config/infra/argocd/pacman/prod.yaml && \
  argocd app create --file ~/config/infra/argocd/pacman/sbox.yaml    
           
```

### OPTION2: NOT Recommended, but recommended by ArgoCD best practices** -> We are going to create two GitHub repositories. One for the application where only application code live, so the application repository has no CI/CD related configuration files anymore. They all are stored in their dedicated repository (config repo). The reason for this separation is because we want to achieve a proper GitOps [practise](https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/). In short, application and config repository are all separated from each other. Note:Not reccomended, because I don't like the idea of pulling a different repository and pushing a commit to it from within a CI pipeline that is for a different repository. I just don't like the sound of this solution.

**Example Deployment strategy (CI/CD GitOps strategy)**:


- dev - Always auto deploys when a PR is merged to master branch.
- sbox - Always auto deploys when a new "tag" is released.
- prod - Never auto deploys!


**DEMO OPTION2:**

```
### Application repository

├── .github
│   └── workflows
│       ├── merge.yaml
│       ├── pull_request.yaml
│       └── release.yaml
├── docker
│   └── Dockerfile
├── .dockerignore
├── main.go
└── main_test.go

### Config repository

└── infra
    ├── argocd
    │   └── pacman
    │       ├── dev.yaml
    │       ├── prod.yaml
    │       ├── project.yaml
    │       └── sbox.yaml
    └── helm
        └── pacman
            ├── Chart.yaml
            ├── crds
            │   └── vcs
            │       ├── hash
            │       └── tag
            ├── dev.yaml
            ├── prod.yaml
            ├── sbox.yaml
            └── templates
                ├── configmap.yaml
                ├── deployment.yaml
                └── service.yaml


Prepare GitHub (I don't like the idea of pulling a different repository and pushing a commit to it from within a CI pipeline that is for a different repository):

I assume you already have an application repository called pacman.


Create Docker access token called GITHUB_ACTIONS with read, write and delete permissions. Result: 8d8937fe-753b-4f2b-96bc-b3603e4ed2b0

Create PAT called ARGOCD using "repo" scopes. Result: ghp_fhjkiuUYuy456frreSgrtry2

Create PAT called GITHUB_ACTIONS using "repo" scopes. Result: ghp_re543tgtu7hgaTHhyh6757

Create pacman repository secret called ACTIONS_TOKEN using ghp_re543tgtu7hgaTHhyh6757.

Create pacman repository secret called DOCKERHUB_TOKEN using 8d8937fe-753b-4f2b-96bc-b3603e4ed2b0.

Create pacman repository secret called DOCKERHUB_USER using your Docker username.


### Prepare Kubernetes

We will have four Kubernetes clusters which are argocd, dev, sbox and prod. ArgoCD runs on argocd cluster and deploys to other three clusters.

$ minikube start -p argocd --vm-driver=virtualbox --memory=2000
$ minikube start -p dev --vm-driver=virtualbox --memory=2000
$ minikube start -p sbox --vm-driver=virtualbox --memory=2000
$ minikube start -p prod --vm-driver=virtualbox --memory=2000

$ kubectl config get-contexts
CURRENT   NAME     CLUSTER   AUTHINFO   NAMESPACE
*         argocd   argocd    argocd
          dev      dev       dev
          prod     prod      prod
          sbox     sbox      sbox

### Prepare ArgoCD

Install ArgoCD in terminal

$ kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.2.4/manifests/install.yaml

Obtain password

$ kubectl get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
// gYzYfJ5kxzJsdZGK

Login to UI

$ kubectl port-forward svc/argd-server 8443:443
Forwarding from 127.0.0.1:8443 -> 8080
 
// Visit 127.0.0.1:8443 and use admin:gYzYfJ5kxzJsdZGK

Login CLI

$ argocd --insecure login 127.0.0.1:8443

Add clusters

$ argocd cluster add dev
$ argocd cluster add sbox
$ argocd cluster add prod

Add repository

This is optional. You could use "default".

$ argocd repo add https://github.com/you/config \
    --type git \
    --name config \
    --project config \
    --username you \
    --password ghp_fhjkiuUYuy456frreSgrtry2

Create project

This is optional. You could use "default".

$ argocd proj create --file ~/local/file/system/config/infra/argocd/project.yaml

Create applications

You could use ApplicationSet instead.

$ argocd app create --file ~/local/file/system/config/infra/argocd/pacman/dev.yaml
$ argocd app create --file ~/local/file/system/config/infra/argocd/pacman/sbox.yaml
$ argocd app create --file ~/local/file/system/config/infra/argocd/pacman/prod.yaml

```

**Note (ArgoCD: OPTION2):** ArgoCD [suggests](https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/) that it is better to keep all manifests or configuration files in a separate repository (e.g. config) for many good reasons. However, it has one problem that beats the whole purpose of fully automated deployment pipeline. For instance, if you want to provide a full automated CD like I do for dev flow, you will have to manually update something in the config repository so that ArgoCD picks it up and triggers deployment. It is because, your configurations are set to watch the config repository, not the application repository anymore. This is not engineer friendly approach because as I said everytime you introduce something to your application repository, you will also have to do something in the config. Too much hassle! Having said all that, some people suggest a solution to avoid this manual operation. It is to push a commit to config repository within CI steps. I find this a bit hacky. What if CI step breaks because of a conflict! Also I don't like the idea of pulling a different repository and pushing a commit to it from within a CI pipeline that is for a different repository. I just don't like the sound of this solution. 

Adopting ArgoCD's suggestion can be very useful if your application repository is setup as a monorepo because you might have hundreds of services in there. In such case, you will often want to deploy specific services, not all of them after every single change in the application repository. This would be crazy! So in short, see what you have and do what is best.

So, considering the GitOps best practices of separating your Application Source Code from the Application’s K8s Configuration in two Git(Hub) repositories, this is a quick guide on configuring CI/CD (Continuous Deployment) with GitHub Actions (CI) and pull based deployment via Argo CD.

This workflow can be easily converted to Continuous Delivery by modifying the GitHub Actions. Continuous Deployment is ideal for lower environments (i.e. Development) and can be triggered by a PR merge, push or even a simple commit to the application source code repository

<img src="pictures/CI-CD-flow.png?raw=true" width="900">

When triggered by a source code modification event, the GitHub Action Workflow 1 will spring into action and build, compile, and test your application and then package it in a Docker container and publish it to a container repository. The Workflow 1 passes the Docker image name and tag as input and triggers the Workflow 2, which will update the K8s deployment manifest and commit and push the changes to the Application/K8s Configuration repository. Argo CD will be pulling this change and update the application on the K8s cluster.


### Other ArgoCD Examples 
- (AWS & GCP) examples with GitHub Actions and ArgoCD:


<img src="pictures/ArgoCD-AWS-EKS.png?raw=true" width="900">


<img src="pictures/ArgoCD-GCP-GKE.png?raw=true" width="900">


- GCP example with GCP Cloud Build and ArgoCD:

<img src="pictures/ArgoCD-GCP-CloudBuild.png?raw=true" width="900">


- GitLab: We can use GitLab for GitOps flow/pipelines also, if not using GitHub for repos. 

Example:

<img src="pictures/GitOps-flow-GitLab-example.png?raw=true" width="900"> 
