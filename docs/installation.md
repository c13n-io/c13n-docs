# Installing c13n manually

## Intro

We are going to setup **c13n** and connect it to an LND. When that is completed, we will place an **Envoy reverse proxy** in front of the c13n RPC service. Envoy is needed in order to enable web applications (like Arc) to normally communicate with the c13n RPC API.

## Prerequisites

- `lnd` instance that you can access
- Basic shell familiarity

## c13n

You can either install and run `c13n-go` on the same host as `lnd`, or install remotely on another machine.

> For remote access to `lnd` you need to include the `tlsextraip` option in your `lnd.conf` file for the ip to be included in the generated cert. For more info check `lnd`'s [sample-lnd.conf](https://github.com/lightningnetwork/lnd/blob/791411cdb85941001cb2af5f3ec5d47e97a5d19c/sample-lnd.conf#L42).


For more instructions on how to set up `c13n-go` and connect it to your `lnd`, please follow the detailed guide on its [Github repository](https://github.com/c13n-io/c13n-go#getting-started).

## Envoy

Due to browser limitations, in order to use c13n API (or any RPC API) using JavaScript, a reverse proxy should be deployed, translating `HTTP/1.1` calls to `HTTP/2`.

In this guide, steps for `gRPC-web` reverse proxying using [Envoy](envoyproxy.io/) are described.

### Installation

Envoy could be either installed as a native application or deployed using Docker. For native installation, please follow the up-to-date [official guide](https://www.envoyproxy.io/docs/envoy/latest/start/install).

For this guide, we are going to use the official envoy Docker image to deploy a gRPC-web proxy.

### Configuration

The reverse proxy is based on the `listener_grpc_web` envoy listener. Below, we provide an example envoy configuration file. 

Replace the following variables:

* `<PROXIED_RPC_HOST>`: Proxy host (used by gRPC-web clients)
* `<PROXIED_RPC_PORT>`: Proxy port (used by gRPC-web clients)
* `<C13N_RPC_API_HOST>`: c13n-go host
* `<C13N_RPC_API_PORT>`: c13n-go port
* `<PATH_TO_PEM_FILE>`: PEM encoded TLS certificate used by `c13n-go` (generated either externally or using `make certgen`)

```yaml
# envoy.yaml
# Example configuration for gRPC-web proxying

static_resources:
  listeners:
  - name: listener_grpc_web
    address:
      socket_address: { address: <PROXIED_RPC_HOST>, port_value: <PROXIED_RPC_PORT> }
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          codec_type: auto
          stat_prefix: ingress_http
          stream_idle_timeout: 0s
          route_config:
            name: local_route
            virtual_hosts:
            - name: local_service
              domains: ["*"]
              routes:
              - match:
                  prefix: "/"
                route:
                  prefix_rewrite: "/"
                  cluster: c13n_backend_grpc_web
                  max_grpc_timeout: 0s
              
              cors:
                allow_origin_string_match:
                  - safe_regex:
                      google_re2: {}
                      regex: .*
                allow_methods: GET, PUT, DELETE, POST, OPTIONS
                allow_headers: authorization,keep-alive,user-agent,cache-control,content-type,content-transfer-encoding,authorization,x-accept-content-transfer-encoding,x-accept-response-streaming,x-user-agent,x-grpc-web,grpc-timeout
                max_age: "1728000"
                expose_headers: authorization,grpc-status,grpc-message
          http_filters:
          - name: envoy.filters.http.cors
          - name: envoy.filters.http.grpc_web
          - name: envoy.filters.http.router

      transport_socket:
        name: envoy.transport_sockets.tls
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.DownstreamTlsContext

          common_tls_context:
            alpn_protocols: "h2"
            tls_certificates:
              - certificate_chain:
                  filename: /etc/envoy/envoy.crt
                private_key:
                  filename: /etc/envoy/envoy.key
            tls_params:
              tls_maximum_protocol_version: TLSv1_3
              tls_minimum_protocol_version: TLSv1_3

  clusters:
  - name: c13n_backend_grpc_web
    connect_timeout: 0.25s
    type: logical_dns
    http2_protocol_options: {}
    lb_policy: round_robin
    load_assignment:
      cluster_name: cluster_0
      endpoints:
        - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: <C13N_RPC_API_HOST>
                    port_value: <C13N_RPC_API_PORT>
    transport_socket:
      name: envoy.transport_sockets.tls
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
        common_tls_context:
          validation_context:
            trusted_ca:
              filename: <PATH_TO_TLS_CERT>

```

### Using SSL

In order to setup SSL with Envoy, a certificate is needed. This can be a preexisting one or a self signed. For the latter, `openssl` can be used:

```bash
export ENVOY_HOST=[Host that envoy is being set up]
openssl \
       req -x509 \
       -newkey rsa:4096 \
       -keyout envoy.key \
       -out envoy.crt \
       -nodes -batch \
       -days 365 \
       -addext "subjectAltName=IP:$ENVOY_HOST"
```

### Usage
Using the configuration file created in the last step (`envoy.yaml`), execute the following command:
    
```bash
docker run \
    -v "$(pwd)/envoy.yaml:/etc/envoy/envoy.yaml" \
    -v "$(pwd)/envoy.crt:/etc/envoy/envoy.crt" \
    -v "$(pwd)/envoy.key:/etc/envoy/envoy.key" \
    -p <PROXIED_RPC_PORT>:<PROXIED_RPC_PORT>
    envoyproxy/envoy:v1.20.0
```

c13n is now proxied over Envoy and ready to be consumed by gRPC clients. For a reference client implementation, consult [arc](https://github.com/c13n-io/arc), a messaging application based on c13n. You can also directly use arc at [c13n-io.github.io/arc](https://c13n-io.github.io/arc/).
