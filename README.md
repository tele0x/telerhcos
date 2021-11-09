# OpenShift with custom RHCOS build

RHCOS is a linux system, based on Red Hat Enterprise linux, optimized to run containers workload. At the core of RHCOS is OStree+rpm-ostree the hybrid package/image immutable filesystem managed like a GIT repository.
Currently RHCOS is built using internal Red Hat repositories, but you can build Fedora CoreOS using (https://github.com/coreos/fedora-coreos-config).

NOTE: Internal information are omitted from this document.

## Overview

In the following steps we will build a RHCOS image with `zabbix_agent` and `nvme-cli` package to control NVMe storage and also add an overlay to demonstrate how binaries/configuration and new files can be added to the RHCOS at built time. 
Will then create a new OpenShift `release_image` based on an existing release and use our custom RHCOS for the OS.

Steps for a build:

- A repository for the OS packages/config with the rpm-ostree manifests (for example:  https://github.com/openshift/os).
- Our custom manifest.yaml, zabbix repo and our overlay
- COSA tool: coreos-assembler (https://github.com/coreos/coreos-assembler).
- Fetch repository
- Build metal and metal4k to generate qcow2 image (for standalone VM usage and testing).
- Build OS Container and upload to public quay.io registry.
- Create temporary internal registry used for mirroring the images.
- Mirror OpenShift release to local registry replacing `machine-os-content` with our custom image.
- Mirror from local registry to public quay.io registyr.

## Prerequisities

Clone this repository:

```bash
$ git clone https://github.com/tele0x/telerhcos
```

Clone public repository

```bash
$ cosa init https:// https://github.com/openshift/os.git

# for a specific branch
$ cosa init --branch release-4.9 https://github.com/openshift/os.git
```

Clone the internal `redhat-coreos` repo

```bash
$ git clone [INTERNAL_REPO]

# For a specific branch
$ git clone --branch release 4.9 [INTERNAL_REPO]
```

Copy repo files from the internal repo into src/config

```
$ cp [INTERNAL_REPO_DIR]/*.repo src/config
```

Replace manifest.yaml and add zabbix custom repo

```
$ cp manifest.yaml src/config/manifest.yaml
$ cp zabbix.repo src/config/
# cp -fr 26-custombin/ src/config/overlay.d/
```

## COSA configuration

Apply `cosa` alias. Change COREOS_ASSEMBLER_CONTAINER variable based on the corresponding RHCOS version.
For example if building for RHCOS 4.8 use:

```
$ export COREOS_ASSEMBLER_CONTAINER=quay.io/coreos-assembler:rhcos-4.8
```

cosa() requires virtualization to run qemu, make sure your system has enabled virtualization on the BIOS and `kvm` module is loaded and `/dev/kvm` device exists.

```bash
cosa() {
    env | grep COREOS_ASSEMBLER
    set -x # so we can see what command gets run
    sudo podman run --rm -ti -v ${PWD}:/srv/ --userns=host --device /dev/kvm --privileged --name cosa \
               -v /etc/pki:/etc/pki \
               -v /etc/rhsm:/etc/rhsm \
               ${COREOS_ASSEMBLER_PRIVILEGED:+--privileged}                                          \
               ${COREOS_ASSEMBLER_CONFIG_GIT:+-v $COREOS_ASSEMBLER_CONFIG_GIT:/srv/src/config/:ro}   \
               ${COREOS_ASSEMBLER_GIT:+-v $COREOS_ASSEMBLER_GIT/src/:/usr/lib/coreos-assembler/:ro}  \
               ${COREOS_ASSEMBLER_CONTAINER_RUNTIME_ARGS}                                            \
               ${COREOS_ASSEMBLER_CONTAINER:-quay.io/coreos-assembler/coreos-assembler:latest} $@
    rc=$?; set +x; return $rc
}
```

Enter internal `cosa` shell and retrieve Red Hat CA to access internal repositories:

```bash
$ cosa shell
[coreos-assembler]$ sudo curl -kL -o /etc/pki/ca-trust/source/anchors/RH_IT_Root_CA.crt https://[INTERNAL_URL]/RH-IT-Root-CA.crt
[coreos-assembler]$ sudo update-ca-trust
```

## Build required images

```
$ cosa shell
[coreos-assembler]$ cosa fetch
[coreos-assembler]$ cosa build
[coreos-assembler]$ cosa build metal
[coreos-assembler]$ cosa build metal4k
[coreos-assembler]$ cosa buildextend-live
```
To build the Live ISO you must first build `metal` and `metal4k` images.
The `metal4k` build is required otherwise with just metal, squashfs uses LZ4 compression and if you boot the ISO will fail with below error:

```
Filesystem uses "lz4" compression. This is not supported"

Script commands doing the compression:

+ mv /tmp/osmet.bin /srv/tmp/tmp-osmet-pack/osmet.bin
Compressing squashfs with lz4
+ /usr/lib/coreos-assembler/gf-mksquashfs builds/410.84.202111041752-0/x86_64/rhcos-410.84.202111041752-0-metal.x86_64.raw /srv/tmp/buildpost-live/initrd-rootfs/root.squashfs lz4
+++ dirname /srv/tmp/buildpost-live/initrd-rootfs/root.squashfs
```
LZO is not enabled by default.
With both metal and metal4k the build script compress rootfs using gzip and the image boot successfully.


## Test the image

After the build is successful we can verify the image is working using `cosa run`

```
[coreos-assembler]$ cosa run
....
...
...
[QEMU guest is booting] [    6.168439] coreos-teardown-initramfs[1188]: info: skipping propagation of default networkin
[QEMU guest is booting] [    6.247832] audit: type=1404 audit(1636474292.029:2): enforcing=1 old_enforcing=0 auid=42949
[QEMU guest is booting] [    6.331859] audit: type=1403 audit(1636474292.113:3): auid=4294967295 ses=4294967295 lsm=sel
[QEMU guest is booting] [    6.359095] systemd[1]: systemd 239 (239-45.el8_4.3) running in system mode. (+PAM +AUDIT +S
[QEMU guest is booting] [    6.454814] systemd[1]: /usr/lib/systemd/system/rhcos-usrlocal-selinux-fixup.service:16: Ign
[QEMU guest is booting] [    6.460872] systemd[1]: /usr/lib/systemd/system/bootupd.service:22: Unknown lvalue 'ProtectH
[QEMU guest is booting] [    6.472562] systemd[1]: Unnecessary job for dev-virtio\\x2dports-mantlejournal.device was re
[QEMU guest is booting] [    6.477859] systemd[1]: systemd-journald.service: Service has no hold-off time (RestartSec=0
[QEMU guest is booting] [    6.478919] systemd[1]: systemd-journald.service: Scheduled restart job, restart counter is 
[QEMU guest is booting] [    6.500368] systemd[1]: Set up automount Arbitrary Executable File Formats File System Autom
[QEMU guest is booting] [    6.504120] systemd[1]: Starting Create list of required static device nodes for the current
[QEMU guest is booting] [    6.510175] systemd[1]: Starting Monitoring of LVM2 mirrors, snapshots etc. using dmeventd o
[EVENT | QEMU guest is ready for SSH] [ [0;32m  OK   [0m] Started Generate SSH keys snippet f…
via console-login-helper-messages.

Red Hat Enterprise Linux CoreOS 49.84.202111051605-0 [tele-rhcos]
  Part of OpenShift 4.9, RHCOS is a Kubernetes native operating system
  managed by the Machine Config Operator (`clusteroperator/machine-config`).

WARNING: Direct SSH access to machines is not recommended; instead,
make configuration changes via `machineconfig` objects:
  https://docs.openshift.com/container-platform/4.9/architecture/architecture-rhcos.html

---
[core@cosa-devsh ~]$ sudo -s
[root@cosa-devsh core]# zabbix_agentd 
```

Here we have our custom RHCOS with zabbix_agentd daemon, this is just for demonstration purpose but we could have created another overlay to add a systemd unit to run the daemon at boot.

Using `cosa list` we can list the current builds:

```
[coreos-assembler]$ cosa list
49.84.202111051605-0
   Timestamp: 2021-11-05T16:08:20Z (4 days, 1:28:47 ago)
   Artifacts: live-initramfs live-iso live-kernel live-rootfs metal metal4k ostree qemu
      Config: release-4.9 (9fcf0d061a25) (dirty)
```

This is showing us what artifacts are available, you can find the files under `builds/`

```
[coreos-assembler]$ ls -l builds/49.84.202111051605-0/x86_64/
total 10779432
-r--r--r--. 1 builder builder      84516 Nov  5 16:08 commitmeta.json
-r--r--r--. 1 builder builder        482 Nov  5 16:05 coreos-assembler-config-git.json
-r--r--r--. 1 builder builder     185193 Nov  5 16:05 coreos-assembler-config.tar.gz
-r--r--r--. 1 builder builder      33514 Nov  5 16:08 manifest-lock.generated.x86_64.json
-rw-r--r--. 1 builder builder       4619 Nov  5 18:49 meta.json
-r--r--r--. 1 builder builder      48028 Nov  5 16:08 ostree-commit-object
-rw-r--r--. 1 builder builder  242391040 Nov  5 18:45 rhcos-49.84.202111051605-0-extensions.x86_64.tar
-rw-r--r--. 1 builder builder   88090424 Nov  5 16:41 rhcos-49.84.202111051605-0-live-initramfs.x86_64.img
-rw-rw-r--. 1 builder builder   10034544 Nov  5 16:41 rhcos-49.84.202111051605-0-live-kernel-x86_64
-rw-rw-r--. 1 builder builder  927718912 Nov  5 16:41 rhcos-49.84.202111051605-0-live-rootfs.x86_64.img
-rw-rw-r--. 1 builder builder 1031798784 Nov  5 16:41 rhcos-49.84.202111051605-0-live.x86_64.iso
-r--r--r--. 1 builder builder 3978297344 Nov  5 16:09 rhcos-49.84.202111051605-0-metal.x86_64.raw
-r--r--r--. 1 builder builder 3978297344 Nov  5 16:35 rhcos-49.84.202111051605-0-metal4k.x86_64.raw
-r--r--r--. 1 builder builder  943063040 Nov  5 16:08 rhcos-49.84.202111051605-0-ostree.x86_64.tar
-r--r--r--. 1 builder builder 2583887872 Nov  5 16:08 rhcos-49.84.202111051605-0-qemu.x86_64.qcow2
```

In the output above we see additional files, ignore them for now.
Check [RHCOS_libvirt.md] to run our custom ISO with libvirt.

## Build container OS image

Next step is to create a containerized version of our RHCOS custom build. 
Will build and upload the container to quay.io, using quay we need to provide the authentication info. `cosa` looks for the auth on ~/.docker/config.json. From anther podman let's login into quay:

```
# podman login quay.io
```

One logged in the auth token is located `/run/users/0/containers/auth.json`, on the cosa shell create the `.docker` directory:

```
[coreos-assembler]$ mkdir /home/builder/.docker/
```

Create a file called `config.json` with the same content of `auth.json`
Run the command to create and upload the container:

```
[coreos-assembler]$ cosa upload-oscontainer --name quay.io/ferossi/telerhcos
```

Exit from the `cosa` container, type exit.

## Create local registry

podman run --privileged -d --name registry -p 5000:5000 -v /var/lib/registry:/var/lib/registry --restart=always registry:2

Add insecure registry to `/etc/containers/registries.conf` under registries.insecure section

```
[registries.insecure]
registries = ['localhost:5000']
```

Restart podman

```
systemctl restart podman
```

## Mirror OpenShift release to local registry

Mirror an OpenShift release to a local registry and replace the `machine-os-content` with our custom RHCOS image.
For the `machine-os-content` you must use a digest instead of the tag, that's becase MCO (Machine Config Operator) requires it.

```
export OPENSHIFT_RELEASE_IMAGE=quay.io/openshift-release-dev/ocp-release:4.9.4-x86_64
export CUSTOM_RHCOS_CONTAINER=quay.io/ferossi/telerhcos@sha256:dbd61e7beade03e3342d5dda8596736baf0a288ce6293af34c57503274123d97

oc adm release new -a auth.json --insecure=true --from-release=$OPENSHIFT_RELEASE_IMAGE machine-os-content=$CUSTOM_RHCOS_CONTAINER --mirror localhost:5000/telerelease --to-image localhost:5000/telerelease:latest
```

We can verify the release contains our image running:

```
# oc adm release extract --insecure=true --from=localhost:5000/telerelease:latest --file=image-references | grep -A 15 machine-os-content
        "name": "machine-os-content",
        "annotations": {
          "io.openshift.build.commit.id": "",
          "io.openshift.build.commit.ref": "",
          "io.openshift.build.source-location": "",
          "io.openshift.build.version-display-names": "machine-os=Red Hat Enterprise Linux CoreOS",
          "io.openshift.build.versions": "machine-os=49.84.202111051605-0",
          "io.openshift.release.override": "true"
        },
        "from": {
          "kind": "DockerImage",
          "name": "localhost:5000/telerelease@sha256:dbd61e7beade03e3342d5dda8596736baf0a288ce6293af34c57503274123d97"
        },
        "generation": null,
        "importPolicy": {},
        "referencePolicy": {
```

Compare the digest with the one from CUSTOM_RHCOS_CONTAINER, it should match.
With `oc adm release new` command you can basically replace any image on an OpenShift release. Simple replace `machine-os-content` with any name part of the `image-references` file.


## Mirror to public registry

Optionally you can use the internal registry to install OpenShift or push the image to a public registry like this:

To push the image to a public registry:

```
# oc adm release new -a /run/users/0/containers/auth.json --insecure=true --from-release=localhost:5000/telerelease:latest--mirror quay.io/ferossi/telerelease --to-image quay.io/ferossi/telerelease:4.9
```


## OpenShift install

At this point use either UPI/IPI or Assisted Installer method to install the OpenShift cluster.
To override the default release-image you can either:

```
# export OPENSHIFT_INSTALL_RELEASE_IMAGE_OVERRIDE=quay.io/ferossi/telerelease:latest
# openshift-install create  cluster
```

Or add `imageContentSources` to the install-config.yaml for example:

```yaml
...
imageContentSources:
- source: q.io/ocp/release-x.y
  mirrors:
  - local.registry.com/ocp/release-x.y
- source: q.io/openshift/x.y
  mirrors:
  - local.registry.com/ocp/release-x.y
```

For Assisted Installer, you must run AI on-prem or Infrastructure Operator on OCP to customize the release-image.


