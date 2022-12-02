### For private repos we can use:
- ArgoCD UI (Settings/Repositories) to add privite repo (user: GitHub Username & password: ARGOCD GitHub token generated previously) 
- argocd CLI (argocd repo add https://github.com/GITHUB_USER/ArgoCD-GitOps-playground --username <GITHUB_USER> --password <GITHUB_ARGOCD_TOKEN>
- teminal/kubectl 
- etc.

Ref: https://argo-cd.readthedocs.io/en/stable/user-guide/private-repositories/ 

```
# Example: via kubectl 

### Encode the token with base64 in terminal (ARGOCD GitHub token generated previously)
$ echo -n ghp_2XXXXXXXXXX | base64
ZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZ

### And user name:
$ echo -n adavarski | base64
YYYYYYY

### Add a Kubernetes Secret in the ArgoCD Namespace: secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: github-access
  namespace: argocd
data:
  username: YYYYYYY
  password: ZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZ

kubectl apply -f secret.yaml -n argocd

### Add its use to the argocd-cm ConfigMap:
kubectl get cm/argocd-cm -n argocd -o yaml > argocd-cm.yaml

Edit argocd-cm.yaml and add following lines:
...
  repositories: |
    - url: https://github.com/adavarski/ArgoCD-GitOps-playground
      passwordSecret:
        name: github-access
        key: password
      usernameSecret:
        name: github-access
        key: username
...
kubectl apply -f argocd-cm.yaml -n argocd

Check via ArgoCD CLI: argocd repocreds list

### Create ArgoCD app 
kubectl apply -f argocd/apps/demo.yaml -n argocd
```
