# Lab 5 :: Control Traffic Bonus

You will explore some features provided by Istio to increase the resiliency between the services.

## Prerequisites

Verify you're in the correct folder for this lab: `~/istio-workshops/istio-basics`. This is a bonus lab that builds on the [lab 05](05-control-traffic.md) where you shift all traffic to `v2` of the `purchase-history` service after dark launch and canary test, and enabled global outbound policy with `REGISTRY_ONLY`.

```bash
cd ~/istio-workshops/istio-basics
```

## Resiliency and Chaos Testing

When you build a distributed application, it is critical to ensure the services in your application are resilient to failures in the underlying platforms or the dependent services. Istio has support for retries, timeouts, circuit breakers and even injecting faults into your service calls to help you test and tune your timeouts. Similar as the dark launch and canary testing you explored earlier, you don't need to add these logic into your application code or redeploy your application when configuring these Istio features to increase the resiliency of your services.

### Retries

Istio has support to program retries for your services in the mesh without you specifying any changes to your code. By default, client requests to each of your services in the mesh will be retried twice. What if you want a different retries per route for some of your virtual services? You can adjust the number of retries or disable them altogether when automatic retries don't make sense for your services. Display the content of the `purchase-history-vs-all-v2-v3-retries.yaml`:

```bash
cat labs/05/purchase-history-vs-all-v2-v3-retries.yaml
```

Note the number of retries configuration is for the `purchase-history` service when the http header `user` matches exactly to `Jason` then routes to v3 of the `purchase-history` service. All other cases, continue to route to `v2` of the `purchase-history` service:

```
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
        subset: v3
        port:
          number: 8080
    retries:
      attempts: 0
  - route:
    - destination:
        host: purchase-history.istioinaction.svc.cluster.local
        subset: v2
        port:
          number: 8080
      weight: 100
```

Apply the virtual service resource to the `istioinaction` namespace, along with the updated `purchase-history` destination rule that contains the `v3` subset:

```bash
kubectl apply -f labs/05/purchase-history-vs-all-v2-v3-retries.yaml -n istioinaction
kubectl apply -f labs/05/purchase-history-dr-v3.yaml -n istioinaction
```

To see the new retry configuration in action, you can create a new version of the purchase history (`v3`) which errors 50% of the time with 503 error code and 4 seconds of error delay. You will use it to simulate a bad application deployment and show how retries would help prevent end users from experiencing them.

```bash
cat labs/05/purchase-history-v3.yaml
```

```
        - name: "ERROR_CODE"
          value: "503"
        - name: "ERROR_RATE"
          value: "0.5"
        - name: "ERROR_TYPE"
          value: "delay"
        - name: "ERROR_DELAY"
          value: "4s"
```

Deploy the new version of the purchase history (`v3`) to the `istioinaction` namespace:

```bash
kubectl apply -f labs/05/purchase-history-v3.yaml -n istioinaction
```

Generate some load with the `user: Jason` header to ensure traffic goes 100% to `v3` of the `purchase-history` service. You will quickly see errors from the v3 of the `purchase-history` service:

<!--bash
kubectl wait --for=condition=Ready pod -l app=purchase-history -n istioinaction
-->
```bash
for i in {1..6}; do kubectl exec deploy/sleep -n istioinaction -- curl -s -H "user: Jason" http://purchase-history:8080/; done
```

You will see 50% successful rate and 50% failure with 503 error code.  If you remove the retries configuration (see `labs/05/purchase-history-vs-all-v2-header-v3.yaml`) and use Istio's default retry, you should not see any errors from `v3` of the `purchase-history` service because the service errors 50% of the time and Istio will retry any failed request to the service with 503 error code automatically up to 2 times:

```bash
cat labs/05/purchase-history-vs-all-v2-header-v3.yaml
kubectl apply -f labs/05/purchase-history-vs-all-v2-header-v3.yaml -n istioinaction
```

Generate some load you should *NOT* see any errors from the `v3` of the `purchase-history` service:

<!--bash
sleep 2
-->
```bash
for i in {1..6}; do kubectl exec deploy/sleep -n istioinaction -- curl -s -H "user: Jason" http://purchase-history:8080/; done
```

If you check the logs of the `purchase-history` service, you will see the retries:

```bash
kubectl logs deploy/purchase-history-v3 -n istioinaction | grep x-envoy-attempt-count
```

In the log shown below, the first request was not successful and the automatic retry was successful:

```
x-envoy-attempt-count: 1
x-envoy-attempt-count: 2
x-envoy-attempt-count: 1
x-envoy-attempt-count: 2
```

Note: The error code has to be `503` for Istio to retry the requests. If you change the `ERROR_CODE` to `500` in the `purchase-history-v3.yaml`, redeploy the updated `purchase-history-v3.yaml` file and send some request to the `purchase-history` service from the `sleep` pod, you will get error 50% of the times.

### Timeouts

Istio has built-in support for timeouts with client requests to services within the mesh. The default timeout for HTTP request in Istio is disabled, which means no timeout. You can overwrite the default timeout setting of a service route within the route rule for a virtual service resource. For example, in the route rule within the `purchase-history-vs` resource below, you can add the following `timeout` configuration to set the timeout of the route to the `purchase-history` service on port `8080`, along with 3 retry attempts with each retry timeout after 3 seconds.

