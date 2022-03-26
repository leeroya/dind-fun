# Container in a Container

The intension for this repository is to try work through the ability to run containers in a container but in a safe way.

## Podman

[Redhat::Podman in a container blog post](https://www.redhat.com/sysadmin/podman-inside-container).

using the folder podman

Normally you need to run:

```CMD
--privileged
```

This is an issue as the requirment is to run containers as you would in a production setting, and in production it is not avised to run a container in privileged mode.

These are the settings that are required to get this to work:

```CMD
"runArgs": [ "--cap-add=sys_admin", "--security-opt", "seccomp=unconfined", "--device=/dev/fuse"],
```

Lets break down each 