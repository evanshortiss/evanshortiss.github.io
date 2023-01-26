---
published: true
title: Quarkus and Testcontainers on macOS
layout: post
categories: containers
---

Recently, I wanted to use Podman for running Testcontainers with a Quarkus application. I followed this helpful [post by Stephen Nimmo](https://stephennimmo.com/using-podman-with-quarkus-and-testcontainers-on-macos/), but my containers were not being run in the Podman VM. It turns out that running both Podman and Docker on the same machine can make things less than straightforward!

If you find yourself in the same situation it's probably because you started the Docker VM before the Podman VM (via `podman machine start` or [Podman Desktop](https://podman-desktop.io/)). This means that Docker is still running and listening at `/var/run/docker.sock` and therefore Testcontainers is talking to it instead of Podman. 

To get your containers running in the Podman VM you can stop the Docker VM by exiting Docker Desktop using [this helpful snippet from Stack Overflow](https://stackoverflow.com/a/68549332) and restarting the Podman machine:

```bash
# Stop the running Podman machine VM
podman machine stop

# Kill the running Docker VM and Docker Desktop
ps ax|grep -i docker|egrep -iv 'grep|com.docker.vmnetd'|awk '{print $1}'|xargs kill

# Restart the Podman machine VM
podman machine start
```

Once the `podman machine start` command has completed it should report that it's listening on */var/run/docker.sock*, since Docker no longer has a hold on it.

You can now start your Quarkus application that uses Testcontainers, and they'll be run using Podman!

*NOTE: I tried setting the `DOCKER_HOST` and `TESTCONTAINERS_DOCKER_SOCKET_OVERRIDE` variables to force Quarkus and Testcontainers to bypass the Docker socket and use my Podman socket at `/Users/$USER/.local/share/containers/podman/machine/podman-machine-default/podman.sock` but this always reported errors about missing containers.*

