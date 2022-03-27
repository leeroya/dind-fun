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

```JSON
"runArgs": [ "--privileged" ],
```

This is an issue as the requirement is to run containers as you would in a production setting, and in production it is not advised to run a container in privileged mode.

If the command argument ```--privileged``` is not passed then the the ability of running a container becomes difficult as certain features have been disabled or have not loaded files from the host.
Having said that, what we can do is turn specific features on or provide specific files from the underlining host.

In the ```.devcontainer/devcontainer.json``` lets replace the current arguments with these:

```JSON
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

### Deployment

Now that we have this working locally, it is now needed to deploy it to a cluster. Some good documentaton to read is [Kubernetes security context setting](https://kubernetes.io/docs/tasks/configure-pod-container/;security-context/)

[Configuring Container Capabilities with Kubernetes](https://www.weave.works/blog/container-capabilities-kubernetes/)

```YAML
apiVersion: v1
kind: Pod
metadata:
 name: mypod
spec:
 containers:
   - name: myshell
     image: "ubuntu:14.04"
     command:
       - /bin/sleep
       - "300"
     securityContext:
       capabilities:
         add:
           - SYS_ADMIN
```

Here is an example of bind the userid at the same time:

```yaml
securityContext:
  privileged: false
  runAsUser: 1000
  runAsGroup: 1000
  capabilities:
    add:
      - SYS_ADMIN
```


Now having a look for a solution for the device fuse so that we have the possiblity for networking, as a developer will want to perform port binding when running a container.
based on [](https://github.com/kubernetes/kubernetes/issues/7890)

```YAML
apiVersion: v1
kind: ConfigMap
metadata:
  name: smarter-device-manager
  namespace: device-manager
data:
  conf.yaml: |
    - devicematch: ^fuse$
      nummaxdevices: 20
```

```YAML
# Pod spec: 
          resources:
            limits:
              smarter-devices/fuse: 1
              memory: 512Mi
            requests:
              smarter-devices/fuse: 1
              cpu: 10m
              memory: 50Mi
```

Or as a daemonset


```yaml
# https://gitlab.com/arm-research/smarter/smarter-device-manager/-/blob/master/smarter-device-manager-ds.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: smarter-device-manager
  namespace: device-manager
  labels:
    name: smarter-device-manager
    role: agent
spec:
  selector:
    matchLabels:
      name: smarter-device-manager
  updateStrategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        name: smarter-device-manager
      annotations:
        node.kubernetes.io/bootstrap-checkpoint: "true"
    spec:
      ## kubectl label node pike5 smarter-device-manager=enabled
      # nodeSelector:
      #   smarter-device-manager : enabled
      priorityClassName: "system-node-critical"
      hostname: smarter-device-management
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      containers:
      - name: smarter-device-manager
        image: registry.gitlab.com/arm-research/smarter/smarter-device-manager:v1.1.2
        imagePullPolicy: IfNotPresent
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop: ["ALL"]
        resources:
          limits:
            cpu: 100m
            memory: 15Mi
          requests:
            cpu: 10m
            memory: 15Mi
        volumeMounts:
          - name: device-plugin
            mountPath: /var/lib/kubelet/device-plugins
          - name: dev-dir
            mountPath: /dev
          - name: sys-dir
            mountPath: /sys
          - name: config
            mountPath: /root/config
      volumes:
        - name: device-plugin
          hostPath:
            path: /var/lib/kubelet/device-plugins
        - name: dev-dir
          hostPath:
            path: /dev
        - name: sys-dir
          hostPath:
            path: /sys
        - name: config
          configMap:
            name: smarter-device-manager
```

As for [seccomp](https://kubernetes.io/docs/tutorials/security/seccomp/) we should not need to do anything as it appears that it is disabled already.
