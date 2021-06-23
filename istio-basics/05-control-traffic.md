# Lab 5 :: Control Traffic

You are now ready to take control of how traffic flows between services. In a Kubernetes environment, there is simple round-robin load balancing between service endpoints. While Kubernetes does support rolling upgrade, it is fairly coarse grained and is limited to moving to a new version of the service. You may find it necessary to dark launch your new version, then canary test your new version before shift all traffics to the new version completely. You will explore many of these types of features provided by Istio to control the traffic between services while increasing the resiliency between the services.

## Prerequisites

Verify you're in the correct folder for this lab: `~/istio-workshops/istio-basics`. This lab builds on the [lab 04](03-secure-services-with-istio.md) where you enforced mTLS for your services in the mesh.

```bash
cd ~/istio-workshops/istio-basics
```

## Dark Launch

You may find the v1 of the `purchase-history` service is rather boring as it always return the `Hello From Purchase History (v1)!` message. You want to make a new version of the `purchase-history` service so that it returns dynamic messages based on the result from querying an external service, for example the [JsonPlaceholder service](http://jsonplaceholder.typicode.com).

Dark launch allows you to deploy and test a new version of a service while minimizing the impact to users, e.g. you can keep the new version of the service in the dark. Using a dark launch appoach enables you to deliver new functions rapidly with reduced risk. Istio allows you to preceisely control how new versions of services are rolled out without the need to make any code change to your services or redeploy your services.

You have v2 of the `purchase-history` service ready in the `labs/05/purchase-history-v2.yaml` file:

```bash
cat labs/05/purchase-history-v2.yaml
```

The main change is the `purchase-history-v2` deployment name and the `version:v2` labels, along with the `fake-service:v2` image and the newly added `EXTERNAL_SERVICE_URL` environment variable. The `purchase-history-v2` pod establishes the connection to the external service at startup time and obtain a random response from the external service when clients call the v2 of the `purchase-history` service.

```text
apiVersion: apps/v1
kind: Deployment
metadata:
  name: purchase-history-v2
  labels:
    app: purchase-history
    version: v2
spec:
  replicas: 1
  selector:
    matchLabels:
        app: purchase-history
        version: v2
  template:
    metadata:
      labels:
        app: purchase-history
        version: v2
    spec:
      serviceAccountName: purchase-history    
      containers:
      - name: purchase-history
        image: linsun/fake-service:v2
        ports:
        - containerPort: 8080
        env:
        - name: "LISTEN_ADDR"
          value: "0.0.0.0:8080"
        - name: "NAME"
          value: "purchase-history-v2"
        - name: "SERVER_TYPE"
          value: "http"
        - name: "MESSAGE"
          value: "Hello From Purchase History (v2)!"
        - name: "EXTERNAL_SERVICE_URL"
          value: "http://jsonplaceholder.typicode.com/posts"
        imagePullPolicy: Always
```

Should you deploy the `labs/05/purchase-history-v2.yaml` to your Kubernetes cluster next? How much percentage of the traffic will visit `v1` and `v2` of the `purchase-history` services? Because both of the deployments have `replicas: 1`, you will see 50% traffic goes to v1 and 50% traffic goes to `v2`. This is not what you wanted because you haven't had chance to test `v2` in your Kubernetes cluster yet.

You can use Istio's networking resources to dark launch the `v2` of the `purchase-history` service. Virtual Service provides you with the ability to configure a list of routing rules that control how the Envoy proxies of the client routes requests to a given service within the service mesh. The client could be Istio's ingress-gateway or any of your service in the mesh. In lab 02, when the client is `istio-ingressgateway`, the virtual service is bound to the `web-api-gateway` gateway. If you recall the Kiali graph for our application from the prior labs, the client for the `purchase-history` service in your application is the `recommendation` service.

Destination rule allows you to define configurations of policies that are applied to a request after the routing rules are enforced as defined in the destination virtual service. In addition, destination rule is also used to define the set of Kubernetes pods that belong to a subset grouping, for example multiple versions of a service, which are called `subsets` in Istio.

You can review the virtual service resource for the `purchase-history` service that configures all traffic to `v1` of the `purchase-history` service:

```bash
cat labs/05/purchase-history-vs-all-v1.yaml
```

```text
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: purchase-history-vs
spec:
  hosts:
  - purchase-history.istioinaction.svc.cluster.local
  http: 
  - route:
    - destination:
        host: purchase-history.istioinaction.svc.cluster.local
        subset: v1
        port:
          number: 8080
      weight: 100
```

Also review the destination rule resource for the `purchase-history` service that defines the `v1` and `v2` subsets. Since `v2` is dark launched and no traffic will go to `v2`, it is not required to have `v2` subsets now but you will need it soon.

```bash
cat labs/05/purchase-history-dr.yaml
```

```text
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: purchase-history-dr
spec:
  host: purchase-history.istioinaction.svc.cluster.local
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

Apply the `purchase-history-vs` and `purchase-history-dr` resources in the `istioinaction` namespace:

```bash
kubectl apply -f labs/05/purchase-history-vs-all-v1.yaml -n istioinaction
kubectl apply -f labs/05/purchase-history-dr.yaml -n istioinaction
```

After you have configured Istio to send 100% of traffic to `purchase-history` to `v1` of the service, you can now deploy the `v2 of the `purchase-history` service`:

```bash
kubectl apply -f labs/05/purchase-history-v2.yaml -n istioinaction
```

Confirm the new `v2` `purchase-history` pod has reached running: 

<!--bash
kubectl wait --for=condition=Ready pod -l app=purchase-history -n istioinaction
-->
```bash
kubectl get pods -n istioinaction -l app=purchase-history
```

You should see both `v1` and `v2` of the `purchase-history` pods are running, each with its own sidecar proxy.

```text
NAME                                   READY   STATUS    RESTARTS   AGE
purchase-history-v1-55989d4c56-vv5d4   2/2     Running   0          2d4h
purchase-history-v2-74886f799f-lgzfn   2/2     Running   0          4m
```

Check the `purchase-history-v2` pod logs to see if there is any errors:

```bash
kubectl logs deploy/purchase-history-v2 -n istioinaction
```

Note the `connection refused` error at the beginning of the log during the service initialization:

```text
2021-06-11T17:47:59.776Z [INFO]  Starting service: name=purchase-history-v2 upstreamURIs= upstreamWorkers=1 listenAddress=0.0.0.0:8080 service type=http
Unable to connect to the external service:  Get "https://jsonplaceholder.typicode.com/posts": dial tcp 104.21.41.57:443: connect: connection refused
2021-06-11T17:47:59.804Z [INFO]  Adding handler for UI static files
2021-06-11T17:47:59.804Z [INFO]  Settings CORS options: allow_creds=false allow_headers=Accept,Accept-Language,Content-Language,Origin,Content-Type allow_origins=*
2021-06-11T17:48:32.473Z [INFO]  Handle inbound request: request="GET / HTTP/1.1
```

This is not good, the `purchase-history-v2` pod *cannot* reach the JasonPlaceHolder external service at startup time. Generate some load on the `web-api` service to ensure your users are not impacted by the newly added v2 of the `purchase-history` service:

<!--bash
GATEWAY_IP=$(kubectl get svc -n istio-system istio-ingressgateway -o jsonpath="{.status.loadBalancer.ingress[0].ip}")
SECURE_INGRESS_PORT=443
-->
```bash
for i in {1..10}; do curl -s --cacert ./labs/02/certs/ca/root-ca.crt -H "Host: istioinaction.io" https://istioinaction.io:$SECURE_INGRESS_PORT --resolve istioinaction.io:$SECURE_INGRESS_PORT:$GATEWAY_IP|grep "Hello From Purchase History"; done
```

You will see all of the 10 responses from `purchase-history` are from v1 of the service. This is great! We introduced the problematic v2 of the service but thankfully it didn't impact any of the behavior of the existing requests.

Recall the `v2` of the `purchase-history` service has some code to call the external service and requires the ability for the pod to connect to the external service during initialization. By default in Istio, the `istio-proxy` starts in parallel with the application container \(`purchase-history-v2` here in our example\) so it is possible that the application container reaches running before `istio-proxy` fully starts thus unable to connect to any external services outside of the cluster.

How can we solve this problem and ensure the application container can connect to services outside of the cluster during the container start time? The `holdApplicationUntilProxyStarts` configuration is introduced in Istio to solve this problem. Let us add this configuration to the pod annotation of the `purchase-history-v2` to use it:

```bash
cat labs/05/purchase-history-v2-updated.yaml
```

From the `holdApplicationUntilProxyStarts` annotation below, you have configured the `purchase-history-v2` pod to delay starting until the `istio-proxy` container reaches the `Running` status:

```text
  template:
    metadata:
      labels:
        app: purchase-history
        version: v2
      annotations:
        proxy.istio.io/config: '{ "holdApplicationUntilProxyStarts": true }'
    spec:
```

Deploy the updated `v2` of the `purchase-history` service:

```bash
kubectl apply -f labs/05/purchase-history-v2-updated.yaml -n istioinaction
```

Check the `purchase-history-v2` pod logs to see to ensure there is no error this time:

```bash
kubectl logs deploy/purchase-history-v2 -n istioinaction
```

You will see we are able to connect to the external service in the log:

```text
2021-06-11T18:13:03.573Z [INFO]  Able to connect to : https://jsonplaceholder.typicode.com/posts=<unknown>
```

Test the v2 of the `purchase-history` service from its own sidecar proxy: 

<!--bash
sleep 2
-->
```bash
kubectl exec deploy/purchase-history-v2 -n istioinaction -c istio-proxy -- curl -s localhost:8080
```

Awesome! You are getting a valid response this time! If you rerun the above command, you will notice a slightly different body from `purchase-history-v2` each time.

```text
{
  "name": "purchase-history-v2",
  "uri": "/",
  "type": "HTTP",
  "ip_addresses": [
    "10.42.0.23"
  ],
  "start_time": "2021-06-10T18:47:15.624118",
  "end_time": "2021-06-10T18:47:15.624438",
  "duration": "320.459µs",
  "body": "Hello From Purchase History (v2)! + History: 24 Title: autem hic labore sunt dolores incidunt Body: autem hic labore sunt dolores incidunt",
  "code": 200
}
```

### Selectively Route Requests

You want to test the `v2` of the `purchase-history` service only from a specific test user while all other requests continue to route to the `v1` of the `purchase-history` service. With Istio's virtual service resource, you can specify HTTP routing rules based on HTTP requests such as header information. In the `purchase-history-vs-all-v1-header-v2.yaml` file shown in the following example, you will see that an HTTP route rule has been defined to route requests from clients using `user: Jason` header to the `v2` of the `purchase-history` service. All other client requests will continue to use the `v1` subset of the `purchase-history` service:

```bash
cat labs/05/purchase-history-vs-all-v1-header-v2.yaml
```

Review the changes of the `purchase-history-vs` virtual service resource:

```text
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: purchase-history-vs
spec:
  hosts:
  - purchase-history.istioinaction.svc.cluster.local
  http:
  - match:
    - headers:
        user:
          exact: Jason
    route:
    - destination:
        host: purchase-history.istioinaction.svc.cluster.local
        subset: v2
        port:
          number: 8080  
  - route:
    - destination:
        host: purchase-history.istioinaction.svc.cluster.local
        subset: v1
        port:
          number: 8080
      weight: 100
```

Apply the changes to your mesh using this command:

```bash
kubectl apply -f labs/05/purchase-history-vs-all-v1-header-v2.yaml -n istioinaction
```

Send some traffic to the `web-api` service through the `istio-ingressgateway`:

```bash
curl --cacert ./labs/02/certs/ca/root-ca.crt -H "Host: istioinaction.io" -H "user: jason" https://istioinaction.io:$SECURE_INGRESS_PORT --resolve istioinaction.io:$SECURE_INGRESS_PORT:$GATEWAY_IP
```

You will get `Hello From Purchase History (v1)!` in the response. Why is that? Recall we configured _exact_ in `exact: Jason` earlier. Change the command using `user: Jason` instead:

```bash
curl --cacert ./labs/02/certs/ca/root-ca.crt -H "Host: istioinaction.io" -H "user: Jason" https://istioinaction.io:$SECURE_INGRESS_PORT --resolve istioinaction.io:$SECURE_INGRESS_PORT:$GATEWAY_IP
```

You should see `Hello From Purchase History (v2)!` in the response! Feel free to send a few more requests. Based on your routing rule configuration, Istio made sure that requests with header `user: Jason` always route to the v2 of the `purchase-history` service while all other requests continue to route to v1 of the `purchase-history` service.

## Canary Testing

You have dark launched and did some basic testing of the `v2` of the `purchase-history` service. You want to canary test a small percentage of requests to the new version to determine whether ther are problems before routing all traffic to the new version. Canary tests are often performed to ensure the new version of the service not only functions properly but also doesn't cause any degradation in performance or reliability.

### Shift 20% Traffic to `v2`

Review the updated `purchase-history` virtual service resource that shifts 20% of the traffic to `v2` of the `purchase-history` service:

```bash
cat labs/05/purchase-history-vs-20-v2.yaml
```

You will notice `subset: v2` is added which will get 20% of the traffic while `subset: v1` will get 80% of the traffic:

```text
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: purchase-history-vs
spec:
  hosts:
  - purchase-history.istioinaction.svc.cluster.local
  http: 
  - route:
    - destination:
        host: purchase-history.istioinaction.svc.cluster.local
        subset: v1
        port:
          number: 8080
      weight: 80
    - destination:
        host: purchase-history.istioinaction.svc.cluster.local
        subset: v2
        port:
          number: 8080
      weight: 20
```

Deploy the updated `purchase-history` virtual service resource:

```bash
kubectl apply -f labs/05/purchase-history-vs-20-v2.yaml -n istioinaction
```

Generate some load on the `web-api` service to check how many requests are served by `v1` and `v2` of the `purchase-history` service. You should see only a few from `v2` while the rest from `v1`. You may be curious why you are not observe the exactly 80%/20% distribution among `v1` and `v2`. You likely need to have over 100 requests to get the desired 80%/20% weighted version distribution.

<!--bash
sleep 2
-->
```bash
for i in {1..20}; do curl -s --cacert ./labs/02/certs/ca/root-ca.crt -H "Host: istioinaction.io" https://istioinaction.io:$SECURE_INGRESS_PORT --resolve istioinaction.io:$SECURE_INGRESS_PORT:$GATEWAY_IP|grep "Hello From Purchase History"; done
```

### Shift 50% Traffic to `v2`

Review the updated `purchase-history` virtual service resource:

```bash
cat labs/05/purchase-history-vs-50-v2.yaml
```

You will notice `subset: v2` is updated to get 50% of the traffic while `subset: v1` will get 50% of the traffic:

```text
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: purchase-history-vs
spec:
  hosts:
  - purchase-history.istioinaction.svc.cluster.local
  http: 
  - route:
    - destination:
        host: purchase-history.istioinaction.svc.cluster.local
        subset: v1
        port:
          number: 8080
      weight: 50
    - destination:
        host: purchase-history.istioinaction.svc.cluster.local
        subset: v2
        port:
          number: 8080
      weight: 50
```

Deploy the updated `purchase-history` virtual service resource:

```bash
kubectl apply -f labs/05/purchase-history-vs-50-v2.yaml -n istioinaction
```

Generate some load on the `web-api` service to check how many requests are served by `v1` and `v2` of the `purchase-history` service. You should observe _roughly_ 50%/50% distribution among the `v1` and `v2` of the service.

<!--bash
sleep 2
-->
```bash
for i in {1..20}; do curl -s --cacert ./labs/02/certs/ca/root-ca.crt -H "Host: istioinaction.io" https://istioinaction.io:$SECURE_INGRESS_PORT --resolve istioinaction.io:$SECURE_INGRESS_PORT:$GATEWAY_IP|grep "Hello From Purchase History"; done
```

### Shift All Traffic to `v2`

Now you haven't observed any ill effect during your test, you can adjust the routing rules to direct all of the traffic to the canary deployment:

Deploy the updated `purchase-history` virtual service resource:

```bash
kubectl apply -f labs/05/purchase-history-vs-all-v2.yaml -n istioinaction
```

Generate some load on the `web-api` service, you should only see traffic to the `v2` of the `purchase-history` service.

<!--bash
sleep 2
-->
```bash
for i in {1..20}; do curl -s --cacert ./labs/02/certs/ca/root-ca.crt -H "Host: istioinaction.io" https://istioinaction.io:$SECURE_INGRESS_PORT --resolve istioinaction.io:$SECURE_INGRESS_PORT:$GATEWAY_IP|grep "Hello From Purchase History"; done
```

## Controlling Outbound Traffic

When you use Kubernetes, any application pod can make calls to services that are outside the Kubernetes cluster unless there is a Kubernetes network policy that prevents calling the target service. However, network policies are restricted to layer 4 rules which means that they can only allow or restrict access to specific IP addresses. What if you want more control over how applications within the mesh can reach external services using layer 7 policies and more fine-grained attribute policy evaluation?

By default, Istio allows all outbound traffic to ensure users have a smooth starting experience. If you choose to restrict all outbound traffic across the mesh, you can update your Istio install to enable restricted outbound traffic access so that only registered external services are allowed. This is highly recommended.

* Check the default Istio installation configuration for `outboundTrafficPolicy`:

```bash
kubectl get istiooperator installed-state -n istio-system -o jsonpath='{.spec.meshConfig.outboundTrafficPolicy.mode}'
```

The output should be empty. This means the default mode `ALLOW_ANY` is used, which allows services in the mesh to access any external service.

* Update your Istio installation so that only registered external services are allowed, using the `meshConfig.outboundTrafficPolicy.mode` configuration:

```bash
istioctl install --set profile=demo --set meshConfig.outboundTrafficPolicy.mode=REGISTRY_ONLY -y
```

* Confirm the new configuration, you should see `REGISTRY_ONLY` from the output:

```bash
kubectl get istiooperator installed-state -n istio-system -o jsonpath='{.spec.meshConfig.outboundTrafficPolicy.mode}'
```

* Send some traffic to the `web-api` service:

```bash
curl --cacert ./labs/02/certs/ca/root-ca.crt -H "Host: istioinaction.io" https://istioinaction.io:$SECURE_INGRESS_PORT --resolve istioinaction.io:$SECURE_INGRESS_PORT:$GATEWAY_IP
```

You should see the request to `purchase-history` to fail because all outbound traffics are blocked by default but the v2 of `purchase-history` service needs to connect to the `jsonplaceholder.typicode.com` service.

```text
{
  "name": "web-api",
  "uri": "/",
  "type": "HTTP",
  "ip_addresses": [
    "10.42.0.50"
  ],
  "start_time": "2021-06-15T02:23:09.222368",
  "end_time": "2021-06-15T02:23:09.361310",
  "duration": "138.942038ms",
  "upstream_calls": [
    {
      "name": "recommendation",
      "uri": "http://recommendation:8080",
      "type": "HTTP",
      "ip_addresses": [
        "10.42.0.43"
      ],
      "start_time": "2021-06-15T02:23:09.225295",
      "end_time": "2021-06-15T02:23:09.326935",
      "duration": "101.640115ms",
      "upstream_calls": [
        {
          "uri": "http://purchase-history:8080",
          "code": 503,
          "error": "Error processing upstream request: http://purchase-history:8080/"
        }
      ],
      "code": 500,
      "error": "Error processing upstream request: http://recommendation:8080/"
    }
  ],
  "code": 500
}
```

Check the pod logs of the `purchase-history-v2` pod:

```bash
kubectl logs deploy/purchase-history-v2 -n istioinaction | grep "x-envoy-attempt-count: 3" -A 10
```

You can see envoy attempted 3 times including the 2 retries by default and the service can't connect to the external service \(`jsonplaceholder.typicode.com` here\) successfully.

```text
x-envoy-attempt-count: 3
x-envoy-internal: true
x-forwarded-proto: https
x-b3-sampled: 1
accept: */*
x-forwarded-for: 10.42.0.1"
2021-06-15T02:23:09.301Z [INFO]  Sleeping for: duration=-265.972µs
Get "https://jsonplaceholder.typicode.com/posts?id=65": EOF
2021-06-15T02:23:09.304Z [INFO]  Finished handling request: duration=3.393441ms
2021/06/15 02:23:09 http: panic serving 127.0.0.6:41109: json: error calling MarshalJSON for type json.RawMessage: invalid character 'h' after top-level value
goroutine 961 [running]:
net/http.(*conn).serve.func1(0xc000192320)
    /usr/local/Cellar/go/1.16/libexec/src/net/http/server.go:1824 +0x153
panic(0x9a5de0, 0xc000404480)
```

Above is the expected behavior of the `REGISTRY_ONLY` outboundTrafficPolicy mode in Istio. When services in the mesh attempts to access external services, only registered external services are allowed and you haven't registered any external services yet.

* Istio has the ability to selectively access external services using a Service Entry resource. A Service Entry allows you to bring a service that is external to the mesh and make it accessible by services within the mesh. In other words, through service entries, you can bring external services as participants in the mesh. You can create the following service entry resource for the `jsonplaceholder.typicode.com` service:

```bash
cat labs/05/typicode-se.yaml
```

```text
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: typicode-svc-https
spec:
  hosts:
  - jsonplaceholder.typicode.com
  location: MESH_EXTERNAL
  ports:
  - number: 443
    name: https
    protocol: TLS
  resolution: DNS
```

Apply the service entry resource into the `istioinaction` namespace:

```bash
kubectl apply -f labs/05/typicode-se.yaml -n istioinaction
```

* Send some traffic to the `web-api` service. You should get the `200` response now.

<!--bash
sleep 2
-->
```bash
curl --cacert ./labs/02/certs/ca/root-ca.crt -H "Host: istioinaction.io" https://istioinaction.io:$SECURE_INGRESS_PORT --resolve istioinaction.io:$SECURE_INGRESS_PORT:$GATEWAY_IP
```

* Another important benefit of importing external services through service entries in Istio is that you can use Istio routing rules with external services to define retries, timeouts, and fault injection policies. For example, you can set a timeout rule on calls to the `jsonplaceholder.typicode.com` service as shown below:

```bash
cat labs/05/typicode-vs.yaml
```

```text
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: typicode-vs
spec:
  hosts:
    - "jsonplaceholder.typicode.com"
  https:
  - timeout: 3s
    route:
      - destination:
          host: "jsonplaceholder.typicode.com"
        weight: 100
```

Run the following command to apply the virtual service resource:

```bash
kubectl apply -f labs/05/typicode-vs.yaml -n istioinaction
```

### Questions

Do you want to securely restrict which pods can access a given external service? Should you send traffic to your external service through the istio-egressgateway? We will cover this in the Istio Expert workshop.

Interested in increasing the resiliency between the services, check out the traffic control [bonus lab](05-control-traffic-bonus.md).

## Conclusion

A service mesh like Istio has the capabilities that enable you to manage traffic flows within the mesh as well as entering and leaving the mesh. These capabilities allow you to efficiently control rollout and access to new features, and make it possible to build resilient services without having to make complicated changes to your application code.

If you like this workshop, visit [Solo.io](https://solo.io) to check out our future Istio workshops. Reach out to us on [slack](https://slack.solo.io) and follow us on [Twitter](https://twitter.com/soloio_inc) to get notified about our future workshops, webinars and more.
