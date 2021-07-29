# Lab 2 :: Automating config with ArgoCD and Flagger

ArgoCD is a collection of tools for keeping Kubernetes clusters in sync with sources of configuration (like Git repositories), and automating updates to configuration when there is new code to deploy.

In this lab, we will show how to store your Istio configuration in Git and have it automatically apply to your cluster.

## Install Argo CD & Argo Rollouts

```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd patch secret argocd-secret \
  -p '{"stringData": {
    "admin.password": "$2a$10$ldvEUwliowstaKXsWbK5b.mvN79pN8yFqQzq1Vq50fIEnzHGhljCa",
    "admin.passwordMtime": "'$(date +%FT%T%Z)'"
  }}'
kubectl -n argocd get pods
```

```
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
```

# ArgoCD UI

```
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

Visit the EXTERNAL-IP of the `argocd-server` service in your browser
```
kubectl get svc argocd-server -n argocd
```

Login using username: `admin` and password: `admin` .


# Argo Rollouts and Istio

Argo Rollouts integrates with the Istio Service Mesh for traffic shaping.

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: rollouts-demo-vsvc
spec:
  gateways:
  - rollouts-demo-gateway
  hosts:
  - rollouts-demo.local
  http:
  - name: primary  # Should match spec.strategy.canary.trafficRouting.istio.virtualService.routes
    route:
    - destination:
        host: rollouts-demo-stable  # Should match spec.strategy.canary.stableService
      weight: 100
    - destination:
        host: rollouts-demo-canary  # Should match spec.strategy.canary.canaryService
      weight: 0
```

# Deploy an application and it's Istio resrouces

1. Select **CREATE APPLICATION**
2. Set **Repository URL** to `https://github.com/argoproj/argocd-example-apps.git`
3. Set **Path** to `bookinfo`
4. Click **Create** at the top

# Perform the update

```
kubectl argo rollouts set image rollouts-demo rollouts-demo=argoproj/rollouts-demo:yellow
kubectl argo rollouts get rollout rollouts-demo
```