```bash
cat labs/05/purchase-history-vs-all-v2-v3-retries-timeout.yaml
```

```
  http:
  - route:
    - destination:
        host: purchase-history.istioinaction.svc.cluster.local
        port:
          number: 8080
    retries:
      attempts: 3
      perTryTimeout: 3s
    timeout: 6s
```

Apply the resource in the `istioinaction` namespace:

```bash
kubectl apply -f labs/05/purchase-history-vs-all-v2-v3-retries-timeout.yaml -n istioinaction
```

Send some traffic to the `purchase-history` service from the `sleep` pod:

<!--bash
sleep 2
-->
```bash
for i in {1..6}; do kubectl exec deploy/sleep -n istioinaction -- curl -s -H "user: Jason" http://purchase-history:8080/|grep timeout; done
```

You will see 3 requests time out:

```
upstream request timeout
upstream request timeout
upstream request timeout
```

Recall that you added the 50% error rate and 4 second delays added to the v3 of the `purchase-history` service, along with the 3 seconds `perTryTimeout` and 6 seconds `timeout`, the request would have timed out (e.g. 6s < 4s + 3s) before the 2nd retry could be attempted when the first request failed.

### Circuit Breakers

Circuit breaking is an important pattern for creating resilient microservice applications. Circuit breaking allows you to limit the impact of failures and network delays, which are often outside of your control when making requests to dependent services. Prior to service mesh, you could add logic directly within your code \(or your language specific library\) to handle situations when the calling service fails to provide the desirable result. Istio allows you to apply circuit breaking configurations within a destination rule resource, without any need to modify your service code.

Take a look at the `web-api-dr` destination rule as shown in the example that follows. It defines the destination rule for the `web-api` service. Within the traffic policy of the `web-api-dr`, you can specify the connection pool configuration to indicate the maximum number of TCP connections, the maximum number of HTTP requests per connection and set the outlier detection to be three minutes after a single error. When any clients access the `web-api` service, these circuit-breaker behavior will be followed.

```bash
cat labs/05/web-api-dr-with-cb.yaml
```

```text
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: web-api-dr
spec:
  host: web-api.istioinaction.svc.cluster.local
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
    outlierDetection:
      consecutive5xxErrors: 1
      interval: 1s
      baseEjectionTime: 3m
      maxEjectionPercent: 100
```

Apply the resource in the `istioinaction` namespace:

```bash
kubectl apply -f labs/05/web-api-dr-with-cb.yaml -n istioinaction
```

#### Questions

* Can you apply the `web-api-dr-with-cb.yaml` in the `istio-system` namespace?  We will answer this in the workshop.
* Note the `web-api-dr` destination rule applies to requests from any client to the `web-api` service in the istioinaction namespace. Do you want to configure the client scope of the destination rule? We will teach how to do that in the Essential badge.

### Fault Injection

It can be difficult to configure service timeouts and circuit-breaker configurations properly in a distributed microservice application. Istio makes it easier to get these settings correct by enabling you to inject faults into your application without the need to modify your code. With Istio, you can perform chaos testing of your application easily by adding an HTTP delay fault into the `web-api` service only for user `Amy` so that the injected fault doesn't affect any other users.

You can inject a 30-second fault delay for 100% of the client requests when the `user` HTTP header value exactly matches the value `Amy`. 

```bash
cat labs/05/web-api-gw-vs-fault-injection.yaml
```

```text
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
  - fault:
      delay:
        fixedDelay: 15s
        percentage:
          value: 100
    match:
    - headers:
        user:
          exact: Amy
    route:
    - destination:
        host: web-api.istioinaction.svc.cluster.local
        port:
          number: 8080
  - route:
    - destination:
        host: web-api.istioinaction.svc.cluster.local
        port:
          number: 8080
```

Apply the virtual service resource using the following command:

```bash
kubectl apply -f labs/05/web-api-gw-vs-fault-injection.yaml -n istioinaction
```

Send some traffic to the `web-api` service, you should see `200` response code right away:

<!--bash
sleep 2
-->
```bash
curl --cacert ./labs/02/certs/ca/root-ca.crt -H "Host: istioinaction.io" https://istioinaction.io:$SECURE_INGRESS_PORT --resolve istioinaction.io:$SECURE_INGRESS_PORT:$GATEWAY_IP
```

Send some traffic to the `web-api` service with the `user: Amy` header, you should see `200` response code after the 15 seconds delay:

```bash
curl --cacert ./labs/02/certs/ca/root-ca.crt -H "Host: istioinaction.io" -H "user: Amy" https://istioinaction.io:$SECURE_INGRESS_PORT --resolve istioinaction.io:$SECURE_INGRESS_PORT:$GATEWAY_IP
```

Note that there is no change to the `web-api` service to inject the delay. Istio injects the delay automatically via programmatically configuring the `istio-proxy` container.

## Conclusion

Congratulations! You have completed all of the challenges of the workshop.

If you like this workshop, visit [Solo.io](https://solo.io) to check out our future Istio workshops. Reach out to us on [slack](https://slack.solo.io) and follow us on [Twitter](https://twitter.com/soloio_inc) to get notified about our future workshops, webinars and more.
