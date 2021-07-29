# Lab 2 :: Certificate rotation and PKI integration

One important capability that Istio provides is workload identity. With workload identity, we can encode an identity into a verifiable document and enforce authentication and authorization policies around this identity. Istio uses x509 certificates and SPIFFE to implement identity and uses this mechanism to accomplish two important security practices: implement authentication and encrypt the transport (TLS/mTLS). With these foundational pieces in place, we can strongly secure all traffic between services.

Istio implements CA capabilities in its control plane component istiod. This component is responsible for bootstrapping the CA, serving a gRPC endpoint that can take Certificate Signing Requests (CSRs), and handling the signing of requests for new certificates or rotated certificates.

Out of the box, Istio’s CA will automatically create a signing key/certificate on bootstrap that’s valid for 10 years. This “root” key will then be used to anchor all trust in the system by signing the workload CSRs and establishing the root certificate as the trusted CA.

If you’re just exploring Istio, this default root CA should be sufficient. If you’re setting up for a live system, you should probably not use the built-in, self-signed root. In fact, you likely already have PKI in your organization and would be able to introduce Intermediate certificates that can be used for Istio workload signing. These intermediates are signed by your existing trusted Roots.

## Cert Manager


1. Install Cert Manager
```
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.4.0/cert-manager.yaml
```

2. Install Cert Manager Istio CSR

```
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install -n cert-manager cert-manager-istio-csr jetstack/cert-manager-istio-csr
```

3.
```
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: selfsigned
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: istio-ca
spec:
  isCA: true
  duration: 2160h # 90d
  secretName: istio-ca
  commonName: istio-ca
  subject:
    organizations:
    - cluster.local
    - cert-manager
  issuerRef:
    name: selfsigned
    kind: Issuer
    group: cert-manager.io
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: istio-ca
spec:
  ca:
    secretName: istio-ca
```

4. Update IstioOperator to:
-  Disable Istio CA
-  Configure Istio workloads request certificates from the cert-manager agent
-  Mount certs and keys from the Cert Manager Certificates

```
istioctl install -y -n istio-system -f labs/01/control-plane-cert-manager.yaml --revision=1-10-1
```

5. Verify


TODO: Talk about GM Vault