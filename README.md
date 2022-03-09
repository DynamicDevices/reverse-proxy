# Overview

This is a BalenaHub 'Block' project to enable flexible management of on-device networking using [Traefik](https://traefik.io/)

# Block Configuration

This is an initial example which will configure a node-exporter service on an internal port :9100 which is then routed by Traefik to be on a path of the public URL on :80

Add the following to a `docker-compose.yml` for your fleet.

**NOTE:** I couldn't get this working for the raspberrypi4-64 which is what I am testing here. It appears to be related to an issue with Balena `--platform` parsing and I had to implement the fix shown [here](https://github.com/balena-io/balena-cli/issues/1408).

```
version: '2'

services:
  node-exporter:
    # Use a specific hash of the node-exporter (which you can find on Docker Hub) as we don't currently pick up architectures correctly
    image: dynamicdevices/balenablock-node-exporter@sha256:9c03ad3c3c7b6201c0f5a644d5d7023e8ecb42767c55945082807e16e4eb60de
    # Restart the container if there's a problem
    restart: always
    # No needfor enhanced privileges
    privileged: false
    ports:
    # The standard node-exporter port is 9100
      - '9100:9100'
    labels:
      # Enable traefik support for this container
      - "traefik.enable=true"
      # Make it available on the "web" entrypoint which is defined below as :80
      - "traefik.http.routers.node-exporter.entrypoints=web"
      # Add a specific path prefix that will be expected on the URL suffix
      - "traefik.http.routers.node-exporter.rule=PathPrefix(`/metrics`)"
  traefik:
    image: dynamicdevices/balenablock-traefik@sha256:072770347a92b1828efca812756051c1652333247fec1e56f7de3077c0b59e7e
    container_name: traefik
    command:
      - "--log.level=DEBUG"
      - "--api.dashboard=true"
      # Uncomment this to enable the Traefik WebUI on port :8080
      #- "--api.insecure"
      - "--providers.docker=true"
      - "--providers.docker.endpoint=unix:///var/run/balena-engine.sock"
      #- "--providers.docker.exposedbydefault=true"
      - "--entrypoints.web.address=:80"
    restart: always
    ports:
      # The HTTP port
      - "80:80"
      # The Dashboard port
      - "8080:8080"
    labels:
      # So that Traefik can listen to the Docker events
      - "io.balena.features.balena-socket=1"
```

With this Fleet configuration deployed you should be able browse to your public endpoint (as seen on the device dashboard)

If  you browse to the base URL you will see an error but if you browse to https://my-balena-device-id.balena-devices.com/metrics your requested will be proxied by Traefik to the node-exporter webserver running on the internal port :9100 on the device

You'll then see a list of key value pairs of device metrics)

The other thing you can do here is enable insecure mode for the Traefik WebUI on port :8080 to see how it is configured.

You can then tunnel with the Balena CLI as follows:

`balena tunnel your-fleet-name 8080:8080`

Use a browser to go to [localhost:8080](http://localhost:8080) and you'll see the Traefik WebUI

# Rebuilding the Docker image and pushing to your own Docker registry

There is a [build-images.sh](https://github.com/DynamicDevices/reverse-proxy/blob/main/build-images.sh) script which I modified from the `pulse` block.

You'll need to change the Docker repo to your own and you should then be able to use this to build and upload the image(s).

I had problems as my installation of `docker buildx` didn't have support for ARM64 building and I had to look through the information [here](https://community.arm.com/arm-community-blogs/b/tools-software-ides-blog/posts/getting-started-with-docker-for-arm-on-linux)

Specifically I needed to register Arm executables to run on x64 machines:

```
docker run --rm --privileged docker/binfmt:820fdd95a9972a5308930a2bdfb8573dd4447ad3
```

To verify the qemu handlers are registered properly run:

```
cat /proc/sys/fs/binfmt_misc/qemu-aarch64
```

