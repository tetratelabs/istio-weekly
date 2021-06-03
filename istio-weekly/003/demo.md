---
title: DEMO: Setting up SSL certificates on Istio ingress gateway
date: 05/27/2021
---

## Prerequisites

You'll need a Kubernetes cluster with [Istio installed](https://istio.tetratelabs.io/). You'll also need a domain name if you're doing the second and third scenario in the demo.

## Deploying a sample Hello World application

Create the deployment and service:
 
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: hello-world
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
  labels:
    app: hello-world
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      serviceAccountName: hello-world
      containers:
        - image: gcr.io/tetratelabs/hello-world:1.0.0
          imagePullPolicy: Always
          name: svc
          ports:
            - containerPort: 3000
---
kind: Service
apiVersion: v1
metadata:
  name: hello-world
  labels:
    app: hello-world
spec:
  selector:
    app: hello-world
  ports:
    - port: 80
      name: http
      targetPort: 3000
```

Save the above to `hello-world.yaml` and deploy it using `kubectl apply -f hello-world.yaml`.


We also need to deploy a public gateway and a virtual service to expose the Hello world application through the ingress gateway:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: public-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - '*'
```

Save the above YAML to `gateway.yaml` and deploy it using `kubectl apply -f gateway.yaml`.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: helloworld
spec:
  hosts:
    - '*'
  gateways:
    - public-gateway
  http:
    - route:
      - destination:
          host: hello-world.default.svc.cluster.local
          port:
            number: 80
