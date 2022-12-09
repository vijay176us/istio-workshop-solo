# Lab 2 :: Automating config with ArgoCD and Argo Rollouts

ArgoCD is a collection of tools for keeping Kubernetes clusters in sync with sources of configuration (like Git repositories), and automating updates to configuration when there is new code to deploy.

In this lab, we will show how to store your Istio configuration in Git and have it automatically apply to your cluster.

# Argo Rollouts and Istio

Argo Rollouts integrates with the Istio Service Mesh for traffic shaping. It can automatically, gradually increment the weights in Istio's VirtualService to slowly send traffic to a new version of your application. While incrementing, it can monitor the prometheus metrics provided by Istio to ensure that the new version is performing adequetly. If the metrics do not match your defined success criteria, Argo Rollouts automatically rolls back the version.

To get started, deploy the httpbin application. Argo Rollouts uses the `Rollout` CRD _instead_ of the your traditional Kubernetes `Deployment` CRD. This resource defines the strategy for updates:

Rollout:
```
  strategy:
    canary:
      # analysis will be performed in background, while rollout is progressing through its steps
      analysis:
        startingStep: 1   # index of step list, of when to start this analysis
        templates:
        - templateName: istio-success-rate
        args:             # arguments allow AnalysisTemplates to be re-used
        - name: service
          value: helloworld
        - name: namespace
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
      trafficRouting:
        istio:
          virtualService:
            name: httpbin-vsvc
            routes:
            - primary
          destinationRule:
            name: rollout-destrule    # required
            canarySubsetName: canary  # required
            stableSubsetName: stable  # required
      steps:
      - setWeight: 20
      - pause: {duration: 30s}
      - setWeight: 40
      - pause: {duration: 30s}
      - setWeight: 60
      - pause: {duration: 30s}
      - setWeight: 80
      - pause: {duration: 30s}
```

Analysis:

While the rollout traverses thru the steps of shifting the traffic by updating the weights, an analysis is being run that can query prometheus for metrics and determine if the new version is running as expected.

```
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: istio-success-rate
spec:
  # this analysis template requires a service name and namespace to be supplied to the query
  args:
  - name: service
  - name: namespace
  metrics:
  - name: success-rate
    initialDelay: 30s
    interval: 20s
    successCondition: result[0] > 0.95
    provider:
      prometheus:
        address: http://prometheus.istio-system:9090
        query: >+
          sum(irate(istio_requests_total{
            reporter="source",
            destination_service=~"{{args.service}}.{{args.namespace}}.svc.cluster.local",
            response_code!~"5.*"}[40s])
          )
          /
          sum(irate(istio_requests_total{
            reporter="source",
            destination_service=~"{{args.service}}.{{args.namespace}}.svc.cluster.local"}[40s])
          )
```
The above query is checking to see if the requests per second that are successful (response code not equal 5xx) drops below successCondition of 95%. If it does drop below, the analysis will fail and traffic will revert back to the previous stable version. Let's get started.

## Install Argo CD & Argo Rollouts

```
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.12/samples/addons/prometheus.yaml


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

Visit the EXTERNAL-IP of the `argocd-server` service in another tab.
```
kubectl get svc argocd-server -n argocd
```

Login using username: `admin` and password: `admin` .


# Deploy an application and it's Istio resrouces

1. Select **CREATE APPLICATION**
2. Set **Application Name** to `helloworld`
3. Set **Project** to `default`
4. Set **Sync Policy** to Automatic
5. Set **Repository URL** to `https://github.com/solo-io/gitops-library`
5. Set **Path** to `helloworld/overlay/app/argo-rollout/namespace/default/`
6. Set **Cluster URL** to `https://kubernetes.default.svc`
7. Click **Create** at the top
This should deploy an helloworld application to the default namespace. Check the pods:

```
kubectl get pods
```

Send traffic to the application thru the Ingress Gateway:

```bash
GATEWAY_IP=$(kubectl get svc -n istio-ingress istio-ingressgateway -o jsonpath="{.status.loadBalancer.ingress[0].ip}")
```

```bash
curl http://$GATEWAY_IP/hello
```
Expected output:
```
Hello version: v1, instance: helloworld-6856bf9865-wxkzl
```


# Perform the update

Let's update the helloworld to V2 by updating the image to `docker.io/istio/examples-helloworld-v2`

Normally, you would update the `Rollout` in GitHub and have Argo sync the changes. However, because this is a lab environment, let's update the `Rollout` directly in the cluster.

```
kubectl argo rollouts set image helloworld helloworld=docker.io/istio/examples-helloworld-v2
kubectl argo rollouts get rollout helloworld
```

Wait a few seconds and repeatedly send traffic:
```
for i in {1..200}; do curl http://$GATEWAY_IP/hello; sleep 2; done
```

You should see a gradual shift of traffic from v1 to v2
