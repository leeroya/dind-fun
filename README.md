# Developing with a container in a container

The intension for this repository is to try work through the ability to run containers in a container but in a safe way.
The problems that we are trying to solve is:

1. As a developer I need and environment that best represents an production type environment.
This is not to be confused with the required SDK's and other tools that is required while developing, but more around the production behavior i.e. security etc.

2. As a developer I need and example of how to map the settings that I use in my development environment to the deployment of the application.

3. As a developer I need a easy cross platform IDE workflow to allow any team member to contribute to a project.

## Introduction

Containers are a great way to run applications in production, and have proven to the perfect way to build applications as all required SDK's and tools can be included in a container image.
There is a lot of examples where running containers as development environments.
If [Docker](https://www.docker.com/) is used, and the intention is to use a container as an environment then the term DinD (Docker in Docker) or DooD (Docker on Docker/Docker For Docker) will come up if it has not already.

All the examples and experiments will make use of [VSCode](https://code.visualstudio.com/) as it solves the last requirement with little to no effort as it has a mature support community, making it easy to find solutions when problems arise. VSCode has a feature that allows the [orchestration of containers for development](https://code.visualstudio.com/docs/remote/containers) and will be used in the examples.

These solutions come with retrospectively similar problems but just presented in different ways, i.e. DinD the need to run ```--privileged``` and with DooD mounting the docker service (docker.sock) as a volume mount giving the container host level permissions.

### Podman

With this in mind, lets start looking at some alternatives.
[Podman](https://podman.io/) is a container management tool that does not need a daemon service running which allows for a little easier management in a container.
Having read [Redhat::Podman in a container blog post](https://www.redhat.com/sysadmin/podman-inside-container) and factored this into the overall goal, we will unpack how this could work.

As before there is a possibility that we can use the following argument when running the container.

```CMD
--privileged
```

This can be placed in the ```.devcontainer/devcontainer.json``` as a ```runArgs``` parameter.

```CMD
"runArgs": [ "--privileged" ],
```

This is an issue as the requirement is to run containers as you would in a production setting, and in production it is not advised to run a container in privileged mode.

If the command argument ```--privileged``` is not passed then the the ability of running a container becomes difficult as certain features have been disabled or have not loaded files from the host.
Having said that, what we can do is turn specific features on or provide specific files from the underlining host.

In the ```.devcontainer/devcontainer.json``` lets replace the current arguments with these:

```CMD
"runArgs": [ "--cap-add=sys_admin", "--security-opt", "seccomp=unconfined", "--device=/dev/fuse"],
```

Lets break down each feature that is being enabled.

#### cap_sys_admin

```CMD
"--cap-add=sys_admin"
```

CAP_SYS_ADMIN is required for the Podman running as root inside of the container to mount the required file systems.

#### Devices Fuse

```CMD
"--device=/dev/fuse"
```

The **--device /dev/fuse** flag must use ``fuse-overlayfs`` inside the container.
This option tells Podman on the host to add **/dev/fuse** to the container so that containerized Podman can use it.

#### seccomp

```CMD
"--security-opt", "seccomp=unconfined",
```

Also need to disable seccomp, since Docker has a slightly stricter seccomp policy than Podman.
You could just use a Podman security policy by **using--seccomp=/usr/share/containers/seccomp.json**
