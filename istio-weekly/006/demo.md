# Envoy fundamentals demo

Let's start by running two Docker containers -- one is listening on port 5050 and the other one listening on port 5000.

```sh
docker run -dit --env BG_COLOR="blue" -p 5050:3000 gcr.io/tetratelabs/color-app:1.0.0

docker run -dit --env BG_COLOR="green" -p 5000:3000 gcr.io/tetratelabs/color-app:1.0.0
```

The two containers run simple Express web app that sets the background color -- the first one sets the background color to blue and the second one to green.

By the end of this demo we'll have an Envoy configuration that routes all traffic sent to `/blue` to the blue container and traffic sent to `/green` to the green container. We'll have a single listener and then based on the URI we'll route traffic to or the other container.

## Installing func-e CLI

Before we start writing the configuration, let's install a CLI call [func-e](https://func-e.io).

You can use this CLI to manage and run different Envoy versions:

```sh
curl -L https://func-e.io/install.sh | bash -s -- -b .
```

Let's start with a minimal configuration that doesn't do much: 

```yaml
static_resources:
  listeners:
  - name: listener_0
    address:
      socket_address:
        address: 0.0.0.0
        port_value: 10000
    filter_chains: [{}]
```

Save the above YAML to `config.yaml`. If we run Envoy with this configuration, the proxy will start and it will listen on port 10000, however, we haven't defined any filter chains, routes or clusters, so it's not going to know what to do with the request:

```sh
func-e run -c config.yaml
```

If we send the request, Envoy will just close the connection.

Let's update this config and add the HTTP filter and the router filter to the HTTP filter chain, but without any configuration:

```yaml
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: edge
          http_filters:
          - name: envoy.filters.http.router
          route_config: {}
```

We are adding a single filter - the HTTP connection manager and then inside that filter we are adding the router filter - remember, that's the last filter in the chain that does the routing. Note that this config doesn't have any routes defined, so if we run this - it works, but we'll get back a 404.

Let's create a very simple route config that just returns a direct response, just so we see how the configuration looks like:

```yaml
          route_config:
            virtual_hosts:
            - name: direct_response_service
              domains: ["*"]
              routes:
              - match:
                  prefix: "/"
                direct_response:
                  status: 200
                  body:
                    inline_string: "Hello"
```

With this copnfiguration we're defining a single virtual host that matches any domains. Once it matches the domain, it will try to match the prefix. Once the prefix is matches as well it returns a direct response.

Let's run Envoy with this configuration and send a request to `localhost:10000`:

```
$ func-e run -c config.yaml
...

$ curl localhost:10000
```

The response should be an HTTP 200 and `Hello` - you'll also notice the server response header is set to Envoy (`server: envoy`).

Sending a direct response is not too useful, let's change that and instead of a `direct_response` use a route that will select a cluster.

```yaml
          route_config:
            virtual_hosts:
            - name: all_domains
              domains: ["*"]
              routes:
              - match:
                  prefix: "/blue"
                route:
                  cluster: blue
              - match:
                  prefix: "/green"
                route:
                  cluster: green
```

With this change we are creating a single virtual host that matches all domains and then within the routes we are checking for the prefix match - first match checks for `/blue` and within the `route` we're specifying the cluster name - similarly for the `/green` path and the `green` cluster.

We also need to define the two clusters:

```yaml
  clusters:
  - name: blue
    connect_timeout: 5s
    load_assignment:
      cluster_name: blue
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: 127.0.0.1
                port_value: 5050
  - name: green
    connect_timeout: 5s
    load_assignment:
      cluster_name: green
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: 127.0.0.1
                port_value: 5000
```

This is where we specify the endpoints - the actual IP addresses and ports.

Let's re-run the Envoy (`func-e run -c config.yaml`) - now if you open `localhost:10000/blue` you'll get the Blue container and if you open `localhost:10000/green` you'll get the Green container.

## Enabling admin interface, metrics, and access logging

Let's see how to get the metrics and enable access logging.

We'll start with the access logger first - we'll define it at the HTTP connection manager filter, and we'll just write the logs to standard out. The other options for logging are to log to a file or provide an Access log service that Envoy can send the logs to through grpc.

Add the following snipper right under the `stat_prefix` field:
```yaml
          access_log:
          - name: envoy.access_loggers.stdout
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.access_loggers.stream.v3.StdoutAccessLog
```

If we make a couple of requests, you'll notice Envoy is writting the access logs to standard out:

```sh

```

Next, let's see how we could get some statistics and figure out how many successful requests were sent to different clusters.

Envoy does expose a statistics endpoint, but we need to enable the admin interface in order to access it.

```yaml
admin:
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 9901
```

If we restart Envoy, we can now send a request to `/green` or `/blue` and then observe how the metrics change:

```
curl localhost:9901/stats
```

Envoy also emits the Prometheus style metrics under /stats/prometheus.
