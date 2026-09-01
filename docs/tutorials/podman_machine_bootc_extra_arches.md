![PODMAN logo](https://raw.githubusercontent.com/containers/common/main/logos/podman-logo-full-vert.png)

# Adding Extra Architectures to the Podman Machine bootc Image

By default, the Podman machine OS image supports `amd64` and `arm64` architectures
for running cross-architecture containers via QEMU emulation. If you need to run
containers for additional architectures (such as `s390x`, `ppc64le`, or `riscv64`),
you must build and apply a custom bootc-based machine OS image that includes the
`qemu-user-static` package.

This tutorial walks through building that custom image and applying it to a Podman
machine.

## Prerequisites

- Podman installed and working on your local system
- An account on a container registry (this tutorial uses `quay.io`)
- The version of the machine OS image you want to extend (check the
  [machine-os releases](https://quay.io/repository/podman/machine-os?tab=tags)
  to find the right tag)

## Step 1: Create the Containerfile

Create a `Containerfile` that installs `qemu-user-static` on top of the base machine
OS image. Replace `<version>` with the tag matching your current Podman machine OS
version.

```dockerfile
FROM quay.io/podman/machine-os:<version>
RUN dnf -y install qemu-user-static
```

## Step 2: Build the Custom Image

Build the image and tag it with your registry and username. Replace `<username>` and
`<version>` with your registry username and the same version tag used in the
Containerfile.

```console
podman build -f Containerfile -t quay.io/<username>/machine-os-custom:<version>
```

**NOTE**: This build runs on your host and produces a bootc-compatible OS image,
not a regular application container image.

## Step 3: Push the Image to a Registry

Push the newly built image to your registry so that `podman machine` can pull it
when applying the OS update.

```console
podman push quay.io/<username>/machine-os-custom:<version>
```

## Step 4: Initialize a Podman Machine

If you do not already have a Podman machine running, initialize and start one now.
If an existing machine is already running you can skip this step.

```console
podman machine init --now
```

## Step 5: Apply the Custom OS Image

Use `podman machine os apply` to replace the machine's OS with your custom image.
The `--restart` flag stops and restarts the machine automatically so the new OS
takes effect.

```console
podman machine os apply --restart docker://quay.io/<username>/machine-os-custom:<version>
```

Once the machine restarts, it will have `qemu-user-static` installed and can run
containers for the additional architectures provided by that package.

## Verification

To confirm the extra architectures are available, try pulling and running a
container for a non-native architecture:

```console
podman run --rm --platform linux/s390x docker.io/library/alpine uname -m
```

You should see `s390x` (or whichever architecture you chose) printed to the console.
