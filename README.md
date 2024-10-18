# mender-multiplatform

## Motivation

Integrating an application, framework or middleware - specifically mender in this repository - into a Yocto/OE build stack can involve some difficulties. For the sake of this document, this will be referred to as a "component", and it is assumed that a metadata layer for the component already is available. A possible appraoch is having an product layer that ties the underlying metadata together, which might be

  - the component layer
  - a distribution layer
  - one or more BSP layers

In this repository this will be:

  - component: [mender](https://github.com/mendersoftware/meta-mender)
  - distribution: poky (the canned Yocto Project reference distribution)
  - BSPs:
    * Beaglebone Black
    * Raspberry Pi 3
    * Raspberry Pi 4

While the concrete implementation is quite specific to the component-hardware combination - just like a product usually is - the general concepts can be seen in context here and applied to other setups.

## Concept

A common practise is to cram modifications into `local.conf`. However that clashes with the multiplatform objective of this repository, as well as being hard to maintain and manage as `local.conf` is (and should not) be part of source control, and therefore needs to be recreated over and over again.

The main structural concept being used are overrides for both `DISTRO` and `MACHINE` configurations to inject metadata in such a way that it is directly using the provided parts without needing to patch anything.

As BSPs are an integral part of the setup, it is mandatory for them to be well behaved, which essentially boils down to one thing:

**The BSP shall have no effect unless a `MACHINE` configuration it provides is selected for the build**

This ensures we can have an arbitrary number of BSPs added to the layer stack, therefore facilitating building for multiple hardware variants.

## Implementation

We do provide overriding configurations for `DISTRO` and all desired `MACHINE`s. To make the context clear, all provided configurations emply the `mmp`-Prefix, which is short for `mender-multiplatform`.

### Override technique

An override configuration usually consists of three parts. Those are, in order:

1. adding the original identifier to the corresponding override variable
2. including the original configuration file
3. modifying/extending the desired metadata

### `DISTRO`

The distribution is essentially defining the API that your product intends to use. So this is the natural place to enable the component that we want to use across all hardware variants. Enabling the mender integration into a poky derived distribution therefore looks like this:

	DISTROOVERRIDES =. "poky:"

	require conf/distro/poky.conf

	INHERIT += "mender-full"
	INIT_MANAGER = "systemd"

This means, we want an almost unmodified `poky` distribution, just with `systemd` as init manager and the `mender-full` integration added across all machines/builds.

See also [mmp-poky](conf/distro/mmp-poky.conf)

### `MACHINE`

In many cases additions or modifications that affect the whole build for a specific target are needed. In this repository the [`mmp-beaglebone-yocto`](conf/machine/mmp-beaglebone-yocto.conf) `MACHINE` is essentially just an adapter to match naming and provide structure for further maintenance/extensions. The [mmp-raspberrypi3](conf/machine/mmp-raspberrypi3.conf) and [mmp-raspberrypi4](conf/machine/mmp-raspberrypi4.conf) `MACHINE` configurations on the other hand need similar extensions, so those are added through a common include file. Specifically, this works like:

`conf/machine/include/raspberrypi.conf`:

	RPI_USE_U_BOOT = "1"
	MENDER_BOOT_PART_SIZE_MB = "40"
	IMAGE_INSTALL_append = " kernel-image kernel-devicetree"
	IMAGE_FSTYPES_remove += " rpi-sdimg"

	MENDER_FEATURES_ENABLE_append = " mender-uboot mender-image-sd"
	MENDER_FEATURES_DISABLE_append = " mender-grub mender-image-uefi"

	ENABLE_UART = "1"

`conf/machine/mmp-raspberrypi3.conf`:

	MACHINEOVERRIDES =. "raspberrypi3:"

	require conf/machine/raspberrypi3.conf
	require conf/machine/include/raspberrypi.conf

## Usage

***Note:***

**This repository is specifically showing a single build directory for multiple machines. Other use cases might need different setup strategies**

In order to build, you need a host that is capable of running a Yocto build as described [here](https://docs.yoctoproject.org/brief-yoctoprojectqs/index.html).

Alternatively, you can use a prepared docker container like [theyoctojester/devcontainer-local-yoep](https://hub.docker.com/r/theyoctojester/devcontainer-local-yoep).

### Setup with `kas`

For ease of use, the repository comes with `kas` configuration files. In order to use it, you need to have the `kas` utility installed, for example via

	sudo pip3 install kas

All you need to do is set up the build, and load the environment into the shell.

	kas checkout kas.yml
	source poky/oe-init-build-env

This creates the build setup, and intentionally sets the `MACHINE` variable to an invalid value to prevent accidentially building for an implicitly selected target.

### Manual setup

You need to clone the following repositories, with the given branches:

| URL | Branch |
| :-- | :----- |
| https://git.yoctoproject.org/git/poky | scarthgap |
| https://git.openembedded.org/meta-openembedded | scarthgap |
| https://github.com/mendersoftware/meta-mender.git | scarthgap |
| https://github.com/mendersoftware/meta-mender-community.git | scarthgap |
| https://github.com/agherzan/meta-raspberrypi.git | scarthgap |

On the example of `poky`, the clone procedure works like this:

	git clone https://git.yoctoproject.org/git/poky
	cd poky # the name of the repository
	git checkout dunfell	# the branch name
	cd ..

Repeat this for all listed repositories. The `master` branch does not need to be checked out specifically.

Once you have obtained all necessary metadata repositories, you can set up the build directory and add the specific layers. The following commands assume that you did all `git clone`s inside the top level directory of this repository. Otherwise, you might need to adjust the pathes accordingly.

	source poky/oe-init-build-env
	bitbake-layers add-layer ../meta-openembedded/meta-oe
	bitbake-layers add-layer ../meta-raspberrypi
	bitbake-layers add-layer ../meta-mender/meta-mender-core
	bitbake-layers add-layer ../meta-mender/meta-mender-demo
	bitbake-layers add-layer ../meta-mender/meta-mender-raspberrypi
	bitbake-layers add-layer ../meta-mender-community/meta-mender-beaglebone

You might want to revisit the default generated settings in `conf/local.conf`, although the automatically generated state should usually work.

### Building

Now you are ready to build  in the same context for all supported targets in any desired combination and order, there should be no conflicts.

	MACHINE=mmp-beaglebone-yocto bitbake core-image-minimal
	MACHINE=mmp-raspberrypi3 bitbake core-image-minimal
	MACHINE=mmp-raspberrypi4 bitbake core-image-minimal

In order to leverage caches for downloads and build artifacts, you can pass them in via environment variables like

	DL_DIR=/path/to/downloads SSTATE_DIR=/path/to/sstate MACHINE=mmp-raspberrypi4 bitbake core-image-minimal
