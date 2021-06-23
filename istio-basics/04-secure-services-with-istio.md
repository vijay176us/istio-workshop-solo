# Lab 4 :: Securing Communication Within Istio

In the previous lab, you explored adding services into a mesh. When you installed Istio using the demo profile, it has permissive security mode. Istio permissive security setting is useful when you have services that are being moved into the service mesh incrementally by allowing both plain text and mTLS traffic. In this lab, you will explore how Istio manages secure communication between services and how to enable strict security communication between services in the sample application.

## Prerequisites

Verify you're in the correct folder for this lab: `~/istio-workshops/istio-basics`. This lab builds on the [lab 03](03-add-services-to-mesh.md) where you added your services to the mesh.

```bash
cd ~/istio-workshops/istio-basics
```

## Permissive mode

By default, Istio automatically upgrades the connection securely from the source service's sidecar proxy to the target service's sidecar proxy. This is why you saw the paddlelock icon in the Kiali graph earlier from Istio ingress gateway to the `web-api` service to the `history` service then to the `recommendation` service. While this is good when onboarding your services to Istio service mesh as the communication between source and target services continues to be allowed via plain text if mutual TLS communication fails, you don't want this in production environment without proper security policy in place.

Check if you have any `peerauthentication` policy in all of your namespaces:

```bash
kubectl get peerauthentication --all-namespaces
```

You should see `No resources found` in the output, which means no peer authentication has been specified and the default `PERMISSIVE` mTLS mode is being used.

## Enable strict mTLS

You can lock down the secure access to all services in your Istio service mesh to require mTLS using a peer authentication policy. Execute this command to define a default policy for the `istio-system` namespace that updates all of the servers to accept only mTLS traffic:

```bash
kubectl apply -n istio-system -f - <<EOF
apiVersion: "security.istio.io/v1beta1"
kind: "PeerAuthentication"
metadata:
  name: "default"
spec:
  mtls:
    mode: STRICT
EOF
```

Verify your `peerauthentication` policy is installed:

```bash
kubectl get peerauthentication --all-namespaces
```

You should see the the default `peerauthentication` policy installed in the `istio-system` namespace with STRICT mTLS enabled:

```text
NAMESPACE      NAME      MODE     AGE
istio-system   default   STRICT   84s
```

Because the `istio-system` namespace is also the Istio mesh configuration root namespace in your environment, this `peerauthentication` policy is the default policy for all of your services in the mesh regardless of which namespaces your services run.

Let us see mTLS in action! 

* You can send to send some traffic to `web-api` from a pod that is not part of the Istio service mesh. Deploy the `sleep` service and pod in the `default` namespace:

```bash
kubectl apply -n default -f sample-apps/sleep.yaml
```

* Access the `web-api` service from the `sleep` pod in the `default` namespace: 

<!--bash
kubectl wait --for=condition=Ready pod -n default --all
-->
```bash
kubectl exec deploy/sleep -n default -- curl http://web-api.istioinaction:8080/
```

The request will fail because the `web-api` service can only be accessed with mutual TLS. The `sleep` pod in the `default` namespace doesn't have the sidecar proxy so it doesn't have the needed keys and certificates to communicate to the `web-api` service via mutual TLS. 

* Run the same command from the `sleep` pod in the `istioinaction` namespace:

```bash
kubectl exec deploy/sleep -n istioinaction -- curl http://web-api.istioinaction:8080/
```

You should see the request succeed.

### Questions

How can you check if a service or namespace is ready to enable the `STRICT` mtls mode? What is the best practice to enable mTLS for your services? We'll cover this topic in our Istio Essential workshop.

## Visualize mTLS enforcement in Kiali

You can visualize the services in the mesh in Kiali. 

* Enable access to Kiali using the command below:

```text
istioctl dashboard kiali
```

