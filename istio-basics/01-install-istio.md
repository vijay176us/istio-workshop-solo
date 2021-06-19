# Lab 1 :: Install Istio

One of the quickest way to get started with Istio is to leverage the demo profile. The demo profile is designed to showcase Istio functionality with modest resource requirements. The demo profile contains Istio control plane \(also called Istiod\), Istio ingress-gateway and egress-gateway, and a few addon components. In this lab, you will install Istio with the demo profile. You will validate the installation is successfully and examinate the installation artifacts. You can use either istioctl, Helm or Istio operator to install Istio, in this lab you will use istioctl to install Istio.

## Download Istio

* Download the istio release binary:

```bash
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.10.0 sh -
```

* Add istioctl client to your path:

```bash
cd istio-1.10.0
export PATH=$PWD/bin:$PATH
```

* Check istioctl version:

<!--bash
pushd /usr/local/bin
  ln -s $(which istioctl)
popd
-->
```bash
istioctl version
```

* Check if your Kubernetes environment meets Istio's platform requirement:

```bash
istioctl x precheck
```

## Install Istio

* List available installation profiles:

```bash
istioctl profile list
```

* Since this is a get started workshop, you will use the demo profile to install Istio.

```bash
istioctl install --set profile=demo -y
```

{% hint style="info" %}
If your Kubernetes environment can only support Kubernetes Service of type `NodePort`, configure your Istio ingress gateway to use type `NodePort`

```text
istioctl install --set profile=demo -y --set values.gateways.istio-ingressgateway.type=NodePort
```
{% endhint %}

* You should see an output that indicate each Istio component is installed successfully. Check out the resources installed by Istio: 

<!--bash
kubectl wait --for=condition=Ready pod --all -n istio-system
-->
```bash
kubectl get all,cm,secrets,envoyfilters -n istio-system
```

* Check out Custom Resource Definitions \(CRDs\) installed by Istio:

```bash
kubectl get crds -n istio-system | grep istio.io
```

* Verify the installation using the following command:

```bash
istioctl verify-install
```

You should see the following at the end of the output to indicate that your Istio is installed successfully:

```
✔ Istio is installed and verified successfully
```

## Install Istio Telemetry Addons

Istio telemetry addons are shipped as samples because these addons are optimized for demo purpose and not for production usage. They provides a convenient way to install these telemetry components that integrate with Istio.

```bash
kubectl apply -f samples/addons
```

{% hint style="info" %}
If you hit an error like below, rerun the above command to ensure the `samples/addons` are applied to your Kubernetes cluster. The error below could happen when you attempt to install any `MonitoringDashboard` custom resource before the `MonitoringDashboard` CRD is installed.

```text
unable to recognize "samples/addons/kiali.yaml": no matches for kind "MonitoringDashboard" in version "monitoring.kiali.io/v1alpha1"
```
{% endhint %}

Wait till all pods in the `istio-system` have reached running:

<!--bash
kubectl wait --for=condition=Ready pod --all -n istio-system
-->
```bash
kubectl get pods -n istio-system
```

Enable the access to the Prometheus dashboard:

```text
istioctl dashboard prometheus
```

Visit [http://localhost:9090](http://localhost:9090) from your browser, you should be able to view the Prometheus UI. Press `ctrl+C` to end the prior `istioctl dashboard prometheus` command and use the command below to enable access to the Grafana dashboard: 

```text
istioctl dashboard grafana
```

Visit [http://localhost:3000](http://localhost:3000) from your browser, you should be able to view the Grafana UI. Press `ctrl+C` to end the prior `istioctl dashboard grafana` command and use the command below to enable access to the Jaeger dashboard:

```text
istioctl dashboard jaeger
```

Visit [http://localhost:16686](http://localhost:16686) from your browser, you should be able to view the Jaeger UI. Press `ctrl+C` to end the prior `istioctl dashboard jaeger` command and use the command below to enable access to the Kiali dashboard:

```text
istioctl dashboard kiali
```

Visit [http://localhost:20001/kiali](http://localhost:20001/kiali) from your browser, you should be able to view the Kiali UI. Press `ctrl+C` to end the prior `istioctl dashboard kiali` command.  You will not see much telemetry data from any of these dashboards, as we don't have any services in the Istio service mesh yet. You will revisit these dashboards in the [lab 03](03-add-services-to-mesh.md).

## Next lab

Congratulations, you have installed Istio control plane \(Istiod\), Istio ingress-gateway and egress-gateway and its addon components successfully. We'll learn to expose your services to Istio ingress gateway securely in the [next lab](02-secure-service-ingress.md).

