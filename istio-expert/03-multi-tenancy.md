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

The lab below assumes you have Istio 1.12.1 installed with revision 1-12-1 with istio ingress gateway in the istio-ingress namespace.  If you don't, you can setup Istio using the steps below with `istioctl` version 1.12.1:

```bash
kubectl create ns istio-system
kubectl apply -f ../deploy-istio/labs/02/istiod-service.yaml
istioctl install -y -n istio-system -f ../deploy-istio/labs/02/control-plane.yaml --revision 1-12-1
kubectl create namespace istio-ingress
istioctl install -y -n istio-ingress -f ../deploy-istio/labs/04/ingress-gateways.yaml --revision 1-12-1
```
### Set up the Kubernetes services for teams

Let us clean up some deployments from the prior lab as we need to re-organize them as teams:

```bash
kubectl delete deployment web-api purchase-history-v1 recommendation sleep -n istioinaction
kubectl delete service web-api purchase-history recommendation sleep -n istioinaction
```

Create the namespaces for team A/B/C:

```bash
# team A:
kubectl create namespace web-api-ns
kubectl label namespace web-api-ns istio.io/rev=1-12-1

# team B:
kubectl create namespace recommendation-ns
kubectl create namespace purchase-history-ns
kubectl label namespace recommendation-ns istio.io/rev=1-12-1
kubectl label namespace purchase-history-ns istio.io/rev=1-12-1

# team C:
kubectl create namespace ratings-ns
kubectl label namespace ratings-ns istio.io/rev=1-12-1

```

For team A (assuming team A has access to the `web-api-ns` namespace), deploy the `web-api` service in the `web-api` namespace:

```bash
kubectl apply -f labs/03/web-api.yaml -n web-api-ns
```

For team B (assuming team B has access to the `recommendation-ns` and `purchase-history-ns` namespaces), deploy the `recommendation` service in the `recommendation-ns` namespace, and the `purchase-history` service in the `purchase-history-ns` namespace`:

```bash
kubectl apply -f labs/03/recommendation.yaml -n recommendation-ns
kubectl apply -f labs/03/purchase-history-v1.yaml -n purchase-history-ns
```

For team C (assuming team B has access to the `ratings-ns` namespace), deploy `ratings` service in the `ratings-ns` namespace:

```bash
kubectl apply -f labs/03/ratings.yaml -n ratings-ns
```
## Gateway and Virtual Service Isolation

### Gateway resources
The gateway team defines a Gateway resource for each team (A, B and C). Let us display and dive into these gateway resources.

```bash
cat labs/03/web-api-gw-https.yaml
```

You'll get the followng output:

```text
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
    - "web-api-ns/web-api.istioinaction.io"
    tls:
      mode: SIMPLE
      credentialName: web-api-cert
