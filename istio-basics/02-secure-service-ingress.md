# Lab 2 :: Istio Ingress Gateway

In this lab, you will deploy the sample application to your Kubernetes cluster, and expose the `web-api` service to the Istio ingress gateway and configure secure access to it. The intention of the ingress gateway is to allow traffic into the mesh. If you need more sophisticated edge gateway capabilities \(request transformation, OIDC, LDAP, OPA, etc\) then use a gateway specifically built for those use cases like [Gloo Edge](https://docs.solo.io/gloo-edge/latest/).

## Prerequisites

Verify you're in the correct folder for this lab: `~/istio-workshops/istio-basics`. This lab builds on the [lab 01](01-install-istio.md) where you installed Istio and its addon components using the demo profile.

```bash
cd ~/istio-workshops/istio-basics
```

## Deploy the sample application

You will use the `web-api`, `recommendation` and `purchase-history` services built using the [fake service](https://github.com/nicholasjackson/fake-service) as our sample application. The `web-api` service calls the `recommendation` service via HTTP and the `recommendation` service calls the `purchase-history` service also via HTTP.

* Set up the `istioinaction` namespace for our services:

```bash
kubectl create ns istioinaction
```

* Deploy the `web-api`, `recommendation` and `purchase-history` services along with the `sleep` service into the `istioinaction` namespace:

```bash
kubectl apply -n istioinaction -f sample-apps/web-api.yaml
kubectl apply -n istioinaction -f sample-apps/recommendation.yaml
kubectl apply -n istioinaction -f sample-apps/purchase-history-v1.yaml
kubectl apply -n istioinaction -f sample-apps/sleep.yaml
```

* After running these commands, you should check all of the pods have reached running in the `istioinaction` namespace: 

<!--bash
kubectl wait --for=condition=Ready pod --all -n istioinaction
-->
```bash
kubectl get po -n istioinaction
```

Wait a few seconds till all of them have reached the `Running` status:

```text
NAME                                   READY   STATUS    RESTARTS   AGE
purchase-history-v1-6c8cb7f8f8-wn4dr   1/1     Running   0          22s
recommendation-c9f7cc86f-nfvmk         1/1     Running   0          22s
sleep-8f795f47d-5jfbn                  1/1     Running   0          14s
web-api-6d544cff77-drrbm               1/1     Running   0          22s
```

## Configure the inbound traffic

The Istio ingress gateway will create a Kubernetes Service of type `LoadBalancer`. Use this IP address to reach the gateway:

```bash
kubectl get svc -n istio-system
```

```text
NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                                                                      AGE
istio-ingressgateway   LoadBalancer   10.44.0.91     35.202.132.20   15021:32218/TCP,80:30062/TCP,443:30105/TCP,15012:32488/TCP,15443:30178/TCP   5m45s
istiod                 ClusterIP      10.44.10.140   <none>          15010/TCP,15012/TCP,443/TCP,15014/TCP                                        47m
```

### Note the GATEWAY\_IP env variable

{% hint style="success" %}
We use the `GATEWAY_IP` environment variable in other parts of this lab.
{% endhint %}

```bash
export GATEWAY_IP=$(kubectl get svc -n istio-system istio-ingressgateway -o jsonpath="{.status.loadBalancer.ingress[0].ip}")
export INGRESS_PORT=80
export SECURE_INGRESS_PORT=443
```

{% hint style="info" %}
If your Istio ingress gateway created a Kubernetes Service of type `NodePort`, use the following commands to set your `GATEWAY_IP`:

```text
export GATEWAY_IP=$(kubectl get po -l istio=ingressgateway -n istio-system -o jsonpath='{.items[0].status.hostIP}')
```

Set the `INGRESS_PORT` and `SECURE_INGRESS_PORT`:

```text
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
```

There is a known issue with MetalLB with MacOS. If you are running this lab on your MacBook, we recommend you to run a vagrant Ubuntu VM on your MacBook and access the `GATEWAY_IP` from your VM's terminal.
{% endhint %}

## Expose our apps

Even though you don't have our apps in the `istioinaction` namespace in the mesh yet, you can still use the Istio ingress gateway to route traffic to them. Using Istio's `Gateway` resource, you can configure what ports should be exposed, what protocol to use etc. Using Istio's `VirtualService` resource, you can configure how to route traffic from the Istio ingress gateway to our `web-api` service.

* Review the `Gateway` resource:

```bash
cat sample-apps/ingress/web-api-gw.yaml
```

In addition to the `web-api-gateway` name of the gateway resource, you can configure that port `80` with protocol `HTTP` is exposed for the `istioinaction.io` host for our Istio ingress gateway selected by the `istio: ingressgateway` selector:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: web-api-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "istioinaction.io"
```

* Review the `VirtualService` resource:

```bash
cat sample-apps/ingress/web-api-gw-vs.yaml
```

You can configure that the `VirtualService` resource is for the `istioinaction.io` host and the `web-api-gateway` gateway resource. If it is the `http` protocol, you want the requests to be routed to port `8080` of our `web-api` service in the `istioinaction` namespace.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: web-api-gw-vs
spec:
  hosts:
  - "istioinaction.io"
  gateways:
  - web-api-gateway
  http:
  - route:
    - destination:
        host: web-api.istioinaction.svc.cluster.local
        port:
          number: 8080
```

Why port number `8080` in the destination route configuration for the `web-api-gw-vs` `VirtualService` resource? Check the service port for the `web-api` service in the `istioinaction` namespace:

```bash
kubectl get service web-api -n istioinaction
```

You can see the service listens on port `8080`:

```text
NAME      TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
web-api   ClusterIP   10.1.54.188   <none>        8080/TCP   19d
```

* Apply the `Gateway` and `VirtualService` resource to expose our `web-api` service outside of the Kubernetes cluster:

```bash
kubectl -n istioinaction apply -f sample-apps/ingress/
```

The Istio ingress gateway will create new routes on the proxy that we should be able to call it from outside of the Kubernetes cluster:

<!--bash
sleep 2
-->
```bash
curl -H "Host: istioinaction.io" http://$GATEWAY_IP:$INGRESS_PORT
```

* Query the gateway configuration using the `istioctl proxy-config` command:

```bash
istioctl proxy-config routes deploy/istio-ingressgateway.istio-system
```

```text
NOTE: This output only contains routes loaded via RDS.
NAME        DOMAINS              MATCH                  VIRTUAL SERVICE
http.80     istioinaction.io     /*                     web-api-gw-vs.istioinaction
            *                    /healthz/ready*
```

If you want to see an individual route, you can ask for its output as `json` like this:

```bash
istioctl proxy-config routes deploy/istio-ingressgateway.istio-system --name http.80 -o json
```

## Secure the inbound traffic

To secure inbound traffic with HTTPS, you need a certificate with the appropriate SAN and configure the Istio ingress-gateway to use it.

* Create a TLS secret for `istioinaction.io` in the `istio-system` namespace:

```bash
kubectl create -n istio-system secret tls istioinaction-cert --key labs/02/certs/istioinaction.io.key --cert labs/02/certs/istioinaction.io.crt
```

* Update the Istio ingress-gateway to use this cert:

```bash
cat labs/02/web-api-gw-https.yaml
```

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: web-api-gateway
spec:
  selector:
    istio: ingressgateway 
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - "istioinaction.io"    
    tls:
      mode: SIMPLE
      credentialName: istioinaction-cert
```

Note, we are pointing to the `istioinaction-cert` and **that the cert must be in the same namespace as the ingress gateway deployment**. Even though the `Gateway` resource is in the `istioinaction` namespace, _the cert must be where the gateway is actually deployed_. 

* Apply the `web-api-gw-https.yaml` in the `istioinaction` namespace. Since this gateway resource is also called `web-api-gateway`, it will replace our prior `web-api-gateway` configuration for port `80`.

```bash
kubectl -n istioinaction apply -f labs/02/web-api-gw-https.yaml
```

* Call the `web-api` service through the Istio ingress-gateway on the secure `443` port: 

<!--bash
sleep 2
-->
```bash
curl --cacert ./labs/02/certs/ca/root-ca.crt -H "Host: istioinaction.io" https://istioinaction.io:$SECURE_INGRESS_PORT --resolve istioinaction.io:$SECURE_INGRESS_PORT:$GATEWAY_IP
```

If you call it on the `80` port with `http`, it will not work as we no longer configure the gateway resource to expose on port `80`.

```bash
curl -H "Host: istioinaction.io" http://$GATEWAY_IP:$INGRESS_PORT
```

## Next lab

Congratulations, you have exposed the web-api service to Istio ingress gateway securely. We'll explore adding services to the mesh in the [next lab](03-add-services-to-mesh.md).