```

Save the above YAML to `vs.yaml` and deploy it using `kubectl apply -f vs.yaml`.

Finally, let's get the external IP address of the ingress gateway:

```sh
export INGRESS_IP=$(kubectl get svc istio-ingressgateway -n istio-system -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
```

If you run `curl $INGRESS_IP` you should get back a "Hello World" response.

## Using self-signed certificate

In this scenario we are manually creating a self-signed certificate. First thing - pick a domain you want to use - note that to test this, you don’t have to own an actual domain name because we will use a self-signed certificate.

I will use `tetratelabs.dev` as my domain name and I'll use a subdomain called `hello`. So we will configure the gateway with a host called `hello.tetratelabs.dev` and present the self-signed certificate.

Let's store that value in a variable because we will use it throughout this example.

```sh
export DOMAIN_NAME=tetratelabs.dev
```

I will also create a separate folder to hold the root certificate and the private key we will create.

```sh
mkdir -p tetratelabs-certs
```

### Creating the cert files

Next we will create the root certificate called `tetratelabs.dev.crt` and the private key used for signing the certificate (file `tetratelabs.dev.key`):

```sh
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=$DOMAIN_NAME Inc./CN=$DOMAIN_NAME' -keyout $DOMAIN_NAME.key -out $DOMAIN_NAME.crt
```

The next step is to create the certificate signing request and the corresponding key: 

```sh
openssl req -out hello.$DOMAIN_NAME.csr -newkey rsa:2048 -nodes -keyout hello.$DOMAIN_NAME.key -subj "/CN=hello.$DOMAIN_NAME/O=hello world from $DOMAIN_NAME"
```

Finally using the certificate authority and it's key as well as the certificate signing requests, we can create our own self-signed certificate: 

```sh
openssl x509 -req -days 365 -CA $DOMAIN_NAME.crt -CAkey $DOMAIN_NAME.key -set_serial 0 -in hello.$DOMAIN_NAME.csr -out hello.$DOMAIN_NAME.crt
```

### Creating the Kubernetes secret

Now that we have the certificate and the correspondig key we can create a Kubernetes secret to store them in our cluster. 

We will create the secret in the `istio-system` namespace and reference it from the Gateway resource:

```sh
kubectl create -n istio-system secret tls tetratelabs-credential --key=hello.tetratelabs.dev.key --cert=hello.tetratelabs.dev.crt
```

With secret in place, let's update the Gateway resource:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: public-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 443
        name: https
        protocol: HTTPS
      tls:
        mode: SIMPLE
        credentialName: tetratelabs-credential
      hosts:
        - hello.tetratelabs.dev
```

Let's also update the VirtualService to update the host name:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: helloworld
spec:
  hosts:
    - hello.tetratelabs.dev
  gateways:
    - public-gateway
  http:
    - route:
      - destination:
          host: hello-world.default.svc.cluster.local
          port:
            number: 80
```

To try it out we will use cURL and set the host header to `hello.tetratelabs.dev` then tell curl to resolve the `hello.tetratelabs.dev` to the $INGRESS_IP address. Additionaly, we are also presenting the certificate authority cert we created:

```sh
curl -H "Host:hello.tetratelabs.dev" --resolve "hello.tetratelabs.dev:443:$INGRESS_IP" --cacert tetratelabs.dev.crt "https://hello.tetratelabs.dev:443"
```

Notice we get back the response - just like we did before. We can re-run the command with `-v` for verbose output and you'll notice that the certificate was used.

Next, let's see how we can use a service called SSL For Free to create a real certificate for a real domain.

## Using "SSL for Free"

Note that for this you'll need an actual real domain name. For this demo I registered a domain called `istioweekly.com`.

I have already created the certificate bundle on SSL For Free website.

You can create your own certificate by clicking "New Certificate" button, provide the domain name and then verify the domain. There are multiple ways to do that, you can get a code sent to an email address from that domain, you can create a `CNAME` entry in the DNS records or you can upload a file to a specific location on your domain.

Either way you go, once the domain is verified you'll be able to download a zip file with your certificate, certficiate key and the CA bundle.

Once you extracted the certificate bundle file, we need to create a new Kuberentes secret to hold the private key and the certificate. Then we will have to update the gateway and the virtual service as well.

Let's start with the secret: 

```sh
kubectl create -n istio-system secret tls istioweekly-credential --key=private.key --cert=certificate.crt
```

Let's update the Gateway with the secret name and the host:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: public-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 443
        name: https
        protocol: HTTPS
      tls:
        mode: SIMPLE
        credentialName: istioweekly-credential
      hosts:
        - istioweekly.com
```

And also update the VirtualService:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: helloworld
spec:
  hosts:
    - istioweekly.com
  gateways:
    - public-gateway
  http:
    - route:
      - destination:
          host: hello-world.default.svc.cluster.local
          port:
            number: 80
```

You can now navigate to the domain to see the "Hello World" response as well as the padlock that shows the connection is secure.


In order to hook-up your domain name with the ingress IP, you have to update the DNS records on the domain. Specifically, create an A record and point it to the external IP address of the ingress. That way, when you type your domain name (e.g. `istioweekly.com`), it resolves to the ingress gateways external IP address.

This works great, however, it's a bit awkward updating the certificates manually before they expire. In the next demo we will use a tool called [cert-manager](https://cert-manager.io/docs/) to automate this.

## Using cert-manager

Let's start by installing the cert-manager first:

```sh
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.3.1/cert-manager.yaml
```

There are two challenges you can use to verify the ownership of the domain - in this demo we will use the HTTP challenge

Once the cert-manager is installed (make sure all Pods in the `cert-manager` namespace are running), we can create the Issuer resource.

Make sure you add your email address in the `email` field:


```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-prod
  namespace: istio-system
spec:
  acme:
    # For staging: https://acme-staging-v02.api.letsencrypt.org/directory
    server: https://acme-v02.api.letsencrypt.org/directory 
    email: [youremail@yourdomain.com]
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - selector: {}
      http01:
        ingress:
          class: istio
```
In the `server` field have an option of using a staging or a production Let's Encrypt server. If you're testing things out and you don't want to get rate limited, switch to the staging URL. For production you should always use the production server address.

Then, in the `solvers` section we specify that we want to use HTTP challenge. Additionally, we specify the `class: istio` annotation. This annotation will later get added to the Ingress resource that gets created by the cert-manager. 

Next we can create the certificate and get it issued:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: hello-istioweekly-com
  namespace: istio-system
spec:
  secretName: hello-istioweekly-com-tls
  issuerRef:
    name: letsencrypt-prod
  commonName: hello.istioweekly.com
  dnsNames:
  - hello.istioweekly.com
```

When we create the `Certificate` a certificate request workflow kicks off - this involves creating a `CertificateRequest` resource, a Pod that runs a web server that's able to answer the challenge and a Kubernetes Ingress resource that points to that workload.

Once the verification is passed, the certificate will get issued. You can check that the certificate was issued by running `kubectl get certificate -A`.

Last thing we need to do is to update the Gateway resource and reference the secret name with the certificate:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: public-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 443
        name: https
        protocol: HTTPS
      tls:
        mode: SIMPLE
        credentialName: hello-istioweekly-com-tls
      hosts:
        - 'hello.istioweekly.com'
```

Also update the VirtualService with the different host:


```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: helloworld
spec:
  hosts:
    - hello.istioweekly.com
  gateways:
    - public-gateway
  http:
    - route:
      - destination:
          host: hello-world.default.svc.cluster.local
          port:
            number: 80
```

>Note: if you use staging issuer, make sure you add `-k` when running curl. Otherwise you'll get an error. Similarly, you'll get an error (untrusted cert) if you use the browser to navigate to the page presenting the staging certificate.