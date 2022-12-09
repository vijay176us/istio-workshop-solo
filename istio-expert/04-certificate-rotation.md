# Lab 4 :: Certificate rotation and PKI integration

One important capability that Istio provides is workload identity. With workload identity, we can encode an identity into a verifiable document and enforce authentication and authorization policies around this identity. Istio uses x509 certificates and SPIFFE to implement identity and uses this mechanism to accomplish two important security practices: implement authentication and encrypt the transport (TLS/mTLS). With these foundational pieces in place, we can strongly secure all traffic between services.

Istio implements CA capabilities in its control plane component istiod. This component is responsible for bootstrapping the CA, serving a gRPC endpoint that can take Certificate Signing Requests (CSRs), and handling the signing of requests for new certificates or rotated certificates.

Out of the box, Istio’s CA will automatically create a signing key/certificate on bootstrap that’s valid for 10 years. This “root” key will then be used to anchor all trust in the system by signing the workload CSRs and establishing the root certificate as the trusted CA.

If you’re just exploring Istio, this default root CA should be sufficient. If you’re setting up for a live system, you should probably not use the built-in, self-signed root. In fact, you likely already have PKI in your organization and would be able to introduce Intermediate certificates that can be used for Istio workload signing. These intermediates are signed by your existing trusted Roots.

## Understand Istio's certificate rotation

When your workload certificate expires?

Workload certificate typically expires 24 hours. For example, you can query the `sleep` pod to get the certificate expiration (hint: check the `Validity` value):

```
kubectl label namespace default istio.io/rev=1-12-1
kubectl apply -f sample-apps/sleep.yaml
kubectl apply -f sample-apps/web-api.yaml
kubectl wait --for=condition=Ready pod --all -n default
istioctl pc secret deploy/sleep -o json | jq '[.dynamicActiveSecrets[] | select(.name == "default")][0].secret.tlsCertificate.certificateChain.inlineBytes' -r | base64 -d | openssl x509 -noout -text
```

How often does the workload certificate rotate?