* Navigate to [http://localhost:20001](http://localhost:20001) and select the Graph tab.

On the "Namespace" dropdown, select "istioinaction". On the "Display" drop down, select "Traffic Animation" and "Security". 

* Generate some load to the data plane \(by calling our `web-api` service\) so that you can observe interactions among your services:

<!--bash
GATEWAY_IP=$(kubectl get svc -n istio-system istio-ingressgateway -o jsonpath="{.status.loadBalancer.ingress[0].ip}")
SECURE_INGRESS_PORT=443
for i in {1..10}; 
  do curl --cacert ./labs/02/certs/ca/root-ca.crt -H "Host: istioinaction.io" https://istioinaction.io:$SECURE_INGRESS_PORT  --resolve istioinaction.io:$SECURE_INGRESS_PORT:$GATEWAY_IP;
  echo "$i/10"
done
-->
```text
for i in {1..200}; 
  do curl --cacert ./labs/02/certs/ca/root-ca.crt -H "Host: istioinaction.io" https://istioinaction.io:$SECURE_INGRESS_PORT  --resolve istioinaction.io:$SECURE_INGRESS_PORT:$GATEWAY_IP;
  sleep 1;
done
```

* You should observe the service interaction graph with some traffic animation and security badges like below:

![](../.gitbook/assets/kiali-istioinaction-mtls-enforced.png)

## Understand how mTLS works in Istio service mesh

Inspect the key and/or certificates used by Istio for the `web-api` service in the `istioinaction` namespace:

```bash
istioctl proxy-config secret deploy/web-api -n istioinaction
```

From the output, you'll see there is the default secret and your Istio service mesh's root CA public certificate:

```text
RESOURCE NAME     TYPE           STATUS     VALID CERT     SERIAL NUMBER                               NOT AFTER                NOT BEFORE
default           Cert Chain     ACTIVE     true           289941409853020398869969650517379116839     2021-06-09T13:39:16Z     2021-06-08T13:39:16Z
ROOTCA            CA             ACTIVE     true           333717302760067951717891865272094769364     2031-06-06T13:36:14Z     2021-06-08T13:36:14Z
```

The `default` secret containers the public certificate information for the `web-api` service. You can analyze the contents of the default secret using openssl.

* Check the issuer of the public certificate:

```bash
istioctl proxy-config secret deploy/web-api -n istioinaction -o json | jq '[.dynamicActiveSecrets[] | select(.name == "default")][0].secret.tlsCertificate.certificateChain.inlineBytes' -r | base64 -d | openssl x509 -noout -text | grep 'Issuer'
```

```text
        Issuer: O = cluster.local
```

2. Check if the public certificate in the default secret is valid:

```bash
istioctl proxy-config secret deploy/web-api -n istioinaction -o json | jq '[.dynamicActiveSecrets[] | select(.name == "default")][0].secret.tlsCertificate.certificateChain.inlineBytes' -r | base64 -d | openssl x509 -noout -text | grep 'Validity' -A 2
```

You should see the public certificate is valid and expires in 24 hours:

```text
        Validity
            Not Before: Jun  8 13:39:16 2021 GMT
            Not After : Jun  9 13:39:16 2021 GMT
```

3. Validate the identity of the client certificate is correct:

```bash
istioctl proxy-config secret deploy/web-api -n istioinaction -o json | jq '[.dynamicActiveSecrets[] | select(.name == "default")][0].secret.tlsCertificate.certificateChain.inlineBytes' -r | base64 -d | openssl x509 -noout -text | grep 'Subject Alternative Name' -A 1
```

You should see the identity of the `web-api` service. Note it is using the spiffe format, e.g. `spiffe://{my-trust-domain}/ns/{namespace}/sa/{service-account}`:

```text
            X509v3 Subject Alternative Name: critical
                URI:spiffe://cluster.local/ns/istioinaction/sa/web-api
```

### Understand the SPIFFE format used by Istio

Where are the `cluster.local` and `web-api` values come from? Check the `istio` configmap in the `istio-system` namespace:

```bash
kubectl get cm istio -n istio-system -o yaml | grep trustDomain -m 1
```

You'll see the `cluster.local` returned as the trustDomain value, per the installation of your Istio using the demo profile.

```text
    trustDomain: cluster.local
```

If you review the `sample-apps/web-api.yaml` file, you will see the `web-api` service account in there.

```bash
cat sample-apps/web-api.yaml | grep ServiceAccount -A 3
```

```text
kind: ServiceAccount
metadata:
  name: web-api
---
```

### How does the `web-api` service obtain the needed key and/or certificates?

In lab 03, you reviewed the injected `istio-proxy` container for the `web-api` pod. Recall there are a few volumes mounted to the `istio-proxy` container.

```text
      volumeMounts`
      - mountPath: /var/run/secrets/istio
        name: istiod-ca-cert
      - mountPath: /var/lib/istio/data
        name: istio-data
      - mountPath: /etc/istio/proxy
        name: istio-envoy
      - mountPath: /var/run/secrets/tokens
        name: istio-token
      - mountPath: /etc/istio/pod
        name: istio-podinfo
      - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
        name: web-api-token-ztk5d
        readOnly: true
...
    - name: istio-token
      projected:
        defaultMode: 420
        sources:
        - serviceAccountToken:
            audience: istio-ca
            expirationSeconds: 43200
            path: istio-token
    - configMap:
        defaultMode: 420
        name: istio-ca-root-cert
      name: istiod-ca-cert
    - name: web-api-token-ztk5d
      secret:
        defaultMode: 420
        secretName: web-api-token-ztk5d
```

The `istio-ca-cert` mounted volumn is from the `istio-ca-root-cert` configmap in the `istioinaction` namespace. During start up time, Istio agent \(also called `pilot-agent`\) creates the private key for the `web-api` service and then sends the certificate signing request \(CSR\) for the Istio CA \(Istio control plane is the Istio CA in your installation\) to sign the private key, using the `istio-token` and the `web-api` service's service account token `web-api-token-ztk5d` as credentials. The Istio agent sends the certificates received from the Istio CA along with the private key to Envoy via the Envoy SDS API.

You noticed earlier that the certificate expires in 24 hours. What happens when the certificate expires? The Istio agent monitors the `web-api` certificate for expiration and repeats the CSR request process described above periodically to ensure each of your workload's certificate is valid.

### How is mTLS strict enforced?

When mTLS strict is enabled, you will find the Envoy configuration for the `istio-proxy` container actually has less lines of configurations. This is because when mTLS strict is enabled, we would only allow mTLS traffic thus no need to have any filter chain configurations to allow any plain text traffic to services in the mesh \(hint: search for `"transport_protocol": "raw_buffer"` in your Envoy configuration when `PERMISSIVE` mode is applied\). If you are curious to explore Envoy configuration for any of your pod, you can use the command below to view all of the Envoy proxy configuration for the `istio-proxy` container of the `web-api` pod.

```bash
istioctl proxy-config all deploy/web-api -n istioinaction -o json
```

### Question

We have only deployed a few services, why there are so much Envoy configuration for the pod? Istio listens to everything in your Kubernetes cluster by default, we will explore using discovery selectors, `exportTo` and `Sidecar` resources for operating Istio in a production environment in the Istio Essential and Expert workshops.

## Next lab

Congratulations, you have enabled strict mTLS policy for all services in the entire Istio mesh. We'll explore controlling traffic with these services in the [next lab](05-control-traffic.md).

