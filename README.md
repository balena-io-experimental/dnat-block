# dnat-block

Dynamically expose ports to services at runtime via environment variables.

Note that your app must be running in bridge networking mode (the default). Host networking does not require this block.

## Environment Variables

To run this project, you will need the following environment variables in your container:

- `FORWARD_SERVICE`: Name of another service in your compose file where the traffic will be forwarded.
- `FORWARD_PORT`: Which port to expose on your host to forward traffic to the defined service.
- `FORWARD_PROTOCOLS`: (optional) Either `tcp`, `udp`, or `tcp udp`. Default is both.
- `FORWARD_COMMENT`: (optional) Comment used when creating rules to ensure they are cleaned up.

See <https://www.balena.io/docs/learn/manage/variables/> for more info on setting device variables.

## Service Labels

To run this project, you will need the following labels on the `dnat` service:

- `io.balena.features.balena-socket=1`: Bind mounts the balena container engine socket into the container and
  sets the environment variable `DOCKER_HOST` with the socket location for use by docker clients.

See <https://www.balena.io/docs/reference/supervisor/docker-compose/#labels> for more info on supervisor labels.

## How It Works

1. The `dnat` service will resolve the bridge address of your forward service.
2. A temporary container be executed with host networking and elevated privileges in order to modify iptables on the host.
3. The `dnat` service will restart if the forward service changes it's bridge address.
4. Rules are cleaned up before/after running based on a comment used during creation.

## Usage/Examples

You can run multiple instances of this block to forward ports to multiple services,
but you should set your own `FORWARD_COMMENT` for one of them so they don't try to clean up each-others rules.

Add this block to your fleet application:

1. Clone or download this repo into a subdirectory of your project, eg. `./dnat`.
2. Using the [included docker-compose file](./docker-compose.yml) for reference and add this block to your project's `docker-compose.yml`.

```yml
services:
  # example only, your service goes here
  mywebapp:
    image: nginx:1.21.4-alpine

  # tell dnat to forward ports to your service
  dnat:
    build: dnat
    labels:
      io.balena.features.balena-socket: 1
    environment:
      FORWARD_SERVICE: mywebapp
      FORWARD_PORT: 80
      FORWARD_PROTOCOLS: tcp
    depends_on:
      - mywebapp
```

You can follow the service logs to see the iptables rules being created.

```plaintext
12.11.21 12:22:05 (-0500) -A POSTROUTING -s 172.18.0.2/32 -d 172.18.0.2/32 -p tcp -m tcp --dport 80 -m comment --comment my-custom-rules -j MASQUERADE
12.11.21 12:22:05 (-0500) -A DOCKER -p tcp -m tcp --dport 80 -m comment --comment my-custom-rules -j DNAT --to-destination 172.18.0.2:80
12.11.21 12:22:05 (-0500) -A DOCKER -d 172.18.0.2/32 -p tcp -m tcp --dport 80 -m comment --comment my-custom-rules -j ACCEPT
```
