# Lab 1 :: Zero downtime upgrades

In this lab, we will learn the proper method of upgrading Istio without your applications experiencing any downtime. This involves deploying a canary version, testing with a small workload first and then gradually moving over all workloads while monitoring. This approach is safe and effective when done correctly.

In this portion of the lab, we will upgrade from Istio 1.11 to 1.12.

## Prerequisites

You will need access to a Kubernetes cluster. If you're doing this via the Solo.io Workshop format, you should have everything ready to go.

Verify you're in the correct folder for this lab: `/home/solo/workshops/istio-day2/2-operate-istio/`.

## Install Istio 1.11
In the workshop material, you should already have Istio `1.11.5` cli installed and ready to go.

To verify, run

```bash
istioctl version
```

```
# OUTPUT:
no running Istio pods in "istio-system"
1.11.5
```

We don't have the Istio control plane installed and running yet. Let's go ahead and do that.


## Installing Istio

Install Istio as described in first part of these series.

Now let's install the control plane. This installation uses the `IstioOperator` CR along with `istioctl`. The IstioOperator's profile is set to minimal, which only installs the control plane (no gateways):

```
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: control-plane
spec:
  profile: minimal
```

```bash
istioctl install -y -f labs/01/control-plane.yaml --revision 1-11-5
```

```
# OUTPUT:
✔ Istio core installed
✔ Istiod installed
✔ Installation complete
```

If we check the `istio-system` workspace, we should see the control plane running:

```bash
kubectl get pod -n istio-system
```

```
# OUTPUT:
NAME                            READY   STATUS    RESTARTS   AGE
istiod-1-11-5-c6d55ff4c-wkr2q   1/1     Running   0          2m1s
```

## Install Istio Gateway

We recommend installing your Istio Ingress Gateways in a different namespace than istio-system for additional security. Let's install it in the a namespace called `istio-ingress`

We will also leverage the new injected gateways feature of Istio which injects the configuration for the gateways similar to the sidecar injection. One of the main advantages of this approach is the fact that we can now point the same gateways at different revisions of the injector and the istiod control plane.

Install the gateway and set it talk to our revisioned control plane:

```bash
kubectl create namespace istio-ingress
istioctl install -y -f labs/01/ingress-gateway.yaml --revision=1-11-5
```

## Install demo app

In this section, we'll enable automatic sidecar injection on the `default` namespace and deploy the httpbin application. Remove any previous injection labels and set the the label to point to our revisioned injector

```bash
kubectl label namespace default istio-injection-
kubectl label namespace default istio.io/rev=1-11-5 --overwrite
```

```bash
kubectl apply -f labs/01/httpbin.yaml
```

Check the status of the proxy to see the version and istiod the proxy is pointed to:

```bash
istioctl ps
```

```
# OUTPUT:
NAME                                                    CDS        LDS        EDS        RDS          ISTIOD                            VERSION
httpbin-74fb669cc6-x5kmb.default                        SYNCED     SYNCED     SYNCED     SYNCED     istiod-1-11-5-c6d55ff4c-wkr2q     1.11.5
istio-ingressgateway-77cdfc5b86-7fdvm.istio-ingress     SYNCED     SYNCED     SYNCED     SYNCED     istiod-1-11-5-c6d55ff4c-wkr2q     1.11.5
```

## Visit the httpbin application

Get the IP address of your Istio Ingress Gateway:
```
export INGRESS_HOST=$(kubectl -n istio-ingress get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo $INGRESS_HOST
```

Curl your httpbin application:
```
curl -I $INGRESS_HOST
```
You should see `HTTP/1.1 200 OK`

In ANOTHER WINDOW, run this loop to constantly curl the httpbin. We will monitor this output during our upgrade of Istio:
```
for i in {1..1000}; do curl -s -o /dev/null -I -w "%{http_code}\n"  $INGRESS_HOST; sleep 2; done
```

# Download Istio 1.12

Lets start our upgrade to 1.12 by downloading it.

```
TAG="1.12.1"
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=${TAG}  sh -
export PATH=$PWD/istio-${TAG}/bin:$PATH
```

Check your istioctl version:
```bash
istioctl version
```

