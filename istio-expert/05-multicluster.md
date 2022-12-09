# Lab 5 :: Istio Multi-Cluster (with endpoint discovery)

In this lab, we explore some of Istio's multi-cluster capabilities. Istio actually has a few different approaches to multi-cluster [as you can see in the documentation](https://istio.io/latest/docs/setup/install/multicluster/) but we recommend you chose an approach that favors running multiple control planes (starting with one per cluster and optimizing from there). 

Multi-cluster service-mesh architectures are extremely important to enable things like high availability, failover, isolation, and regulatory/compliance. We have found the teams that chose to run a single large cluster with a single Istio control plane (or try run a single Istio control plane across multiple clusters) have issues with tenancy, blast radius, and overall stability of their platform long-term. If you run multiple clusters, you will likely need your service mesh deployed accordingly. 

In the model we explore here in this lab, we will deploy an individual Istio control plane per cluster and then connect them through a remote-access protocol. 

This is the approach suggested in the [Multi-Primary](https://istio.io/latest/docs/setup/install/multicluster/multi-primary/) documentation on the Istio website, *however*, **THERE ARE SOME SERIOUS DRAWBACKS TO THIS APPROACH**, specifically in the **area of security posture**. 

We will walk through this set up and then discuss the drawbacks at the end. 

Let's dive in!

## A multi-cluster environment

... i'll need help here... we need 3 clusters for this lab with kube context named `istio-1`, `istio-2`, `istio-3` respectively.


## Installing Istio across multiple clusters

Let's install Istio across multiple clusters. Note, the various clusters are referred to as `$CLUSTER1`, `$CLUSTER2`, and `$CLUSTER3` with the following values:

```bash
export CLUSTER1="istio-1"
export CLUSTER2="istio-2"
export CLUSTER3="istio-3"
```

We will create the namespaces for each of the installations into their respective clusters. Note, we will also label the namespaces with an indication of which network to which they belong. Istio uses a network designator to know whether a service communication can (or should) cross a network boundary. Perform the following prep for our installation:

```bash
kubectl --context ${CLUSTER1} create ns istio-system
kubectl --context ${CLUSTER1} label namespace istio-system topology.istio.io/network=network1

kubectl --context ${CLUSTER2} create ns istio-system
kubectl --context ${CLUSTER2} label namespace istio-system topology.istio.io/network=network1

kubectl --context ${CLUSTER3} create ns istio-system
kubectl --context ${CLUSTER3} label namespace istio-system topology.istio.io/network=network1
```

### Install secrets

Before we actually do the installation of Istio, we'll make sure to use an intermediate signing certificate rooted in the same Root CA (see Lab 04 for more on certificates and rotation). We will set up these secrets ahead of time named `cacerts` in the `istio-system` namespace. 

```bash
kubectl --context ${CLUSTER1} create secret generic cacerts -n istio-system \
--from-file=labs/05/certs/cluster1/ca-cert.pem \
--from-file=labs/05/certs/cluster1/ca-key.pem 
--from-file=labs/05/certs/cluster1/root-cert.pem \
--from-file=labs/05/certs/cluster1/cert-chain.pem

kubectl --context ${CLUSTER2} create secret generic cacerts -n istio-system   \
--from-file=labs/05/certs/cluster2/ca-cert.pem  \
--from-file=labs/05/certs/cluster2/ca-key.pem  \
--from-file=labs/05/certs/cluster2/root-cert.pem \
--from-file=labs/05/certs/cluster2/cert-chain.pem

kubectl --context ${CLUSTER3} create secret generic cacerts -n istio-system  \
--from-file=labs/05/certs/cluster3/ca-cert.pem  \
--from-file=labs/05/certs/cluster3/ca-key.pem  \
--from-file=labs/05/certs/cluster3/root-cert.pem \
--from-file=labs/05/certs/cluster3/cert-chain.pem
```

When the Istio control plane comes up in each of these clusters, it will use this intermediate signing CA to issue workload certificates to each of the running applications and associated sidecar. Although the workloads in each of the clusters is issuing leaf certificates signed by different signing CAs, they are all rooted in the same root CA, so traffic can be verified and trusted. 

### Install Istio with ew-gw on all three clusters

Now that we've correctly configured the namespaces and the signing certificates for each control plane, let's use the `istioct install` command to install each of the control planes and supporting components:

```bash
istioctl install -y --context ${CLUSTER1} -f ./labs/05/istio/cluster1.yaml
istioctl install -y --context ${CLUSTER1} -f ./labs/05/istio/ew-gateway1.yaml
kubectl --context=${CLUSTER1} apply -n istio-system -f ./labs/05/istio/expose-services.yaml
```

You can see we install the Istio control plane as well as an `east-west gateway` that will be used to connect traffic between clusters. You can run the following command to verify that these components came up successfully:

```bash
kubectl --context ${CLUSTER1} get po -n istio-system


NAME                                     READY   STATUS    RESTARTS   AGE
istio-eastwestgateway-85c5855c76-6swpx   1/1     Running   0          131m
istiod-76ddb688f7-2trh9                  1/1     Running   0          132m
```

If that looks good, continue on installing a separate control plane into each of cluster 2 and cluster 3:

```bash
istioctl install -y --context ${CLUSTER2} -f ./istio/cluster2.yaml
istioctl install -y --context ${CLUSTER2} -f ./istio/ew-gateway2.yaml
kubectl --context=${CLUSTER2} apply -n istio-system -f ./labs/05/istio/expose-services.yaml

istioctl install -y --context ${CLUSTER3} -f ./istio/cluster3.yaml
istioctl install -y --context ${CLUSTER3} -f ./istio/ew-gateway3.yaml
kubectl --context=${CLUSTER3} apply -n istio-system -f ./labs/05/istio/expose-services.yaml
```

At this point, all we've done is set up separate Istio control planes into each of the clusters. We could deploy workloads into each of the clusters but they would be isolated in the mesh. Let's see what steps we need to connect the various control planes and service-mesh networks.

### Set up Endpoint Discovery

In this section, we will use the `istioctl` cli to automate setting up connectivity between each of the clusters. We do this by running `istioctl x create-remote-secret` for each of the other peer clusters. Let's run the following commands and then explore what happened:

```bash
istioctl x create-remote-secret  --context=${CLUSTER2} --name=cluster2 | kubectl apply -f - --context=${CLUSTER1}
istioctl x create-remote-secret  --context=${CLUSTER3} --name=cluster3 | kubectl apply -f - --context=${CLUSTER1}
```

These two commands will create a kube-config secret for each of cluster 2 and cluster 3 and store it into the `istio-system` namespace in cluster 1. After running these commands, let's verify what was created:

```bash
kubectl --context ${CLUSTER1} get secret -n istio-system
NAME                                           TYPE                                  DATA   AGE
cacerts                                        Opaque                                4      135m
default-token-npwbd                            kubernetes.io/service-account-token   3      135m
istio-eastwestgateway-service-account-token    kubernetes.io/service-account-token   3      135m
istio-reader-service-account-token-pwf7h       kubernetes.io/service-account-token   3      135m
istio-remote-secret-cluster2                   Opaque                                1      134m
istio-remote-secret-cluster3                   Opaque                                1      134m
istiod-service-account-token-gqlwc             kubernetes.io/service-account-token   3      135m
istiod-token-n6hs8                             kubernetes.io/service-account-token   3      135m
```

You can see a secret called `istio-remote-secret-cluster2` and `istio-remote-secret-cluster3`. Let's take a look at one of these secrets:

```bash
kubectl --context ${CLUSTER1} get secret istio-remote-secret-cluster2 -n istio-system -o yaml
```

Woah! That's a lot of... hidden stuff... let's take a look at what is in the `cluster2` key in the secret:

```bash
kubectl --context istio-1 get secret istio-remote-secret-cluster2 -n istio-system -o jsonpath="{.data.cluster2}" | base64 --decode
```

You should see an output like this (with the keys/certs redacted in this listing for brevity):

```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: 
	< CERT DATA REDACTED >
    server: https://10.128.0.37:7002
  name: cluster2
contexts:
- context:
    cluster: cluster2
    user: cluster2
  name: cluster2
current-context: cluster2
kind: Config
preferences: {}
users:
- name: cluster2
  user:
    token: < TOKEN DATA REDACTED >
```

Interesting... isn't this a bit familiar??

If you said "this is a kube-config file for accessing a cluster" you would be correct!

For this form of multi-cluster, Istio creates a kube-config for each of the remote clusters with direct access to the Kubernetes API of the remote cluster. We will come back to this at the end of the lab, but for now, let's continue enabling endpoint discovery (through the Kubernetes API) for each of the other clusters:

```bash
istioctl x create-remote-secret  --context=${CLUSTER1} --name=cluster1  | kubectl apply -f - --context=${CLUSTER2}
istioctl x create-remote-secret  --context=${CLUSTER3} --name=cluster3 | kubectl apply -f - --context=${CLUSTER2}

istioctl x create-remote-secret  --context=${CLUSTER1} --name=cluster1 | kubectl apply -f - --context=${CLUSTER3}
istioctl x create-remote-secret  --context=${CLUSTER2} --name=cluster2 | kubectl apply -f - --context=${CLUSTER3}
```


At this point, multi-cluster service and endpoint discovery has been enabled. We don't have any sample apps deployed, yet, so let's take a look at that:

## Set up sample apps

Let's set up a couple of sample applications to test our multi-cluster connectivity. First, let's create the right namespaces and label them for injection on all of the clusters:

```bash
kubectl --context ${CLUSTER1} create ns sample
kubectl label --context=${CLUSTER1} namespace sample istio-injection=enabled

kubectl --context ${CLUSTER2} create ns sample
kubectl label --context=${CLUSTER2} namespace sample istio-injection=enabled

kubectl --context ${CLUSTER3} create ns sample
kubectl label --context=${CLUSTER3} namespace sample istio-injection=enabled
```

Next, let's deploy services into each of the clusters:

```bash
kubectl apply --context=${CLUSTER1} -f ./labs/05/istio/helloworld.yaml -l service=helloworld -n sample
kubectl apply --context=${CLUSTER2} -f ./labs/05/istio/helloworld.yaml -l service=helloworld -n sample
kubectl apply --context=${CLUSTER3} -f ./labs/05/istio/helloworld.yaml -l service=helloworld -n sample


kubectl apply --context=${CLUSTER1} -f ./labs/05/istio/helloworld.yaml -l version=v1 -n sample
kubectl apply --context=${CLUSTER2} -f ./labs/05/istio/helloworld.yaml -l version=v1 -n sample
kubectl apply --context=${CLUSTER3} -f ./labs/05/istio/helloworld.yaml -l version=v2 -n sample
```

Lastly, let's deploy a sample "sleep" application into each cluster that we will use as the "client" to connect to services across clusters:

```bash
kubectl apply --context=${CLUSTER1} -f ./labs/05/istio/sleep.yaml -n sample
kubectl apply --context=${CLUSTER2} -f ./labs/05/istio/sleep.yaml -n sample
kubectl apply --context=${CLUSTER3} -f ./labs/05/istio/sleep.yaml -n sample
```


From the `sleep` client in cluster 1, let's make a call to the `helloworld` service:

```bash
kubectl exec --context ${CLUSTER1} -n sample -c sleep deploy/sleep -- curl -sS helloworld.sample:5000/hello

Hello version: v1, instance: helloworld-v1-776f57d5f6-b946k
```

You should see the `helloworld` that's deployed in the same cluster respond. Let's take a look at the endpoints that the `sleep` pod's Istio service proxy knows about

```bash
istioctl --context $CLUSTER1 pc endpoints deploy/sleep.sample --cluster "outbound|5000||helloworld.sample.svc.cluster.local"


ENDPOINT                STATUS      OUTLIER CHECK     CLUSTER
10.101.171.136:5000     HEALTHY     OK                outbound|5000||helloworld.sample.svc.cluster.local
172.18.2.1:15443        HEALTHY     OK                outbound|5000||helloworld.sample.svc.cluster.local
172.18.3.1:15443        HEALTHY     OK                outbound|5000||helloworld.sample.svc.cluster.local
```
You can see from this output that Istio's service proxy knows about three endpoints for the `helloworld` service. One of the endpoints is local (10.101.x.x) while two of them are remote (172.18.2.x and 172.18.3.x). What are those remote IP addresses?

Those are the External IP addresses of the east-west gateways in cluster 2 and cluster 3! Let's verify:

Cluster 2:

```bash
kubectl --context $CLUSTER2 get svc -n istio-system
NAME                    TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)                                                           AGE
istio-eastwestgateway   LoadBalancer   10.2.99.180   172.18.2.1    15021:31979/TCP,15443:32356/TCP,15012:30091/TCP,15017:30379/TCP   157m
istiod                  ClusterIP      10.2.71.233   <none>        15010/TCP,15012/TCP,443/TCP,15014/TCP                             158m
```

Cluster 3:

```bash
kubectl --context $CLUSTER3 get svc -n istio-system
NAME                    TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)                                                           AGE
istio-eastwestgateway   LoadBalancer   10.3.44.77    172.18.3.1    15021:31593/TCP,15443:30335/TCP,15012:31608/TCP,15017:32168/TCP   158m
istiod                  ClusterIP      10.3.47.235   <none>        15010/TCP,15012/TCP,443/TCP,15014/TCP   
```

Istio has discovered `helloworld` services running on cluster 2 and cluster 3 and has correctly configured the remote endpoints as the east-west gateways in the remote clusters. The east-west gateway in the respective clusters can then route to the local `helloworld` services and maintain an end-to-end mTLS connection. Let's verify the connectivity works as we expect:

## Verify installation


For each cluster, let's make a call to the `helloworld` service a few times and watch the load get balanced:

```bash
echo "Calling from cluster 1"
for i in {1..10}
do
    kubectl exec --context ${CLUSTER1} -n sample -c sleep deploy/sleep -- curl -sS helloworld.sample:5000/hello
done
```


```bash
echo "Calling from cluster 2"
for i in {1..10}
do
    kubectl exec --context ${CLUSTER2} -n sample -c sleep deploy/sleep -- curl -sS helloworld.sample:5000/hello
done
```

```bash
echo "Calling from cluster 3"
for i in {1..10}
do
    kubectl exec --context ${CLUSTER3} -n sample -c sleep deploy/sleep -- curl -sS helloworld.sample:5000/hello
done
```

## Drawbacks to this approach

In this lab we explored one of Istio's multi-cluster approaches called "Multi-Primary" with endpoint discovery. As we saw, with this approach we create tokens/kube-configs in each of the clusters to give direct Kubernetes API access to each of the remote clusters. We have found organizations are hesitant to give direct Kubernetes API access to each of the clusters in a multi-cluster deployment. The access is "read-only" (for the most part.. not completely) and is a major risk: what if one of the clusters gets compromised? Do they have access to all of the other clusters in the fleet?

Another drawback we've seen with this approach is the lack of control over what constitutes a service across multiple clusters. Using the endpoint-discovery mechanism, services are matched based on name and must be in the same namespace. What if services don't run in the same namespace across clusters? This scenario is quite common, and would not work for the approach to multi-cluster as presented here.

At Solo.io, we have built our service mesh using Istio but NOT relying on this endpoint-discovery model. We do use the multi-primary control plane deployment, but instead of proliferating Kubernetes credentials to every cluster, we have a global service-discovery mechanism that uses information to then inform Istio which services to federate and how. This approach uses `ServiceEntry`s and does not rely on sharing Kubernetes credentials across clusters. Additionally, with Gloo Mesh, you get more control over what services are included in a "virtual destination" by specifying labels/namespaces/clusters and services can reside in other namespaces. We also do a lot of the certificate federation for you and handle rotation so to minimize the operational burden. Please see the [Gloo Mesh workshop](https://www.solo.io/events/workshops/) for more details about how this works. 


## Wrap up, what's next?

Thanks for joining our Advanced Istio Workshop for Operationalizing Istio for Day 2! We hope it was able to show you powerful ways of leveraging Istio in your environment. That's the the end, however. The next step is to go to the next level and see what best practices for operating Istio in multi-tenant, multi-cluster architectures look like with Gloo Mesh. Please register [for an upcoming Gloo Mesh](https://www.solo.io/events/workshops/) workshop which will show how to *simplify* the operations and usage of Istio. 

