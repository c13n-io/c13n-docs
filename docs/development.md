# c13n Development

## Testbench
While you can always deploy a fully fledged lightning daemon and a c13n node, it is always useful to set up a testbench during development. In this guide, we will deploy a local testbench with a Neutrino backed lnd on testnet, a c13n node and an example application consuming the c13n API.

The testbench is using the following containerized elements via `Docker` and `docker-compose`:

* `lnd`: Lightning node using Neutrino as a BTC backend in Testnet
* `envoy`: Proxy to provide `gRPC-web` support for the `c13n` RPC server
* `c13n`: The main stack element
* `arc`: An instant messaging web interface leveraging `c13n` functionality

During development, some of the testbench elements may not be required. For this reason, the docker-compose definition is using profiles. Each stack element should be explicitly declared when starting the stack using `--profile`.

### Requirements
* Docker
* docker-compose `>=1.29.2`
* openssl

### Setup

1. Download the deployment stack definition
```shell
git clone https://github.com/c13n-io/c13n-dev-stack.git
cd c13n-dev-stack
```

2. Generate TLS certificates for c13n
```shell
cd c13n/certs
openssl req -nodes -x509 -newkey ec -pkeyopt ec_paramgen_curve:prime256v1 -config cert.conf -extensions v3_exts -days 365 -keyout c13n.key -out c13n.pem
```

3. Generate TLS certificates for envoy
```shell
cd envoy/certs
openssl req -x509 -newkey rsa:4096 -keyout envoy.key -out envoy.crt -nodes -batch -days 365 -addext 'subjectAltName=IP:0.0.0.0'
```
Before proceeding to the next step, ensure that both `envoy.crt` and `envoy.key` are readable from inside the container (usually a `chmod o+r envoy.*` is enough).

4. Generate encryption key for c13n database
> **This should be avoided in production. Encryption keys should not be stored in the same host with the encrypted database**
```shell
cd c13n/conf
tr -dc 'a-zA-Z0-9' < /dev/urandom | dd bs=1 count=32 of=store.key
```

5. Generate RPC server password hash
    * Generate the hash using `c13n genpwdhash`
    ```shell
    docker run -it ghcr.io/c13n-io/c13n-go:latest genpwdhash
    ```
    * Set the c13n RPC server password hash to the generated value in `c13n/conf/config.yml:server.pwdhash`

6. Configure stack elements
Configuration for each stack element is located in the following paths:
    * `lnd`: `lnd/conf/lnd.conf`
    * `c13n`: `c13n/conf/config.yml`
    * `envoy`: `envoy/conf/envoy.yml`

7. Start the stack
The stack is now ready to be initialized.
```shell
docker-compose --profile arc --profile c13n-go --profile lnd --profile envoy up
```

8. Create the lnd wallet
Even though the stack is initialized, a lightning wallet should first be created before c13n can connect with the lnd node.
```shell
docker-compose exec lnd bash
# lncli --network testnet create
```

9. Unlock lnd wallet
```shell
docker-compose exec lnd bash
# lncli --network testnet unlock
```

10. Access stack elements
    * `c13n` RPC service: `0.0.0.0:9999`
    * `arc`: `0.0.0.0:443/c13n`
    * `lnd`:
        * RPC: `0.0.0.0:10009`
        * REST: `0.0.0.0:8083`

With this, the c13n development node is up and running, exposing the c13n RPC API.

## Custom testbench elements
Testbench elements are pluggable and can be replaced with custom implementations during development and testing. The testbench setup is based on `docker-compose`, making the usage of custom elements as easy as replacing the `image:` tag value for each particular image. For example, the following steps enable the usage of a custom `c13n-go` testbench image:

1. Build a custom `c13n-go` image with the desired changes:
```bash
git clone https://github.com/c13n-io/c13n-go.git
cd c13n

# Modify c13n
# ...

docker build -t test_c13n_go -f docker/c13n/Dockerfile . 
```

    The last command builds a modified docker image, tagging it as `test_c13n_go` in the local Docker registry.

2. Modify the `docker-compose` definition, Replacing `image: ghcr.io/c13n-io/c13n-go:latest` with `image: test_c13n_go` under the c13n service definition.
3. Start the modified stack:
```bash
docker-compose --profile arc --profile c13n-go --profile lnd --profile envoy up
```

## RPC
When directly consuming the RPC API provided by c13n, there is no need for a reverse proxy deployment so the only profiles needed are the following:

```bash
docker-compose --profile lnd --profile c13n up
```

The certificates generated during setup for c13n can then be used to consume RPC with TLS.

## gRPC-web
By using the `--profile envoy`, we are placing an **Envoy reverse proxy** in front of the c13n RPC service. Envoy is needed in order to enable web applications (like [arc](https://github.com/c13n-io/arc/)) to normally communicate with the c13n RPC API.

Spawning the testbench with `gRPC-web` support is just a matter of executing docker compose as following:

```bash
docker-compose --profile envoy --profile c13n --profile lnd up
```

The `gRPC-web` root API path is `https://0.0.0.0/c13n-grpc/`.