```bash
# OUTPUT:
client version: 1.12.1
control plane version: 1.11.5
data plane version: 1.11.5 (2 proxies)
```

## Deploy Istio 1.12 control plane

```bash
istioctl x precheck
istioctl install -y -n istio-system -f labs/01/control-plane.yaml --revision=1-12-1
```

You should now have both istiod versions running:
```
kubectl get pods -n istio-system
```

```bash
# OUTPUT:
NAME                             READY   STATUS    RESTARTS   AGE
istiod-1-11-5-c6d55ff4c-wb44r   1/1     Running   0          3h22m
istiod-1-12-1-8b859874f-psrv8   1/1     Running   0          15s
```

## Switch workloads to the new control plane

Point the default namespace injection label to the 1-21-1 revision injector:

```bash
kubectl label namespace default istio.io/rev=1-12-1 --overwrite
``

Then, recreate the httpbin pods

```
kubectl rollout restart deployment httpbin
```

Check that the pod now points to the new Istio version:

```bash
istioctl ps
```

```bash
# OUTPUT
NAME                                                    CDS        LDS        EDS        RDS          ISTIOD                             VERSION
httpbin-56594c4bc5-vdbg4.default                        SYNCED     SYNCED     SYNCED     SYNCED     istiod-1-12-1-8b859874f-psrv8     1.12.1
istio-ingressgateway-54c4f455b8-mtvhr.istio-ingress     SYNCED     SYNCED     SYNCED     SYNCED     istiod-1-11-5-c6d55ff4c-wb44r     1.11.5
```

## Upgrade Istio Gateway to 1.12

Peform an in-place upgrade of the gateway to 1.12

Similar to your application pods, the gateways are in place by reinstalling them and pointing them to the new revision of istiod and injector.

Before upgrading our current gateways, we can deploy ANOTHER gateway as a canary that points to the new revision. If we're satisfied by the canary, we can upgrade our main one and delete the canary.

Deploy a canary ingress gateway:

```bash
istioctl install -y -f labs/01/ingress-gateway-canary.yaml --revision 1-12-1
```

You should now have two Istio gateway deployment and service.
```
kubectl get deployment,service -n istio-ingress
NAME                                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/istio-ingressgateway        1/1     1            1           22m
deployment.apps/istio-ingressgateway-test   1/1     1            1           2m7s

NAME                                TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                                      AGE
service/istio-ingressgateway        LoadBalancer   10.56.52.97    34.70.229.216   15021:32049/TCP,80:32018/TCP,443:31392/TCP   22m
service/istio-ingressgateway-test   LoadBalancer   10.56.59.225   34.66.95.35     15021:31204/TCP,80:30699/TCP,443:32080/TCP   2m6s
```


Visit the External-IP of the Istio 1.12 ingress gateway service. You should be able to see your httpbin application.

```
export CANARY_INGRESS_HOST=$(kubectl -n istio-ingress get service istio-ingressgateway-test -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo $CANARY_INGRESS_HOST
curl -I $CANARY_INGRESS_HOST
```

If you visit your other tab, you should see that the 1-11-5 gateway is still functional.

Because the new gateway is working well, you can now do an inplace rolling update of your original ingress gateway:

```
istioctl install -y -f labs/01/ingress-gateway.yaml --revision 1-12-1
```

Finish by deleting the canary gateway.

```
istioctl manifest generate -f labs/01/ingress-gateway-canary.yaml | kubectl delete -f -
```

## Uninstall Istio 1.11.5 control plane

Confirm that all your workloads and gateways are now talking to the 1-12-1 control plane.
```
istioctl ps
```

```
httpbin-56594c4bc5-vdbg4.default                        SYNCED     SYNCED     SYNCED     SYNCED     istiod-1-12-1-8b859874f-psrv8     1.12.1
istio-ingressgateway-5b9f9cc5bb-nkdlm.istio-ingress     SYNCED     SYNCED     SYNCED     SYNCED     istiod-1-12-1-8b859874f-psrv8     1.12.1
```

Once you're ready, uninstall:
```
istioctl x uninstall --revision 1-11-5
```

That should remove the 1.11.5 istiod, leaving you with just Istio 1.12.1!

## Next Lab


