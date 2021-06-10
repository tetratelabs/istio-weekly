# Multiple gateways

Prerequisites:
- Kubernetes cluster
- VM running in the same network as the Kubernetes cluster

To get started we'll initialize the Istio operator first. To do that we can run the istioctl operator init command.

```sh
istioctl operator init
```

The init command will create the operator in the istio-operator namespace as well as the namespaces the operator watches -- since we haven't provided any watched namespaces, it defaults to `istio-system`; we can look at the actual operator Pod in that namespace:

```sh
kubectl get po -n istio-operator
```
Next, we can create the IstioOperator resource to install Istio. We'll use the default profile - that profile includes the `istiod` and a public `istio-ingressgateway`:

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
  name: default-installation
spec:
  profile: default
```

Save the above to `default-installation.yaml` and create the resource with `kubectl apply -f default-installation.yaml`.

We can check the status of the installation by listing the Istio operator resource:

```
kubectl get iop -A 
```

With Istio installed, let's create a namespace called `internal` and an operator that will install internal gateway into that namespace:

```
kubectl create ns internal
```

Now we can create another operator that installs an internal gateway to that namespace:

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
  name: internal-gw
spec:
  profile: empty
  components:
    ingressGateways:
      - namespace: internal
        name: ilb-gateway
        enabled: true
        label:
          istio: internal-ingressgateway
        k8s:
          serviceAnnotations:
            networking.gke.io/load-balancer-type: "Internal"
```

Save the above YAML to `internal-gw.yaml` and deploy it using `kubectl apply -f internal-gw.yaml`.

Just like before, we can check the status by listing the operator resources in all namespaces:

```sh
kubectl get iop -A 
```

If we look at the services (`kubectl get svc -n internal`) we'll notice that the external IP address is a private IP address that's only accessible within the same network.

Let's also create a Gateway resource for this internal ingress gateway:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: internal-gateway
  namespace: internal
spec:
  selector:
    istio: internal-ingressgateway
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - '*'
```

Save the above YAML to `internal-gateway.yaml` and create it using `kubectl apply -f internal-gateway.yaml`.

We'll use the `httpbin` as an example workload in the internal namespace. Let's label the namespace for injection and then deploy httpbin to that namespace:

```sh
kubectl label ns internal istio-injection=enabled
kubectl apply -f https://raw.githubusercontent.com/istio/istio/master/samples/httpbin/httpbin.yaml -n internal
```

Finally, we can create the VirtualService and attach the `internal-gateway` resource to it:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
  namespace: internal
spec:
  hosts:
    - '*'
  gateways:
    - internal-gateway
  http:
    - route:
      - destination:
          host: httpbin.internal.svc.cluster.local
          port:
            number: 8000
```

Save the above to `httpbin-vs.yaml` and create it using `kubectl apply -f httpbin-vs.yaml`.

To test the internal gateway, create a VM instance, then SSH into the instance and run `curl [internal-gateway-IP]`.