# Yomi - Yet one more installer

# What is Yomi

Yomi (yet one more installer) is a new proposal for an installer for
the [open]SUSE family. It is designed as a
[SaltStack](https://www.saltstack.com/) state, and expected to be used
in situations were unattended installations for heterogeneous nodes is
required, and where some bits of intelligence in the configuration
file can help to customize the installation.

Being also a Salt state makes the installation process one more step
during the provisioning stage, making on Yomi a good candidate for
integration in any workflow were SaltStack is used.


# Overview

To execute Yomi we need a modern version of Salt, as we need special
features are only on the
[develop](https://github.com/saltstack/salt/tree/develop) branch of
Salt. Technically we can use the last released version of Salt for
salt-master, but for the minions we need the most up-to-date
version. The good news is that most of the patches are currently
merged in the openSUSE package of Salt.

Yomi is developed in
[OBS](https://build.opensuse.org/project/show/systemsmanagement:yomi),
and actually consists on two components:

* [yomi-formula](https://build.opensuse.org/package/show/systemsmanagement:yomi/yomi-formula):
  contains the Salt states and modules requires to drive an
  installation. The [source code](https://github.com/openSUSE/yomi) of
  the project in available under the openSUSE group in GitHub.
* [openSUSE-Tubleweed-Yomi](https://build.opensuse.org/package/show/systemsmanagement:yomi/openSUSE-Tumbleweed-Yomi):
  is the image that can be used too boot the new nodes, that includes
  the `salt-minion` service already configured. There are two versions
  of this image, one that is used as a LiveCD image and other designed
  to be used from a PXE Boot server.

The installation process of Yomi will require:

* Install and configure the
  [`salt-master`](#installing-and-configuring-salt-master) service.
* Install the [`yomi-formula`](#the-yomi-formula) package.
* Prepare the [pillar](#pillar-in-yomi) for the new installations.
* Boot the new systems with the [ISO image](#the-iso-image) or via
  [PXE boot](#pxe-boot)


# Installing and configuring salt-master

SaltStack can be deployed with different architectures. The
recommended one will require the `salt-master` service.

```bash
zypper in salt-master

systemctl enable --now salt-master.service
```

## Other ways to install salt-master

For different ways of installation, read the [official
documentation](https://docs.saltstack.com/en/latest/topics/installation/index.html). For
example, for development purposes installing it inside a virtual
environment can be a good idea:

```bash
python3 -mvenv venv

source venv/bin/activate

pip install --upgrade pip
pip install salt

# Create the basic layout and config files
mkdir -p venv/etc/salt/pki/{master,minion} \
      venv/etc/salt/autosign_grains \
      venv/var/cache/salt/master/file_lists/roots

cat <<EOF > venv/etc/salt/master
root_dir: $(pwd)/venv

file_roots:
  base:
    - $(pwd)/srv/salt

pillar_roots:
  base:
    - $(pwd)/srv/pillar
EOF
```

## Looking for the pillar

Salt pillar are the data that the Salt states use to decide the
actions that needs to be done. For example, in the case of Yomi the
typical data will be the layout of the hard disks, the software
patterns that will be installed, or the users that will be
created. For a complete explanation of the pillar required by Yomi,
check the section [Pillar in Yomi](#pillar-in-yomi)

By default Salt will search the states in `/srv/salt`, and the pillar
in `/srv/pillar`, as established by `file_roots` and `pillar_roots`
parameters in the default configuration file (`/etc/salt/master`).

To indicate a different place where to find the pillar, we can add a
new snippet in the `/etc/salt/master.d` directory:

```bash
cat <<EOF > /etc/salt/master.d/pillar.conf
pillar_roots:
  base:
    - /srv/pillar
	- /usr/share/yomi/pillar
EOF
```

The `yomi-formula` package already contains an example of such
configuration. Check section [Looking for the pillar in
Yomi](#looking-for-the-pillar-in-yomi)

## Enabling auto-sign

To simplify the discovery and key management of the minions, we can
use the auto-sign feature of Salt. To do that we need to add a new
file in `/etc/salt/master.d`.

```bash
echo "autosign_grains_dir: /etc/salt/autosign_grains" > \
     /etc/salt/master.d/autosign.conf
```

The Yomi ISO image available in Factory already export some UUIDs
generated for each minion, so we need to list into the master all the
possible valid UUIDs.

```bash
mkdir -p /etc/salt/autosign_grains

for i in $(seq 0 9); do
  echo $(uuidgen --md5 --namespace @dns --name http://opensuse.org/$i)
done > /etc/salt/autosign_grains/uuid
```

The `yomi-formula` package already contains an example of such
configuration. Check section [Enabling auto-sing in
Yomi](#enabling-auto-sing-in-yomi)

## Salt API

The `salt-master` service can be accessed via a REST API, provided by
an external tool that needs to be enabled.

```bash
zypper in salt-api

systemctl enable --now salt-api.service
```

There are different options to configure the `salt-api` service, but
is safe to choose `CherryPy` as a back-end to serve the requests of
Salt API.

We need to configure this service to listen to one port, for example
8000, and to associate an authorization mechanism. Read the Salt
documentation about this topic for different options.

```bash
cat <<EOF > /etc/salt/master.d/salt-api.conf
rest_cherrypy:
  port: 8000
  debug: no
  disable_ssl: yes
EOF

cat <<EOF > /etc/salt/master.d/eauth.conf
external_auth:
  file:
    ^filename: /etc/salt/user-list.txt
    salt:
      - .*
      - '@wheel'
      - '@runner'
      - '@jobs'
EOF

echo "salt:linux" > /etc/salt/user-list.txt
```

The `yomi-formula` package already contains an example of such
configuration. Check section [Salt API in Yomi](#salt-api-in-yomi)


# The Yomi formula

The states and modules required by Salt to drive an installation can
be installed where the `salt-master` resides:

```bash
zypper in yomi-formula
```

This package will install the states in
`/usr/share/salt-formulas/states`, some pillar examples in
`/usr/share/yomi/pillar` and configuration files in `/usr/share/yomi`.


## Looking for the pillar in Yomi

Yomi expect from the pillar to be a normal YAML document, optionally
generated by a Jinja template, as is usual in Salt.

The schema of the pillar is described in the section [Pillar reference
for Yomi](#pillar-reference-for-yomi), but the `yomi-formula` package
provides a set of examples that can be used to deploy MicroOS
installations, Kubic, LVM, RAID or simple openSUSE Tumbleweed ones.

In order to `salt-master` can find the pillar, we need to change the
`pillar_roots` entry in the configuration file, or use the one
provided by the package:

```bash
cp -a /usr/share/yomi/pillar.conf /etc/salt/master.d/
systemctl restart salt-master.service
```

## Enabling auto-sing in Yomi

The images generated by the Open Build Service that are ready to be
used together with Yomi contains a list a random UUID, that can be
used as a auto-sing grain in `salt-master`.

We can enable this feature adding the configuration file provided by
the package:

```bash
cp /usr/share/yomi/autosign.conf /etc/salt/master.d/
systemctl restart salt-master.service
```

## Salt API in Yomi

As described in the section [Salt API](#salt-api), we need to enable
the `salt-api` service in order to provide a REST API service to
`salt-minion`.

This service is used by Yomi to monitor the installation, reading the
event bus of Salt. To enable the real-time events we need to enable
set `events` field to `yes` in the configuration section of the
pillar.

We can enable this service easily (after installing the `salt-api`
package and the dependencies) using the provided configuration file:

```bash
cp /usr/share/yomi/salt-api.conf /etc/salt/master.d/
systemctl restart salt-master.service
```

Feel free to edit `/etc/salt/master.d/salt-api.conf` and provide the
required certificates to enable the SSL connection, an use a different
authorization mechanism. The current one is based on reading the file
`/usr/share/yomi/user-list.txt`, that is storing the password in plain
text. So please, *do not* use this in production.

### Real time monitoring in Yomi

Once we check that in our `config` of the pillar contains this:

```yaml
config:
  events: yes
```

We can launch the `monitor` tool.

```bash
export SALTAPI_URL=http://localhost:8000
export SALTAPI_EAUTH=file
export SALTAPI_USER=salt
export SALTAPI_PASS=linux

monitor -r -y
```

The `monitor` tool store in a local cache the authentication tokens
generated by Salt API. This will accelerate the next connection to the
service, but sometimes can cause authentication errors (for example,
when the cache is in place but the salt-master get reinstalled). The
option `-r` makes sure that this cache is removed before
connection. Check the help option of the tool for more information.


# Booting a new machine

As described in the previous sections, Yomi is a set of Salt states
that are used to drive the installation of a new operating system. To
take full control of the system where the installation will be done,
you will need to boot from an external system that provides an already
configured `salt-minion`, and a set of CLI tools required during the
installation.

We can deploy all the requirements using different mechanisms. One,
for example, is via PXE boot. We can build a server that will deliver
the Linux `kernel` and an `initrd` will all the required
software. Another alternative is to have an already live ISO image
that you use to boot from the USB port.

There is an already available image that contains all the requirements
in
[Factory](https://build.opensuse.org/package/show/openSUSE:Factory/openSUSE-Tumbleweed-Yomi). This
is an image build from openSUSE Tumbleweed repositories that includes
a very minimal set of tools, including the openSUSE version of
`salt-minion`.

To use the last version of the image, together with the last version
of `salt-minion` that includes all the patches that are under review
in the SaltStack project, you can always use the version from the
[devel
project](https://build.opensuse.org/package/show/systemsmanagement:yomi/openSUSE-Tumbleweed-Yomi)

Note that this image is a `_multibuild` one, and generates two
different images. One is a LiveCD ISO image, ready to be booted from
USB or DVD, and the other one is a PXE Boot ready image.

## The ISO image

The ISO image is a LiveCD that can be booted from USB or from DVD, and
the last version can be always be downloaded from:

```bash
wget https://download.opensuse.org/repositories/systemsmanagement:/yomi/images/iso/openSUSE-Tumbleweed-Yomi.x86_64-livecd.iso
```

## PXE Boot

The second image available is a OEM ramdisk that can be booted from
PXE Boot.

To install the image we first need to download the file
`openSUSE-Tumbleweed-Yomi.x86_64-${VERSION}-pxeboot-Build${RELEASE}.${BUILD}.install.tar`
from the Factory, or directly from the development project.

We need to start the `sftpd` service or use `dnsmasq` to behave also
as a tftp server. There is some documentation in the [openSUSE
wiki](https://en.opensuse.org/SDB:PXE_boot_installation), and if you
are using QEMU you can also check the appendix document.

```bash
mkdir -p /srv/tftpboot/pxelinux.cfg
cp /usr/share/syslinux/pxelinux.0 /srv/tftpboot

cd /srv/tftpboot
tar -xvf $IMAGE

cat <<EOF > /srv/tftpboot/pxelinux.cfg/default
default yomi
prompt   1
timeout  30

label yomi
  kernel pxeboot.kernel
  append initrd=pxeboot.initrd.xz rd.kiwi.install.pxe rd.kiwi.install.image=tftp://${SERVER}/openSUSE-Tumbleweed-Yomi.xz rd.kiwi.ramdisk ramdisk_size=1048576
EOF
```

## Finding the master node

The `salt-minion` configuration in the Yomi image will search the
`salt-master` system under the `master` name. Is expected that the
local DNS service will resolve the `master` name to the correct IP
address.

During boot time of the Yomi image we can change the address where is
expected to find the master node. To do that we can enter under the
GRUB menu the entry `master=my_master_address`. For example
`master=10.0.2.2` will make the minion to search the master in the
address `10.0.2.2`.

An internal systemd service in the image will detect this address and
configure the `salt-minion` accordingly.

Under the current Yomi states, this address will be copied under the
new installed system, together with the key delivered by the
`salt-master` service. This means that once the system is fully
installed with the new operating system, the new `salt-minion` will
find the master directly after the first boot.

## Setting the minion ID

In a similar way, during the boot process we can set the minion ID
that will be assigned to the `salt-minion`. Using the parameter
`minion_id`. For example, `minion_id=worker01` will set the minion ID
for this system as `worker01`.

The rules for the minion ID are a bit more complicated. Salt, by
default, set the minion ID equal to the FQDN or the IP of the node if
no ID is specified. This cannot be a good idea if the IP changes, so
the current rules are:

* The value from `minion_id` boot parameter.
* The FQDN hostname of the system, if is different from localhost.
* The MAC address of the first interface of the system.


# Basic operations

Once `salt-master` is configured and running, the `yomi-formula`
states are available and a new system is booted with a up-to-date
`salt-minion`, we can start to operate with Yomi.

The usual process is simple: describe the pillar information and apply
the `yomi.install` state to the node or nodes. Is not relevant how the
pillar was designed (maybe using a smart template that cover all the
cases or writing a raw YAML that only covers one single installation).
In this section we will provide some hints about how get information
and can help in this process.

## Getting hardware information

The provided pillar are only an example of what we can do with
Yomi. Eventually we need to adapt them based on the hardware that we
have.

We can discover the hardware configuration with different
mechanism. One is get the `grains` information directly from the
minion:

```bash
salt node grains.items
```

We can get more detailed information using other Salt modules, like
`partition.list`, `network.interfaces` or `udev.info`.

With Yomi we provided a simple interface to `hwinfo` that provides in
a single report some of the information that is required to make
decisions about the pillar.

```bash
# Synchronize all the modules to the minion
salt node saltutil.sync_all

# Get a short report about some devices
salt node devices.hwinfo

# Get a detailled report about some devices
salt node devices.hwinfo short=no
```

## Configuring the pillar

The package `yomi-formula` provides some pillar examples that can be
used as a reference when you are creating your own profiles.

Salt search the pillar information in the directories listed in the
`pillar_roots` configuration entry, and using the snippet from the
section [Pillar in Yomi](#pillar-in-yomi), we can make those examples
available in our system.

In the case that we want to edit those files, we can copy them in a
different directory and add it to the `pillar_roots` entry.

```bash
mkdir -p /srv/pillar-yomi
cp -a /usr/share/yomi/pillar/* /srv/pillar-yomi

cat <<EOF > /etc/salt/master.d/pillar.conf
pillar_roots:
  base:
    - /srv/pillar-yomi
    - /srv/pillar
EOF
systemctl restart salt-master.service
```

The pillar tree start with the `top.sls` file (there is another
`top.sls` file for the states, do not confuse them).

```yaml
base:
  '*':
    - installer
```

This file is used to map the node with the data that the states will
use later. For this example the file that contain the data is
`installer.sls`, but feel free to choose a different name when you are
creating your own pillar.

This `installer.sls` is used as an entry point for the rest of the
data. Inside the file there is some Jinja templates that can be edited
to define different kinds of installations. This feature is leveraged
by the
[openQA](https://github.com/os-autoinst/os-autoinst-distri-opensuse/tree/master/tests/yomi)
tests, to easily make multiple deployments.

You can edit the `{% set VAR=VAL %}` section to adjust it to your
current profile, or create one from scratch. The files
`_storage.sls.*` are included for different scenarios, and this is the
place where the disk layout is described. Feel free to include it
directly on your pillar, or use a different mechanism to decide the
layout.

## Cleaning the disks

Yomi try to be careful with the current data stored in the disks. By
default will not remove any partition, nor will make an implicit
decision about the device where the installation will run.

If we want to remove the data from the device, we can use the provided
`devices.wipe` execution module.

```bash
# List the partitions
salt node partition.list /dev/sda

# Make sure that the new modules are in the minion
salt node saltutil.sync_all

# Remove all the partitions and the filesystem information
salt node devices.wipe /dev/sda
```

To wipe all the devices defined in the pillar at once, we can apply
the `yomi.storage.wipe` state.

```bash
# Make sure that the new modules are in the minion
salt node saltutil.sync_all

# Remove all the partitions and the filesystem information
salt node state.apply yomi.storage.wipe
```

## Applying the yomi.install state

Finally, to install the operating system defined by the pillar into
the new node, we need to apply the high-state:

```bash
salt node state.apply yomi.installer
```

If we have a `top.sls` file similar to this example, living in
`/srv/salt` or in any other place where `file_roots` option is
configured:

```yaml
base:
  '*':
    - yomi.installer
```

We can apply directly the high state:

```bash
salt node state.highstate
```

# Pillar reference for Yomi

To install a new node, we need to provide some data to describe the
installation requirements, like the layout of the partitions, file
systems used, or what software to install inside the new
deployment. This data is collected in what is Salt is known as a
[pillar](https://docs.saltstack.com/en/latest/topics/tutorials/pillar.html).

To configure the `salt-master` service to find the pillar, check the
section [Looking for the pillar](#looking-for-the-pillar).

Pillar can be associated with certain nodes in our network, making of
this technique a basic one to map a description of how and what to
install into a node. This mapping is done via the `top.sls` file:

```yaml
base:
  'C7:7E:55:62:83:17':
    - installer
```

In `installer.sls` we will describe in detail the installation
parameters that will be applied to the node which minion-id match with
`C7:7E:55:62:83:17`. Note that in this example we are using the MAC
address of the first interface as a minion-id (check the section
**Enabling Autosign** for an example).

The `installer.sls` pillar consist on several sections, that we can
describe here.

## `config` section

The `config` section contains global configuration options that will
affect the installer.

* `events`: Boolean. Optional. Default: `yes`

  Yomi can fire Salt events before and after the execution of the
  internal states that Yomi use to drive the installation. Using the
  Salt API, WebSockets, or any other mechanism provided by Salt, we
  can listen the event bus and use this information to monitor the
  installer. Yomi provides a basic tool, `monitor`, that shows real
  time information about the installation process.

  To disable the events, set this parameter to `no`.

  Note that this option will add three new states for each single Yomi
  state. One extra state is executed always before the normal state,
  and is used to signalize that a new state will be executed. If the
  state is successfully terminated, a second extra state will send an
  event to signalize that the status of the state is positive. But if
  the state fails, a third state will send the fail signal. All those
  extra states will be showed in the final report of Salt.

* `reboot`: String. Optional. Default: `yes`

  Control the way that the node will reboot. There are three possible
  values:

  * `yes`: Will produce a full reboot cycle.

  * `no`: Will no reboot after the installation.

  * `kexec`: Instead of rebooting, reload the new kernel installed in
    the node.

* `snapper`: Boolean. Optional. Default: `no`

  In Btrfs configurations (and in LVM, but still not implemented) we
  can install the snapper tool, to do automatic snapshots before and
  after updates in the system. One installed, a first snapshot will be
  done and the GRUB entry to boot from snapshots will be added.

* `grub2_theme`: Boolean. Optional. Default: `no`

  If `yes` the `grub2-branding-openSUSE` package will be installed and
  configured.

* `grub2_console`: Boolean. Optional. Default: `no`

  If `yes` Yomi will append `console=tty0 console=ttyS0,115200` in the
  Linux command line during the boot. This option is help full when
  the we want to have serial access to console to the new machine.

* `locale`: String. Optional. Default: `en_US.utf8`

  Sets the system locale, more specifically the LANG= and LC\_MESSAGES
  settings. The argument should be a valid locale identifier, such as
  `de_DE.UTF-8`. This controls the locale.conf configuration file.

* `locale_message`: String. Optional.

  Sets the system locale, more specifically the LANG= and LC\_MESSAGES
  settings. The argument should be a valid locale identifier, such as
  `de_DE.UTF-8`. This controls the locale.conf configuration file.

* `keymap`: String. Optional. Default: `us`

  Sets the system keyboard layout. The argument should be a valid
  keyboard map, such as `de-latin1`. This controls the "KEYMAP" entry
  in the vconsole.conf configuration file.

* `timezone`: String. Optional. Default: `UTC`

  Sets the system time zone. The argument should be a valid time zone
  identifier, such as "Europe/Berlin". This controls the localtime
  symlink.

* `hostname`: String. Optional.

  Sets the system hostname. The argument should be a host name,
  compatible with DNS. This controls the hostname configuration file.

* `machine_id`: String. Optional.

  Sets the system's machine ID. This controls the machine-id file.

* `target`: String. Optional. Default: `multi-user.target`

  Set the default target used for the boot process.

Example:

```yaml
config:
  # Do not send events, useful for debugging
  events: no
  # Do not reboot after installation
  reboot: no
  # Always install snapper if possible
  snapper: yes
  # Set language to English / US
  locale: en_US.UTF-8
  # Japanese keyboard
  keymap: jp
  # Universal Timezone
  timezone: UTC
  # Boot in graphical mode
  target: graphical.target
```

## `partitions` section

Yomi separate partitioning the devices from providing a file system,
creating volumes or building arrays of disks. The advantage of this is
that this, usually, compose better that other approaches, and makes
more easy adding more options that needs to work correctly with the
rest of the system.

* `config`: Dictionary. Optional.

  Subsection that store some configuration options related with the
  partitioner.

  * `label`: String. Optional. Default: `msdos`

    Default label for the partitions of the devices. We use any
    `parted` partition recognized by `mklabel`, like `gpt`, `msdos` or
    `bsd`. For UEFI systems, we need to set it to `gpt`. This value
    will be used for all the devices if is not overwritten.

  * `initial_gap`: Integer. Optional. Default: `0`

    Initial gap (empty space) leaved before the first
    partition. Usually is recommended to be 1MB, so GRUB have room to
    write the code needed after the MBR, and the sectors are aligned
    for multiple SSD and hard disk devices. Also is relevant for the
    sector alignment in devices. The valid units are the same for
    `parted`. This value will be used for all the devices if is not
    overwritten.

* `devices`: Dictionary.

  List of devices that will be partitioned. We can indicate already
  present devices, like `/dev/sda` or `/dev/hda`, but we can also
  indicate devices that will be present after the RAID configuration,
  like `/dev/md0` or `/dev/md/myraid`. We can use any valid device
  name in Linux such as all the `/dev/disk/by-id/...`,
  `/dev/disk/by-label/...`, `/dev/disk/by-uuid/...` and others.

  For each device we have:

  * `label`: String. Optional. Default: `msdos`

    Partition label for the device. The meaning and the possible
    values are identical for `label` in the `config` section.

  * `initial_gap`: Integer. Optional. Default: `0`

    Initial gap (empty space) leave before the first partition for
    this device.

  * `partitions`: Array. Optional.

    Partitions inside a device are described with an array. Each
    element of the array is a dictionary that describe a single
    partition.

    * `number`: Integer. Optional. Default: `loop.index`

      Expected partition number. Eventually this parameter will be
      really optional, when the partitioner can deduce it from other
      parameters. Today is better to be explicit in the partition
      number, as this will guarantee that the partition is found in
      the hard disk if present. If is not set, number will be the
      current index position in the array.

    * `id`: String. Optional.

      Full name of the partition. For example, valid ids can be
      `/dev/sda1`, `/dev/md0p1`, etc. Is optional, as the name can be
      deduced from `number`.

    * `size`: Float or String.

      Size of the partition expressed in `parted` units. All the units
      needs to match for partitions on the same device. For example,
      if `initial_gap` or the first partition is expressed in MB, all
      the sized needs to be expressed in MB too.

      The last partition can use the string `rest` to indicate that
      this partition will use all the free space available. If after
      this another partition is defined, Yomi will show a validation
      error.

    * `type`: String.

      A string that indicate for what this partition will be
      used. Yomi recognize several types:

      * `swap`: This partition will be used for SWAP.
      * `linux`: Partition used to root, home or any data.
      * `boot`: Small partition used for GRUB when in BIOS and `gpt`.
      * `efi`: EFI partition used by GRUB when UEFI.
      * `lvm`: Partition used to build an LVM physical volume.
      * `raid`: Partition that will be a component of an array.

Example:

```yaml
partitions:
  config:
    label: gpt
    initial_gap: 1MB
  devices:
    /dev/sda:
      partitions:
        - number: 1
          size: 256MB
          type: efi
        - number: 2
          size: 1024MB
          type: swap
        - number: 3
          size: rest
          type: linux
```

## `lvm` section

To build an LVM we usually create some partitions (in the `partitions`
section) with the `lvm` type set, and in the `lvm` section we describe
the details. This section is a dictionary, were each key is the name
of the LVM volume, and inside it we can find:

* `devices`: Array.

  List of components (partitions or full devices) that will constitute
  the physical volumes and the virtual group of the LVM. If the
  element of the array is a string, this will be the name of a device
  (or partition) that belongs to the physical group. If the element is
  a dictionary it will contains:

  * `name`: String.

    Name of the device or partition.

  The rest of the elements of the dictionary will be passed to the
  `pvcreate` command.

  Note that the name of the virtual group will be the key where this
  definition is under.

* `volumes`: Array.

  Each element of the array will define:

  * `name`: String.

    Name of the logical volume under the volume group.

  The rest of the elements of the dictionary will be passed to the
  `lvcreate` command. For example, `size` and `extents` are used to
  indicate the size of the volume, and they can include a suffix to
  indicate the units. Those units will be the same used for
  `lvcreate`.

The rest of the elements of this section will be passed to the
`vgcreate` command.

Example:

```yaml
lvm:
  system:
    devices:
      - /dev/sda1
      - /dev/sdb1
      - name: /dev/sdc1
        dataalignmentoffset: 7s
    clustered: 'n'
    volumes:
      - name: swap
        size: 1024M
      - name: root
        size: 16384M
      - name: home
        extents: 100%FREE
```

## `raid` section

In the same way that LVM, to create RAID arrays we can setup first
partitions (with the type `raid`) and configure the details in this
section. Also, similar to the LVM section, the keys a correspond to
the name of the device where the RAID will be created. Valid values
are like `/dev/md0` or `/dev/md/system`.

* `level`: String.

   RAID level. Valid values can be `linear`, `raid0`, `0`, `stripe`,
   `raid1`, `1`, `mirror`, `raid4`, `4`, `raid5`, `5`, `raid6`, `6`,
   `raid10`, `10`, `multipath`, `mp`, `faulty`, `container`.

* `devices`: Array.

  List of devices or partitions that build the array.

* `metadata`: String. Optional. Default: `default`

  Metadata version for the superblock. Valid values are `0`, `0.9`,
  `1`, `1.0`, `1.1`, `1.2`, `default`, `ddm`, `imsm`.

The user can specify more parameters that will be passed directly to
`mdadm`, like `spare-devices` to indicate the number of extra devices
in the initial array, or `chunk` to speficy the chunk size.

Example:

```yaml
raid:
  /dev/md0:
    level: 1
    devices:
      - /dev/sda1
      - /dev/sdb1
      - /dev/sdc1
    spare-devices: 1
    metadata: 1.0
```

## `filesystems` section

The partitions, devices or arrays created in previous sections usually
requires a file system. This section will simply list the device name
and the file system (and properties) that will be applied to it.

* `filesystem`. String.

  File system to apply in the device. Valid values are `swap`,
  `linux-swap`, `bfs`, `btrfs`, `cramfs`, `ext2`, `ext3`, `ext4`,
  `minix`, `msdos`, `vfat`. Technically Salt will search for a command
  that match `mkfs.<filesystem>`, so the valid options can be more
  extensive that the one listed here.

* `mountpoint`. String.

  Mount point where the device will be registered in `fstab`.

* `fat`. Integer. Optional.

  If the file system is `vfat` we can force the FAT size, like 12, 16
  or 32.

* `subvolumes`. Dictionary.

  For `btrfs` file systems we can specify more details.

  * `prefix`. String. Optional.

    `btrfs` sub-volume name where the rest of the sub-volumes will be
    under. For example, if we set `prefix` as `@` and we create a
    sub-volume named `var`, Yomi will create it as `@/var`.

  * `subvolume`. Dictionary.

    * `path`. String.

      Path name for the sub-volume.

	* `copy_on_write`. Boolean. Optional. Default: `yes`

      Value for the copy-on-write option in `btrfs`.

Example:

```yaml
filesystems:
  /dev/sda1:
    filesystem: vfat
    mountpoint: /boot/efi
    fat: 32
  /dev/sda2:
    filesystem: swap
  /dev/sda3:
    filesystem: btrfs
    mountpoint: /
    subvolumes:
      prefix: '@'
      subvolume:
        - path: home
        - path: opt
        - path: root
        - path: srv
        - path: tmp
        - path: usr/local
        - path: var
          copy_on_write: no
        - path: boot/grub2/i386-pc
        - path: boot/grub2/x86_64-efi
```

## `bootloader` section

* `device`: String.

  Device name where GRUB2 will be installed. Yomi will take care of
  detecting if is a BIOS or an UEFI setup, and also if Secure-Boot in
  activated, to install and configure the bootloader (or the shim
  loader)

* `kernel`: String. Optional. Default: `splash=silent quiet`

  Line assigned to the `GRUB_CMDLINE_LINUX_DEFAULT` parameter.

* `disable_os_prober`: Boolean. Optional. Default: `False`

  Value for `GRUB_DISABLE_OS_PROBER` parameter.

Example:

```yaml
bootloader:
  device: /dev/sda1
```

## `software` section

We can indicate the repositories that will be registered in the new
installation, and the packages and patterns that will be installed.

* `config`. Dictionary. Optional

  Local configuration for the software section.

  * `minimal`: Boolean. Optional. Default: `no`

    Configure zypper to make a minimal installation, excluding
    recommended, documentation and multi-version packages.

* `repositories`. Dictionary.

  Each key of the dictionary will be the name under where this
  repository is registered, and the key is the URI associated with it.

* `packages`. Array.

  List of packages or patters to be installed.

Example:

```yaml
software:
  repositories:
    repo-oss: "http://download.opensuse.org/tumbleweed/repo/oss"
  packages:
    - patterns-base-base
    - kernel-default
```

## `salt-minion` section

Install and configure the salt-minion service.

* `configure`. Boolean. Optional. Default: `no`

  If `yes`, the configuration and cetificates of the new minion will
  be the same that the current minion that is activated. This will
  copy the minion configuration, certificates and grains.

  This option will be replaced in the future with more detailed ones.

Example:

```yaml
salt-minion:
  configure: yes
```

## `services` section

We can list the services that will be enabled or disabled during boot
time.

* `enabled`. Array. Optional

  List of services that will be enabled and started during the boot.

* `disabled`. Array. Optional

  List of services that will be exclicitly disabled during the boot.

Example:

```yaml
services:
  enabled:
    - salt-minion
```

## `users` section

In this section we can list a simple list of users and passwords that
we expect to find once the system is booted.

* `username`. String.

  Login or username for the user.

* `password`. String. Optional.

  Shadow password hash for the user.

* `certificates`. Array. Optional.

  Certificates that will be added to .ssh/authorized_keys. Use only
  the encoded key (remove the "ssh-rsa" prefix and the "user@host"
  suffix).

Example:

```yaml
users:
  - username: root
    password: "$1$wYJUgpM5$RXMMeASDc035eX.NbYWFl0"
  - username: aplanas
    certificates:
      - "AAAAB3NzaC1yc2EAAAADAQABAAABAQDdP6oez825gnOLVZu70KqJXpqL4fGf\
        aFNk87GSk3xLRjixGtr013+hcN03ZRKU0/2S7J0T/dICc2dhG9xAqa/A31Qac\
        hQeg2RhPxM2SL+wgzx0geDmf6XDhhe8reos5jgzw6Pq59gyWfurlZaMEZAoOY\
        kfNb5OG4vQQN8Z7hldx+DBANPbylApurVz6h5vvRrkPfuRVN5ZxOkI+LeWhpo\
        vX5XK3eTjetAwWEro6AAXpGoQQQDjSOoYHCUmXzcZkmIWEubCZvAI4RZ+XCZs\
        +wTeO2RIRsunqP8J+XW4cZ28RZBc9K4I1BV8C6wBxN328LRQcilzw+Me+Lfre\
        eDPglqx"
```
