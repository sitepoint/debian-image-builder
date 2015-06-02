# Debian GNU/Linux image builder for multiple IaaS providers #

This script bootstraps a basic Debian GNU/Linux installation to create
(currently) either an Amazon machine image or a Google Compute Engine
image. The image contains no latent logfiles, no .bash\_history or
even the apt package cache. In fact, the image is created via
debootstrap and never booted during the creation process, providing a
very clean setup without surprises.

The machine configuration this script creates has been thoroughly
tested on EC2. I no longer test GCE, but accept patches if required.

* Both HVM and PVM EC2 instance types can be created, with either an
  instance store or EBS backed root volume - plugins no longer
  required!

* This script has been tested with AWS on Wheezy and Jessie.

* To create an AMI, this bootstrapper needs to be run on an Amazon EC2
  instance - we'll be attaching an EBS volume temporarily during
  execution.

This project is a fork of the Bash version of build-debian-cloud,
prior to the project switching to Python. This is due to the Python
version proving to be too inflexible for my needs, as I was unable to
easily add the functionaly I required via a plugin due to the chosen
architecture. In my opinion, debian-image-builder is also easier to
read and understand.


## Usage ##

The script is started with ``./debian-image-builder``.  You can choose
to either bootstrap a Debian AMI (``./debian-image-builder ec2``) or a
Google Compute Engine image (``./debian-image-builder gce``).  Both
modes have sensible defaults and can be configured with options and
plugins. To see a list of options use ``--help``.  When creating an
AMI, the script at least needs to know your AWS credentials.

As there are no interactive prompts, the bootstrapping can run
entirely unattended from start till finish. A plugin could optionally
change this behaviour.

Some plugins are included in the plugins directory. A list of external
plugins is also provided there. If none of those scratch your itch,
you can of course write your own plugin (see HOWTO.md in the plugins
directory).


## Examples ##

This basic example creates a 15G EBS-backed HVM instance. Many
defaults are used.

```
./debian-image-builder ec2 --arch amd64 --codename jessie \
    --volume-size 15 \
    --plugin plugins/standard-packages --virt hvm \
    --name "$(date +%Y%m%d%H%M)" \
    --description "Debian 7 (Jessie) 15Gb, HVM, EBS"
```

This next example creates a Wheezy x86_64 paravirtual image with a 10G
instance-backed root volume, formatted to have 5000000 inodes. The
image timezone and locales have been set, and the image name suffix is
the date and time of execution.

```
./debian-image-builder ec2 --arch amd64 --codename wheezy \
    --volume-type instance \
    --filesystem ext4 --volume-size 10 --volume-inodes 5000000 \
    --plugin plugins/standard-packages \
    --timezone Australia/Melbourne --locale en_AU --charmap UTF-8 \
    --virt paravirtual --name "$(date +%Y%m%d%H%M)" \
    --description "Debian 7 (Wheezy) 10Gb, paravirtual, instance-store"
```


## Features ##

### AMI features ###

* EBS booted
* Base installation uses only 289MB
* Base installation bootup time ~45s* (from AMI launch to SSH connectivity)
* Support for both ext* and xfs
* Uses standard Debian Xen kernel from apt
* update-grub creates an actual menu.lst which pvGrub can read
* ec2 system log is not cluttered by grub menu
* ec2 startup scripts:
  * `ec2-get-credentials`: Copies the ec2 keypair to `~/.ssh/authorized_keys`
  * `ec2-run-user-data`: If the userdata starts with `#!` it will be executed
  * `generate-ssh-hostkeys`: Generates hostkeys for sshd on first boot
  * `expand-volume`: Expands the root partition to the volume size

*\*The bootup time was measured with [this script](https://gist.github.com/3813743).*

### Bootstrapper (AMI) features ###

* EBS volume is automatically created, mounted, formatted, unmounted, "snapshotted" and deleted
* AMI is automatically registered with the right kernels for the current region of the host machine
* Supports Wheezy and Jessie
* Can create both 32-bit and 64-bit AMIs
* Plugin system to keep the bootstrapping process automated
* The process is divided into simple task based scripts
* Uses only free software in accordance with the [Debian Social Contract](http://www.debian.org/social_contract)
  (eg. Unlike other solutions, we use [euca2ools](http://www.eucalyptus.com/download/euca2ools)
  instead of Amazon's proprietary [EC2 API Tools](http://aws.amazon.com/developertools/351)).
