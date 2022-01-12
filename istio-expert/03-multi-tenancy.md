# Lab 3 :: Multi-tenancy

In Istio, a tenant is a group of users that share common access and priviledges for a set of deployed workloads. Tenants are like teams, and can be used to provide a level of isolation between different teams.

Istio supports three different types of tenancy models:
- Namespace tenancy
- Cluster tenancy
- Mesh tenancy

In this lab, we will dive into namespace tenancy where 3 teams each have their own hostname, deployed workloads, and Istio configuration while sharing the same cluster. What are running within each team should not impact any other team, unless there are service dependencies among the teams.

## Introduce our teams

Below are our gateway team and A, B, C teams and their responsibilities:

The gateway team owns the configuration of the Istio ingress gateway such as what ports to expose on the gateway service, Istio gateway resources and their associated resources.

Team A owns `web-api` service which is also exposed on the Istio ingress gateway as `web-api.istioinaction.io`. The `web-api` service is deployed in the web-api namespace. Team A is a producer for the `web-api.istioinaction.io` host name and a consumer of team B's `recommendation` service.

Team B owns the `recommendation` and `purchase-history` service. The recommendation service is consumed by the `web-api` service from Team A, along with some mobile applications thus it is also exposed on the Istio ingress gateway as recommendation.istioinaction.io. The `recommendation` service is deployed in the recommendation namespace and the `purchase-history` service is deployed in the purchase-history namespace. The `recommendation` service from team B is a producer for the `recommendation.istioinaction.io` host name and team A's `web-api` service. The `purchase-history` service is a private service where it is only produced/consumed for services within team B, where the `recommendation` service calls the `purchase-history` service. Nothing outside of team B should be able to view or call the `purchase-history` service.

Team C doesn't consume any other services, it produces the `ratings` service for some mobile applications via Istio ingress gateway as ratings.istioinaction.io but doesn't produce anything for team A or B to consume.


TODO: add a diagram for the gateway team and team A, B and C

The lab below assumes you have Istio 1.12.0 installed with revision 1-12-0 with istio ingress gateway in the istio-ingress namespace.
### Set up the Kubernetes services for teams

Let us clean up some deployments from the prior lab as we need to re-organize them as teams:

```
kubectl delete deployment web-api purchase-history-v1 recommendation sleep -n istioinaction
kubectl delete service web-api purchase-history recommendation sleep -n istioinaction
```

Create the namespaces for team A/B/C:

```
# team A:
kubectl create namespace web-api
kubectl label namespace web-api istio.io/rev=1-12-0

# team B:
kubectl create namespace recommendation
kubectl create namespace purchase-history
kubectl label namespace recommendation istio.io/rev=1-12-0
kubectl label namespace purchase-history istio.io/rev=1-12-0

# team C:
kubectl create namespace ratings
kubectl label namespace ratings istio.io/rev=1-12-0

```

For team A (assuming team A has access to the `web-api` namespace), deploy the `web-api` service in the `web-api` namespace:

```
kubectl apply -f labs/03/web-api.yaml -n web-api
```

For team B (assuming team B has access to the `recommendation` and `purchase-history` namespaces), deploy the `recommendation` service in the `recommendation` namespace, and the `purchase-history` service in the `purchase-history` namespace`:

```
kubectl apply -f labs/03/recommendation.yaml -n recommendation
kubectl apply -f labs/03/purchase-history-v1.yaml -n purchase-history
```

For team C (assuming team B has access to the `ratings` namespace), deploy `ratings` service in the `ratings` namespace:

```
kubectl apply -f labs/03/ratings.yaml -n ratings
```
## Gateway and Virtual Service Isolation

The gateway team defines a Gateway resource for each team (A, B and C). Let us display and dive into these gateway resources.
```
cat labs/03/web-api-gw-https.yaml
```

The `web-api-gateway` resource is for team A. For the `hosts` field where it defines that when traffic arrives for `web-api.istioinaction.io`, the routing behavior must be defined in the `web-api` namespace which is owned by team A.  

```
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: web-api-gateway
  namespace: istio-ingress
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - "web-api/web-api.istioinaction.io"
    tls:
      mode: SIMPLE
      credentialName: web-api-cert
```

Review the gateway resources for team B and C, you'll notice the similar host configuration along with a unique credential for the hostname:

```
cat labs/03/recommendation-gw-https.yaml
cat labs/03/ratings-gw-https.yaml
```
### VS merging


Alternatively, the gateway team can define 1 gateway resource with all the configuration inside it.

### VirtualService delegation


### Listener ports for the gateway resource

Note: mention exposing ports on gateway as needed

### Global host name

Cover recommendation service using global hostname such as recommendation.istioinaction.io for client in the mesh and client out of the mesh.

## Service Isolation

### private service and public service

What if only 1 service is public and others are private services?

Sharing public services among teams?  To which team?

## Config Isolation

### Config isolation to specific ports

TODO: Talk about GM Workspace

## Next Lab