```

The `web-api-gateway` resource is for team A. For the `hosts` field where it defines that when traffic arrives for `web-api.istioinaction.io`, the routing behavior must be defined in the `web-api-ns` namespace which is owned by team A.  

Review the gateway resources for team B and C, you'll notice the similar host configuration along with unique credential for the hostname:

```bash
cat labs/03/recommendation-gw-https.yaml
cat labs/03/ratings-gw-https.yaml
```

Acting as the gateway team, apply these gateway resources to the `istio-ingress` namespace. 

```bash
kubectl apply -f labs/03/web-api-gw-https.yaml
kubectl apply -f labs/03/recommendation-gw-https.yaml
kubectl apply -f labs/03/ratings-gw-https.yaml
```

What else is missing? Run `istioctl analyze` command to get some hints:

```bash
istioctl analyze -n istio-ingress
```

From the output, you can see we need to create the credentials for each gateway resource:

```text
Error [IST0101] (Gateway istio-ingress/ratings-gateway) Referenced credentialName not found: "ratings-cert"
Error [IST0101] (Gateway istio-ingress/recommendation-gateway) Referenced credentialName not found: "recommendation-cert"
Error [IST0101] (Gateway istio-ingress/web-api-gateway) Referenced credentialName not found: "web-api-cert"
```

You can create the `ratings-cert`, `recommendation-cert` and `web-api-cert` credentials using cert-manager, refer to the Deploy Istio workshop for details on understanding these steps: 

```bash
# prep for the installation of cert manager:
kubectl create namespace cert-manager
helm repo add jetstack https://charts.jetstack.io
helm repo update
# do the actual installation:
helm install cert-manager jetstack/cert-manager --namespace cert-manager --version v1.2.0 --create-namespace --set installCRDs=true
# wait for cert-manager pods to come up
kubectl wait --for=condition=Ready pod --all -n cert-manager
# install root certs/key (just for the lab, you'd use Vault, LetsEncrypt, or your own PKI in production)
kubectl create -n cert-manager secret tls cert-manager-cacerts --cert labs/03/certs/ca/root-ca.crt --key labs/03/certs/ca/root-ca.key
# deploy a cluster issuer
kubectl apply -f labs/03/cert-manager/ca-cluster-issuer.yaml 
# ask cert manager to provision certs
kubectl apply -f labs/03/cert-manager/web-api-istioinaction-io-cert.yaml 
kubectl apply -f labs/03/cert-manager/recommendation-istioinaction-io-cert.yaml 
kubectl apply -f labs/03/cert-manager/ratings-istioinaction-io-cert.yaml 
```

Check on the secrets created by the cert manager:

```bash
kubectl get secrets -n istio-ingress
```

Confirm the secrets are created:

```text
NAME                                               TYPE                                  DATA   AGE
default-token-g5km4                                kubernetes.io/service-account-token   3      2d3h
istio-ingressgateway-service-account-token-zv2sb   kubernetes.io/service-account-token   3      2d3h
ratings-cert                                       kubernetes.io/tls                     3      28h
recommendation-cert                                kubernetes.io/tls                     3      28h
web-api-cert                                       kubernetes.io/tls                     3      28h
```

Now the cert should be loaded in the istio ingress gateway and marked as `ACTIVE`, for example:

```bash
istioctl pc secret deploy/istio-ingressgateway -n istio-ingress 
```

You should see these secrets are loaded to the `istio-ingressgateway` deployment running in the `istio-ingress` namespace:

```text
RESOURCE NAME                        TYPE           STATUS     VALID CERT     SERIAL NUMBER                               NOT AFTER                NOT BEFORE
kubernetes://ratings-cert            Cert Chain     ACTIVE     true           37366564123948328554505903164932401801      2022-04-13T16:50:38Z     2022-01-13T16:50:38Z
kubernetes://web-api-cert            Cert Chain     ACTIVE     true           256089257652381406128593923667139355802     2022-04-13T16:50:37Z     2022-01-13T16:50:37Z
kubernetes://recommendation-cert     Cert Chain     ACTIVE     true           41739643360456883013502421916594136332      2022-04-13T16:50:37Z     2022-01-13T16:50:37Z
```

Run the analyzer command again to ensure no error is reported:

```bash
istioctl analyze -n istio-ingress
```

### VirtualService resources

When the traffic arrives at the Gateway for any of our host (e.g. web-api.istioinaction.io:443), you need route configuration to define where the traffic goes which you can define it in a `VirtualService` resource. Who would own the VirtualService resource?  A common model is that each team owns the VirtualService resource which is what Team A and B will follow.  Another model is to leverage VirtualService delegation so that team can learn as minimal as possible on the resource. Team C will follow this model.
#### VirtualService and Gateway resources owned by different team

Team A and B owns the virtual service resources while team C owns only the delegated virtual service resource.

You can review the `web-api-vs.yaml` file:

```bash
cat labs/03/web-api-vs.yaml
```

From the output, note the resource is for the `web-api` namespace, and the value for `gateways`, which refers to the `web-api-gateway` resource in the `istio-ingress` namespace:

```output
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: web-api-vs
  namespace: web-api
spec:
  hosts:
  - "web-api.istioinaction.io"
  gateways:
  - istio-ingress/web-api-gateway
  http:
  - route:
    - destination:
        host: web-api.web-api-ns.svc.cluster.local
```

Review the content for `labs/03/recommendation-vs.yaml`. Acting as team A and B, deploy the virtual service files for the `web-api-gateway` (to the `web-api` namespace) and `recommendation-gateway` (to the `recommendation` namespace):

```bash
kubectl apply -f labs/03/web-api-vs.yaml
kubectl apply -f labs/03/recommendation-vs.yaml
```

#### VirtualService delegation
Team C prefers to learn as little Istio resources as possible. The gateway team plans to use Virtual Service delegation feature in Istio to handle this sceanrio where the gateway team can delegate the routing behavior to be defined from another VirtualService, which is often called delegated VirtualService.

You can review the `ratings-vs.yaml` file for the gateway team:

```bash
cat labs/03/ratings-vs.yaml
```

From the output, note the value for `gateways`, which refers to the `web-api-gateway` resource in the `istio-ingress` namespace:

```output
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: ratings-vs
  namespace: istio-ingress