By default, it rotates every 12 hours (half of the total validity duration which is 24 hours). This is configurable in Istio via the `SECRET_GRACE_PERIOD_RATIO` environment variable on the [`pilot-agent` component](https://istio.io/latest/docs/reference/commands/pilot-agent/), which has the default value of `0.5`.

How workload certification rotation works?

During workload start up time, Istio agent (also called pilot-agent) creates the private key for the sleep service and then sends the certificate signing request (CSR) for the Istio CA (by default, Istiod is the Istio CA) to sign the private key, using the istio-token and the sleep service's service account token as credentials. The Istio agent sends the certificates received from the Istio CA along with the private key to the Envoy Proxy via the Envoy SDS API.

The Istio agent monitors the workload certificate for expiration and repeats the CSR process when it is time to refresh the certificate (every 12 hours by default) to ensure each of your workload's certificate is still valid.

What about root certificate?

Root certificate typically doesn't expire until much longer, by default it is 10 years in Istio.
Root certificate configmap is made avail in each namespace in your Kubernetes cluster. Check the configmap that starts with `istio-ca`:

```bash
kubectl get cm -A | grep istio-ca
```

From the output below, you can see Istio control plane has created `istio-ca-root-cert` configmap in each of the namespaces in the cluster, excluding a few system namespaces.

```
default              istio-ca-root-cert                    1      3m28s
istio-ingress        istio-ca-root-cert                    1      3m25s
istio-system         istio-ca-root-cert                    1      3m28s
```

Take a look at the root certificate:

```bash
kubectl get cm istio-ca-root-cert -o json | jq '[.data[]][0]' -r | openssl x509 -noout -text
```

You'll see the confirm the issuer, validity, subject, public key info from the root certificate.

## Cert Manager

Mostly likely, you already have PKI in your organization and you do not want to use the built-in, self-signed root. In fact, you probably want to use Intermediate certificates that are signed by your existing trusted roots. These trusted roots are commonly generated from a cert manager, that can be integrated with a lot of backend PKI --- which is a big reason why cert manager is so popular.

Cert-manager is already installed from lab 03. In this lab, you will install the [`istio-csr`](https://github.com/cert-manager/istio-csr) project and use it as your CA instead which can connect to various backend PKI via cert manager.

1. Install the Issuer and certificate resources. For simplicity, this lab will use the self signed issuer but you'd be most likely use Vault or other issuer that integrates with your PKI.

```bash
cat labs/04/cert-manager-issuer-and-cert.yaml
```

```bash
kubectl apply -f labs/04/cert-manager-issuer-and-cert.yaml
```

2. Export root ca to a local file:

```
# Export our cert from the secret it's stored in, and base64 decode to get the PEM data.
kubectl get -n istio-system secret istio-ca -ogo-template='{{index .data "tls.crt"}}' | base64 -d > ca.pem

# Out of interest, we can check out what our CA looks like
openssl x509 -in ca.pem -noout -text

# Add our CA to a secret
kubectl create secret generic -n cert-manager istio-root-ca --from-file=ca.pem=ca.pem
```

3. Prepare Istio environment for switching to a different root certificate. Unfortunatelly, this often comes with some Istio control plane downtime, so plan this carefully.

```bash
kubectl delete secret istio-ca-secret -n istio-system
kubectl scale deployment istiod-1-12-1 -n istio-system --replicas 0
```

Check what secret you have in the `istio-system` namespace to ensure the new secret is created and `istio-ca-secret` is removed from the `istio-system` namespace:

```bash
kubectl get secret -n istio-system
```

You'll see the newly created secret called `istio-ca`, which is the root CA you will use for your Istio control plane:

```
NAME                                       TYPE                                  DATA   AGE
default-token-tftxc                        kubernetes.io/service-account-token   3      3m48s
istio-ca                                   kubernetes.io/tls                     3      53s
istio-reader-service-account-token-mh7dl   kubernetes.io/service-account-token   3      3m47s
istiod-1-12-1-token-xcd9z                  kubernetes.io/service-account-token   3      3m46s
istiod-service-account-token-pnf86         kubernetes.io/service-account-token   3      3m47s
```

4. Install Cert Manager Istio CSR. Note the `istio-root-ca` secret is from the `cert-manager` namespace and you configure revision 1-12-1 during the installation of the istio-csr so it knows to configure the webhook certificate for istiod-1-12-1:

```
helm install -n cert-manager cert-manager-istio-csr jetstack/cert-manager-istio-csr \
	--set "app.tls.rootCAFile=/var/run/secrets/istio-csr/ca.pem" \
	--set "volumeMounts[0].name=root-ca" \
	--set "volumeMounts[0].mountPath=/var/run/secrets/istio-csr" \
	--set "volumes[0].name=root-ca" \
	--set "volumes[0].secret.secretName=istio-root-ca" \
  --set "app.istio.revisions={default,1-12-1}"
```

Confirm that the istio-csr pod is running and ready:

```
kubectl get pods -n cert-manager
```

5. Review the update to the Istio control plane's IstioOperator spec for the following changes:
-  Disable the built in Istio CA
-  Configure Istio data plane to send CSR request to the `istio-csr` service you deployed earlier.
-  Mount certs and keys from the Cert Manager Certificates you created earlier

```bash
cat labs/04/control-plane-cert-manager.yaml
```

Apply the Istio control plane update, and the ingress gateway update to switch to use the updated control plane:

```bash
istioctl install -y -n istio-system -f labs/04/control-plane-cert-manager.yaml --revision=1-12-1
kubectl scale deployment istiod-1-12-1 -n istio-system --replicas 1
istioctl install -y -n istio-ingress -f labs/01/ingress-gateway.yaml --revision 1-12-1
```

6. Deploy `web-api-canary` and `sleep-canary` to test with the new Istiod that delegates istio-csr as the CA:

```bash
kubectl apply -f  labs/04/sleep-canary.yaml
kubectl apply -f labs/04/web-api-canary.yaml
```

Confirm the canary pods are up running:

```bash
kubectl get pods
```

What about the `sleep` and `web-api` pods? They would not be able to communicate to the Istio control plane due to the invalid workload certs. You can confirm this by reviewing the logs or check the proxy status:

```bash
istioctl proxy-status
```

From the output, note the `sleep` and `web-api` pods are not in the list below because the pods are no longer connected to the Istio control plane when using the workload certs issued from the prior root CA.

```
NAME                                                         CDS        LDS        EDS        RDS          ISTIOD                             VERSION
istio-ingressgateway-test-7c974db9c6-b24zl.istio-ingress     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-1-12-1-7d956cb854-7t9td     1.12.1
sleep-canary-6fb84cbcf-26bbr.default                         SYNCED     SYNCED     SYNCED     SYNCED       istiod-1-12-1-7d956cb854-7t9td     1.12.1
web-api-canary-6d87d9c4-ndcvp.default                        SYNCED     SYNCED     SYNCED     SYNCED       istiod-1-12-1-7d956cb854-7t9td     1.12.1
```

The `sleep` pod log confirms its proxy having trouble to connect to Istiod:

```bash
kubectl logs deploy/sleep -c istio-proxy
```

You'll see many warnings in the log, like below:

```
2022-02-04T03:32:43.998066Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed since 32018s ago: 14, connection error: desc = "transport: authentication handshake failed: x509: certificate signed by unknown authority"
```

You can fix it by restarting the `sleep` and `web-api` pods:

```bash
kubectl rollout restart deploy/sleep
kubectl rollout restart deploy/web-api
kubectl wait --for=condition=Ready pod --all -n default
```

Rerun the `proxy-status` command, you will see the `sleep` and `web-api` pod proxy status now synched:

```bash
istioctl proxy-status
```

Clean up the canary deployments, and confirm the trustCA of the certificate matches the Istio root CA file (e.g. `ca.pem`) that you created earlier:

```bash
kubectl delete deploy sleep-canary web-api-canary
# base 64 decode the root certificate for sleep's workload cert
istioctl pc secret deploy/sleep -o json | jq '[.dynamicActiveSecrets[] | select(.name == "ROOTCA")][0].secret.validationContext.trustedCa.inlineBytes' -r | base64 -d
# display ca.pem file for comparison
cat ca.pem

```

You should see the CA root certificate content to be the same. Congratulations, you have updated your Istio control plane and data plane to use the newly issued root CA from cert-manager!

### Some Challenges

While above approach using istio-csr is interesting, many of our users have shared with us that the CA private keys need to be present in the Kubernetes cluster is a security concern for them. If you recall, you viewed root certificate in the `istio-ca` secret earlier and the private key is also in that secret:

```bash
kubectl get -n istio-system secret istio-ca -ogo-template='{{index .data "tls.key"}}' | base64 -d
```

How do you fix this if this is a security concern for you?
## Kubernetes CSR

In addition to the istio-csr project, Istio also provides experiemental support for [custom CA integration using Kubernetes CSR](https://istio.io/latest/docs/tasks/security/cert-management/custom-ca-k8s/). This approach simply configures Istio to use an external signer that implements the Kubernetes CSR interface. However, this requires writing Kubernetes controller for the custom CA, which watches the CertificateSigningRequest and Certificate Resources and act on them.

## Gloo Mesh Enterprise

Based on what we learned from working with large enterprise customers, we enhanced Gloo Mesh enterprise to enable Vault as an intermediate CA provider without persisting the private key anywhere in the Kubernetes cluster. We implemented an `istiod-agent` sidecar to Istiod so the private key is only in memory and not persisted. Check out the [Gloo Mesh documentation](https://docs.solo.io/gloo-mesh-enterprise/main/setup/certs/vault-certs/vault-istio/) to learn more about this.
## Next Lab

Congratulations, you have learned how Istio workload certificates work, how to plugin your own CA using the `istio-csr` project and other CA integrations with Istio such as external issuer that implementes the Kubernetes CSR interface or Gloo Mesh's istio-agent sidecar to Istiod. In the next lab, we will dive into multicluster Istio deployments.

