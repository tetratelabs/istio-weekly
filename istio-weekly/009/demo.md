# Customizing Istio metrics

The metrics can be customize by configuring the [stats extension](https://github.com/istio/proxy/tree/master/extensions/stats).

## Adding dimension to an existing metric

We'll use the Istio operator to update the `istio_requests_total` metric and add two dimensions to it:

- my_request_id
- gcp_location

The `my_request_id` will get its value from the `request.id` [Envoy attribute](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/advanced/attributes). The `gcp_location` value is coming from the bootstrap portion of the Envoy proxy configuration.

Deploy the following IstioOperator:

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: demo-install
  namespace: istio-system
spec:
  profile: demo
  meshConfig:
    defaultConfig:
      extraStatTags:
        - my_request_id
        - gcp_location
  values:
    telemetry:
      v2:
        prometheus:
          configOverride:
            inboundSidecar:
              metrics:
                - name: requests_total
                  dimensions:
                    my_request_id: request.id
                    gcp_location: node.metadata.PLATFORM_METADATA['gcp_location']
```

If you make a couple of requests to a workload running the mesh, you can check the `istio_requests_total` metric updated with the two dimensions we added. (Use `istioctl dash envoy [workload]` to open the Envoy dashboard)

## Creating a new metric

To create a new metric we'll define it in the `definitions` field. The value of the metric is a string, however, it needs to evaluate to an integer. Note the name of the metric (`my_custom_metric`) will have the stat prefix `istio` (this is hardcoded and can't be changed at the moment). That's why we need to use the full name (`istio_my_custom_metric`) in the `inclusionPrefixes` field.

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: demo-install
  namespace: istio-system
spec:
  profile: demo
  meshConfig:
    defaultConfig:
      proxyStatsMatcher:
        inclusionPrefixes:
          - istio_my_custom_metric
  values:
    telemetry:
      v2:
        prometheus:
          configOverride:
            inboundSidecar:
              definitions:
                - name: my_custom_metric
                  type: "COUNTER"
                  value: "(request.method.startsWith('POST') ? 1 : 0)"
```

### Creating new attributes

To create a new attribute we use the [AttributeGen extension](https://github.com/istio/proxy/tree/master/extensions/attributegen).

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: new-attribute
  namespace: istio-system
spec:
  configPatches:
  - applyTo: HTTP_FILTER
    match:
      context: SIDECAR_INBOUND
      proxy:
        proxyVersion: '1\.10.*'
      listener:
        filterChain:
          filter:
            name: "envoy.filters.network.http_connection_manager"
            subFilter:
              name: "istio.stats"
    patch:
      operation: INSERT_BEFORE
      value:
        name: istio.attributegen
        typed_config:
          "@type": type.googleapis.com/udpa.type.v1.TypedStruct
          type_url: type.googleapis.com/envoy.extensions.filters.http.wasm.v3.Wasm
          value:
            config:
              configuration:
                "@type": type.googleapis.com/google.protobuf.StringValue
                value: |
                  {
                    "attributes": [
                      {
                        "output_attribute": "istio_my_new_attribute",
                        "match": [
                          {
                            "value": "GetRequest",
                            "condition": "request.method == 'GET'"
                          },
                          {
                            "value": "PostRequest",
                            "condition": "request.method == 'POST'"
                          }
                        ]
                      }
                    ]
                  }
              vm_config:
                runtime: envoy.wasm.runtime.null
                code:
                  local: { inline_string: "envoy.wasm.attributegen" }
```

We can now add the new attribute as a dimension to the existing metric:


```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: demo-install
  namespace: istio-system
spec:
  profile: demo
  meshConfig:
    defaultConfig:
      extraStatTags:
        - my_method
  values:
    telemetry:
      v2:
        prometheus:
          configOverride:
            inboundSidecar:
              metrics:
                - name: requests_total
                  dimensions:
                    my_method: istio_my_new_attribute
```


## Resources

- Envoy attributes: https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/advanced/attributes
  - Attributes in source: https://github.com/istio/proxy/blob/master/src/istio/utils/attribute_names.cc
- Standard Istio metrics: https://istio.io/latest/docs/reference/config/metrics/
- Stats Wasm extension: https://github.com/istio/proxy/tree/master/extensions/stats 
- Stats configuration: https://istio.io/latest/docs/reference/config/proxy_extensions/stats/ 
- Attribute gen configuration: https://istio.io/latest/docs/reference/config/proxy_extensions/attributegen/ 