spec:
  hosts:
  - "ratings.istioinaction.io"
  gateways:
  - ratings-gateway
  http:
  - delegate:
      name: ratings-delegated-vs
      namespace: ratings-ns
```

Review the delegated VirtualService for team C:

```bash
cat labs/03/ratings-delegated-vs.yaml
```

Note that the name and namespace of the resource, which should match exactly what is in the `delegate` field in the `ratings-vs` VirtualService resource. This delegated virtual service resource controls the route behavior for traffic arriving at `ratings.istioinaction.io` without requiring any change in its parent virtual service resource (e.g. `ratings`).

```text
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: ratings-delegated-vs
  namespace: ratings-ns
spec:
  http:
  - route:
    - destination:
        host: ratings.ratings-ns.svc.cluster.local
```

Acting as the gateway team, deploy the `ratings-vs` virtual service file to the `istio-ingress` namespace:

```bash
kubectl apply -f labs/03/ratings-vs.yaml
```

Acting as team C, deploy the `ratings-delegated-vs` virtual service file to the `ratings-ns` namespace:

```bash
kubectl apply -f labs/03/ratings-delegated-vs.yaml
```

Test all three urls to ensure they work:

```bash
curl --cacert ./labs/03/certs/ca/root-ca.crt -H "Host: web-api.istioinaction.io" https://web-api.istioinaction.io --resolve web-api.istioinaction.io:443:$GATEWAY_IP
curl --cacert ./labs/03/certs/ca/root-ca.crt -H "Host: recommendation.istioinaction.io" https://recommendation.istioinaction.io --resolve recommendation.istioinaction.io:443:$GATEWAY_IP
curl --cacert ./labs/03/certs/ca/root-ca.crt -H "Host: ratings.istioinaction.io" https://ratings.istioinaction.io --resolve ratings.istioinaction.io:443:$GATEWAY_IP
```

You'll get greeting messages from the `web-api.istioinaction.io` and `recommendation.istioinaction.io` hosts, but you'll notice you'll get an empty reply from the `ratings.istioinaction.io` host.  Run the analyzer command to troubleshoot this:

```bash
istioctl analyze -n istio-ingress
```

The output is very helpful, which indicated that there is no `VirtualService` found thus the gateway doesn't know where to route when traffic arrives at `ratings.istioinaction.io`:

```text
Warning [IST0132] (VirtualService istio-ingress/ratings) one or more host [ratings.istioinaction.io] defined in VirtualService istio-ingress/ratings not found in Gateway istio-ingress/ratings-gateway.
```

Review the `ratings-gw-https.yaml` file:

```bash
cat labs/03/ratings-gw-https.yaml
```

From the output, do you see what may be wrong here?

```text
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: ratings-gateway
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
    - "ratings-ns/ratings.istioinaction.io"    
    tls:
      mode: SIMPLE
      credentialName: ratings-cert
```

If you recall, you used `VirtualService` delegation for the `ratings-gateway` so the `ratings.istioinaction.io` hostname is defined in the `ratings-vs.yaml` deployed in the `istio-ingress` namespace. The `ratings-delegated-vs.yaml` file doesn't configure `hosts` or `gateways` field.  The fix is to update `ratings-ns/ratings.istioinaction.io` to `ratings.istioinaction.io` because the `ratings-vs.yaml` is deployed in the same namespace as the `ratings-gateway`.

TODO: draw a diagram here for VS delegation

Deploy the correct `ratings-gateway` with `hosts` set to `ratings.istioinaction.io`:

```bash
kubectl apply -f labs/03/ratings-gw-https-correct-host.yaml
```

Run the analyzer command to confirm the previous host not found warning is gone:

```bash
istioctl analyze -n istio-ingress
```

Access the `ratings.istioinaction.io` url, you will get the `Hello From Ratings!` greeting:

```bash
curl --cacert ./labs/03/certs/ca/root-ca.crt -H "Host: ratings.istioinaction.io" https://ratings.istioinaction.io --resolve ratings.istioinaction.io:443:$GATEWAY_IP
```

### Global host name

When the `web-api` service calls the `recommendation` service, it uses the `http://recommendation.recommendation-ns:8080` as the value of its `UPSTREAM_URIS`.  If you don't recall where this is defined, check out the `labs/03/web-api.yaml` file.

