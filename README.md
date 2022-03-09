# Overview

This is a BalenaHub 'Block' project to enable flexible management of on-device networking using [Traefik](https://traefik.io/)

# Block Configuration

*TBD*

Add the following to a `docker-compose.yml` for your fleet. At this time we expose the `node-exporter` on port 80 i.e. the public URL.

**NOTE:** I couldn't get this working for the raspberrypi4-64 which is what I am testing here. It appears to be related to an issue with Balena `--platform` parsing and I had to implement the fix shown [here](https://github.com/balena-io/balena-cli/issues/1408). See below.

```
version: '2'

services:
  node_exporter:
    image: dynamicdevices/balenablock-node-exporter
    restart: always
    privileged: false
    ports:
      - '80:9100'
```

For the RPi-64 I had to pin the image to the current latest arm64 SHA. Note that this may be out of date in future. e.g.

```
version: '2'

services:
  node_exporter:
    image: dynamicdevices/balenablock-node-exporter@sha256:9c03ad3c3c7b6201c0f5a644d5d7023e8ecb42767c55945082807e16e4eb60de
    restart: always
    privileged: false
    ports:
      - '80:9100'
```

With this Fleet configuration deployed you should be able browse to your public endpoint (:80 on the balena device dashboard) and see a list of key value pairs of device metrics)

e.g. `https://your-device-id.balena-devices.com/metrics`

![image](https://user-images.githubusercontent.com/1537834/156930694-47536493-83a1-4fb8-b167-a2e5c36ddd9d.png)

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