When you run the curl command from outside of the Kubernetes cluter to access the `recommendation` service, you used `recommendation.istioinaction.io` because that is the hostname the gateway team has configured on the Istio's ingress gateway to route the traffic to the `recommendation` service.

Wouldn't be nice if you can have the global host name `recommendation.istioinaction.io` for the `recommendation` service, regardless of where the clients are calling it whether they are from inside of the mesh or outside of the mesh? You can use Istio's support for global host name to configure this.

TODO: a diagram for this, clients inside the mesh and clients outside of mesh.

First, we need to enable the `ISTIO_META_DNS_AUTO_ALLOCATE` configuration in Istio control plane's `proxyMetadata`, so that Istio can automatically allocat IP addresses for ServiceEntrys that do not explicitly define one.  Open up the `control-plane.yaml`:

```bash
cat labs/03/control-plane.yaml
```

The `ISTIO_META_DNS_AUTO_ALLOCATE: "true"` is added to the proxyMetadata configuration:

```text
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: control-plane
spec:
  profile: minimal
  meshConfig:
    defaultConfig:
      proxyMetadata:
        ISTIO_META_DNS_CAPTURE: "true"
        ISTIO_META_DNS_AUTO_ALLOCATE: "true"
    enablePrometheusMerge: true
```

Deploy the update to your Istio control plane:

```bash
istioctl install -y -n istio-system -f labs/03/control-plane.yaml --revision 1-12-1
```

Second, review the ServiceEntry resource in the `recommendation-se.yaml` file:

```bash
cat labs/03/recommendation-se.yaml
```

You can see the host name for `recommendation.istio.io`, along with `workloadSelector` that selects which workload the hosts are for, and port `8080` for the `app: recommendation` workload. Note there is no `addresses` field in this ServiceEntry resource.

```text
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: recommendation-se
  namespace: recommendation-ns
spec:
  hosts:
    - recommendation.istioinaction.io
  location: MESH_INTERNAL
  ports:
    - name: http
      number: 8080
      protocol: http
  resolution: STATIC
  workloadSelector:
    labels:
      app: recommendation
```

Note:  The workload selector and ports have to be correct, else, you will get the `Could not resolve host: recommendation.istioinaction.io` error.

Deploy the above ServiceEntry resource:

```bash
kubectl apply -f labs/03/recommendation-se.yaml
```

Third, update the `web-api` service to use the newly created global host name (`recommendation.istioinaction.io`), by replacing the current value of `UPSTREAM_URIS` to `http://recommendation.istioinaction.io:8080`:

```bash
cat labs/03/web-api-global-host.yaml
```

Deploy the updated `web-api` service:

```bash
kubectl apply -f labs/03/web-api-global-host.yaml -n web-api-ns
```

You can confirm the global host name is working by visiting the `web-api.istioinaction.io` url which calls `recommendation.istioinaction.io`:

```bash
curl --cacert ./labs/03/certs/ca/root-ca.crt -H "Host: web-api.istioinaction.io" https://web-api.istioinaction.io --resolve web-api.istioinaction.io:443:$GATEWAY_IP
```

You can also continue to visit the `recommendation.istioinaction.io`:

```bash
curl --cacert ./labs/03/certs/ca/root-ca.crt -H "Host: recommendation.istioinaction.io" https://recommendation.istioinaction.io --resolve recommendation.istioinaction.io:443:$GATEWAY_IP
```

Congratulations, you have setup global host name for the `recommendation` service successfully!
## Service Isolation

By default, Istio networking resources and services are visible to all services running in all namespaces that are part of the Istio service mesh. As part of the team concept, it is often desirable to have clear boundry among teams where the services and Istio resources are isolated whenever possible.

Let us review the desired configuration for team A, B, and C:

<!--
Team gateway (`istio-ingress` namespace):
- The `istio-ingressgateway` service, its Gateway resources, and VirtualService resources should be isolated within the `istio-ingress` namespace. 
-->

Team A (`web-api-ns` namespace):
- The `web-api` service, plus its VirtualService resource. 
- The `web-api` service calls the `recommendation` service and nothing else. The `web-api` service is consumed by the `istio-ingressgateway` service and should not be made available to any other teams.

Team B (`recommendation-ns` & `purchase-history-ns` namespaces):
- The `recommendation` and `purchase-history` service, plus the VirtualService and ServiceEntry resources for the `recommendation` service. 
- The `purchase-history` service doesn't call any other service. The `purchase-history` service is an internal service and should not be exposed outside of the team. 
- The `recommendations` service calls the `purchase-history` service and nothing else. The `recommendation` service is consumed by the `web-api` service and the `istio-ingressgateway` service. It should not be made available to any other teams.

Team C (`ratings-ns` namespace):
- The `ratings` service, plus its delegated VirtualService resource. 
- The `ratings` service doesn't call any other service. The `ratings` service is consumed by the `istio-ingressgateway` service and should not be made available to any other teams.

## Service consumers

The section below focuses on each team's service consumer perspectives, e.g. which other service(s) a given team calls.

Acting as team A, you want to restrict all of the services owned by the team to reach out ONLY to the current namespace (represented as `.` below), `istio-system` and `recommendation-ns` namespaces as the `web-api` service calls the `recommendation` service:

```text
apiVersion: networking.istio.io/v1beta1
kind: Sidecar
metadata:
  name: default
spec:
  egress:
  - hosts:
    - "./*"    
    - "istio-system/*"
    - "recommendation-ns/*"
```

Deploy the following Sidecar resource to the `web-api-ns`:

```bash
kubectl apply -f labs/03/sidecar-team-a.yaml -n web-api-ns
```

Acting as team B, you want to restrict all of the services owned by the team to reach out ONLY to the team's namespaces (`recommendation-ns` and `purchase-history-ns`) on port `8080`, and the `istio-system` namespace:

```
apiVersion: networking.istio.io/v1beta1
kind: Sidecar
metadata:
  name: default
spec:
  egress:
  - hosts:
    - "recommendation-ns/*"
    - "purchase-history-ns/*"
    port:
      number: 8080
      protocol: HTTP
      name: egresshttp
  - hosts:
    - "istio-system/*"
```

Deploy the above Sidecar resource to the `recommendation-ns` and `purchase-history-ns` namespaces:

```bash
kubectl apply -f labs/03/sidecar-team-b.yaml -n recommendation-ns
kubectl apply -f labs/03/sidecar-team-b.yaml -n purchase-history-ns
```

Acting as team C, you want to restrict all of the services owned by the team to reach out ONLY to the current namespace (represented as `.` below)on port `8080`, and the `istio-system` namespace:

```
apiVersion: networking.istio.io/v1beta1
kind: Sidecar
metadata:
  name: default
spec:
  egress:
  - hosts:
    - "./*"
    port:
      number: 8080
      protocol: HTTP
      name: egresshttp
  - hosts:           
    - "istio-system/*"
```

Deploy the above Sidecar resource to the `ratings-ns`:

```bash
kubectl apply -f labs/03/sidecar-team-c.yaml -n ratings-ns
```

Confirm that all services continue to work by visiting `web-api.istioinaction.io` and `ratings.istioinaction.io`:

```bash
curl --cacert ./labs/03/certs/ca/root-ca.crt -H "Host: web-api.istioinaction.io" https://web-api.istioinaction.io --resolve web-api.istioinaction.io:443:$GATEWAY_IP
curl --cacert ./labs/03/certs/ca/root-ca.crt -H "Host: ratings.istioinaction.io" https://ratings.istioinaction.io --resolve ratings.istioinaction.io:443:$GATEWAY_IP
```

Note: you may be wondering why the `sidecar-team-a.yaml` file doesn't restrict one or more given ports like the sidecar resource for team B or C's. The reason is the `web-api` deployment's sidecar proxy needs to use DNS to lookup the `recommendation.istioinaction.io` hostname, yet Istio's sidecar resource port protocol doesn't support the `DNS` type yet.

## Service producers

The section below focuses on each team's service producer perspectives, e.g. a given service or Istio resource is made *ONLY* available to the team that needs to use it.

### Istio networking resources

As explained in the [deploy Istio for production](../deploy-istio/README.md) workshop, Service owners can apply `export-To` in VirtualService, DestinationRule and ServiceEntry resources to define a list of namespaces that the Istio networking resources can be applied to.

Acting as team A, adding `export-To` to the `web-api-vs` to make it available only to the `istio-ingress` namespace:

```text
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: web-api-vs
  namespace: web-api-ns
spec:
  hosts:
  - "web-api.istioinaction.io"
  gateways:
  - istio-ingress/web-api-gateway
  exportTo:
  - istio-ingress
  http:
  - route:
    - destination:
        host: web-api.web-api-ns.svc.cluster.local
```

Deploy the updated `web-api-vs` resource:

```bash
kubectl apply -f labs/03/web-api-vs-exportto.yaml
```

Acting as team B, adding `export-To` to the `recommendation-vs` to make it available only to the `istio-ingress` namespace.

```text
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: recommendation-vs
  namespace: recommendation-ns
spec:
  hosts:
  - "recommendation.istioinaction.io"
  gateways:
  - istio-ingress/recommendation-gateway
  exportTo:
  - istio-ingress
  http:
  - route:
    - destination:
        host: recommendation.recommendation-ns.svc.cluster.local
```

Also, adding `export-To` to the `recommendation-se` to make it available only to the `web-api-ns` namespace.

```text
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: recommendation-se
  namespace: recommendation-ns
spec:
  hosts:
    - recommendation.istioinaction.io
  exportTo:
  - web-api-ns
  location: MESH_INTERNAL
  ports:
    - name: http
      number: 8080
      protocol: http
  resolution: STATIC
  workloadSelector:
    labels:
      app: recommendation
```

Deploy the updated `recommendation-vs` and `recommendation-se` resources:

```bash
kubectl apply -f labs/03/recommendation-vs-exportto.yaml
kubectl apply -f labs/03/recommendation-se-exportto.yaml
```

Acting as team C, adding `export-To` to the `ratings-delegated-vs` to make it available only to the `istio-ingress` namespace:

```text
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: ratings-delegated-vs
  namespace: ratings-ns
spec:
  exportTo:
  - istio-ingress
  http:
  - route:
    - destination:
        host: ratings.ratings-ns.svc.cluster.local
```

Deploy the updated `ratings-delegated-vs` resource:

```bash
kubectl apply -f labs/03/ratings-delegated-vs-exportto.yaml
```

Confirm that all services continue to work by visiting `web-api.istioinaction.io` and `ratings.istioinaction.io`:

```bash
curl --cacert ./labs/03/certs/ca/root-ca.crt -H "Host: web-api.istioinaction.io" https://web-api.istioinaction.io --resolve web-api.istioinaction.io:443:$GATEWAY_IP
curl --cacert ./labs/03/certs/ca/root-ca.crt -H "Host: ratings.istioinaction.io" https://ratings.istioinaction.io --resolve ratings.istioinaction.io:443:$GATEWAY_IP
```

TODO: add commands to view the isolated config.
### Kubernetes services

Service owners can declare their services' visibility via the `networking.istio.io/exportTo` annotation. This can be used as an extra step to further configure service visibility on top of the above methods.

Run the `istioctl proxy-config` cmd below to understand the Envoy clusters for the Istio ingressgateway:

```bash
istioctl pc cluster deploy/istio-ingressgateway -n istio-ingress
```

You'll see an entry for the `purchase-history` service which is not desired because Istio ingress gateway doesn't need to be aware of the service:

```text
purchase-history.purchase-history-ns.svc.cluster.local     8080      -          outbound      EDS
```

You can leverage the `networking.istio.io/exportTo` annotation to fix this. Acting as team B, update the `purchase-history` service to add the `networking.istio.io/exportTo` annotation to make the service available only to the current namespace and the `recommendation-ns` namespace:

```text
apiVersion: v1
kind: Service
metadata:
  name: purchase-history
  annotations:
    networking.istio.io/exportTo: ".,recommendation-ns" 
spec:
  selector:
    app: purchase-history
  ports:
  - name: http
    protocol: TCP
    port: 8080
    targetPort: 8080
```

Deploy the updated `purchase-history` service:

```bash
kubectl apply -f labs/03/purchase-history-service-exportto.yaml -n purchase-history-ns
```

Confirm that all related services continue to work by visiting `web-api.istioinaction.io`:

```bash
curl --cacert ./labs/03/certs/ca/root-ca.crt -H "Host: web-api.istioinaction.io" https://web-api.istioinaction.io --resolve web-api.istioinaction.io:443:$GATEWAY_IP
```

## Workspaces

While working with the above team concepts in Istio, 
## Next Lab

Congratulations, you have set up multiple teams with Istio with the proper isolation. In the next lab, we will dive into certification rotation and integrate with your own PKI.