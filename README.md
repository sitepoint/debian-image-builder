debian-image-builder
====================

Debian GNU/Linux image builder for multiple IaaS providers.

This script bootstraps a basic Debian GNU/Linux installation to create
(currently) either an Amazon Machine Image or a Google Compute Engine
image. The image contains no latent logfiles, no .bash\_history or
even the apt package cache. In fact, the image is created via
debootstrap and never booted during the creation process, providing a
very clean setup without surprises.

The machine configuration this script creates has been thoroughly
tested on EC2. I no longer test GCE, but accept patches if required.


Features
--------

* Both HVM and PVM EC2 instance types can be created, with either an
  instance store or EBS backed root volume.

* Template support makes recreating newer versions of AMIs quick and
  easy.

* This script has been tested with AWS on Wheezy and Jessie.

Note: To create an AMI, debian-image-builder needs to be run on an
Amazon EC2 instance - we'll be attaching an EBS volume temporarily
during execution, regardless of the type of AMI being created.


Why debian-image-builder?
-------------------------

This project started out as a fork of the Bash version of
[build-debian-cloud](https://github.com/camptocamp/build-debian-cloud),
prior to the project switching to bootstrap-vz (Python). This was
initially due to bootstrap-vz proving to be too inflexible for my
needs, and my pull requests for build-debian-cloud were no longer
accepted due to the desire to see bootstrap-vz gain traction.

In my opinion, debian-image-builder is easier to read and understand,
while retaining the power and flexibility of alternatives.

### Advantages of debian-image-builder over bootstrap-vz include: ###

* No dependendencies on anything outside of what is packaged within
  Debian (aside from euca2ools which is automatically fetched and
  installed if required).

* A commitment to using only free software tools.
  debian-image-builder uses only free software in accordance with the
  [Debian Social Contract](http://www.debian.org/social_contract).
  Unlike other solutions, we use
  [euca2ools](http://www.eucalyptus.com/download/euca2ools) instead of
  Amazon's proprietary
  [EC2 API Tools](http://aws.amazon.com/developertools/351) software.
  Work to this effect has been contributed to projects
  debian-image-builder depends on. eg.
  https://eucalyptus.atlassian.net/browse/TOOLS-294

* The ``no-systemd`` plugin. Keep your Jessie AMIs as free from
  Systemd as is reasonably possible (still using only official Debian
  packages).

* The ``pause-before-umount`` plugin. Allows you to inspect the root
  filesystem for problems (and optionally make manual changes) just
  prior to the AMI registration step. Great if you don't want to make
  a dedicated plugin for a once-off custom AMI, or if you are just
  trying to debug something.

* The ``move-s3-path`` plugin. Have all of your instance-backed AMIs
  stored safely in the same bucket, using your preferred directory
  structure!

* The ``grant-launch-permission`` plugin. Automatically add launch
  permissions to other AWS accounts for generated AMIs.

* Support for more EC2 AMI types. Want Jessie HVM AMIs with a
  instance-store volume for the root device (for example)? We've got
  you covered.

* Jessie on EC2 uses the ``cloud-init`` package by default.


Setup
-----

The script is started with ``./debian-image-builder``.  You can choose
to either bootstrap a Debian AMI (``./debian-image-builder ec2``) or a
Google Compute Engine image (``./debian-image-builder gce``).  Both
modes have sensible defaults and can be configured with options and
plugins. To see a list of options use ``--help``.  When creating an
AMI, the script at least needs to know your AWS credentials.

As there are no interactive prompts, the bootstrapping can run
entirely unattended from start till finish. Plugins can optionally
change this behaviour (eg. ``pause-before-umount``).

A number of plugins are included in the plugins directory. A list of
external plugins is also provided there in the README.md file. If
none of those scratch your itch, you can of course very easily write
your own plugin (see HOWTO.md in the plugins directory).


### Usage examples ###

Start by switching to the root user, and exporting the environment
variables required by euca2ools:

```
export AWS_ACCESS_KEY='access_key'
export AWS_SECRET_KEY='secret_key'
export EC2_CERT="${HOME}/x.509/cert.pem"
export EC2_PRIVATE_KEY="${HOME}/x.509/pk.pem"
export EC2_USER_ID="5555-5555-5555"
export EC2_REGION="us-west-2"
```

Consult the AWS documentation if any of these are unfamiliar to you.
Next, you will need to set the path to your cloud certificate. Amazon
includes this in the proprietary ec2-ami-tools package, and it is
possible we need a license to redistribute it. Hence, you'll have to
obtain that yourself either manually or using the included ``getcert``
script.

```
export EUCALYPTUS_CERT="${HOME}/cert-ec2.pem"
```

If you wish to use partitioned block devices on EC2 for use with
HVM-compatible AMIs, please ensure hdparm and parted is installed.

```
apt-get install hdparm parted
```

If creating AMIs with instance-store volumes, you will need to set
S3_BUCKET, and optionally also the CUSTOM_S3_PATH environment
variables. S3_BUCKET is used to specify the S3 bucket (minus the s3://
prefix and any suffix path) you wish to bundle and upload your AMI
to. If you do not specify CUSTOM_S3_PATH (and don't use the
``move-s3-path`` plugin), your AMI will be registered here. However if
you would rather have a more organised path like
s3://my-company-region/debian-gnu_linux/jessie/x86_64/201506191210/
where you can consolidate multiple AMIs into a single bucket, specify
the bucket and path name for the CUSTOM_S3_PATH environment variable
(again, sans the s3:// prefix) and the AMI will be registered there
instead.

Note that due to limitations of AWS and/or euca2ools, the AMI will be
uploaded to S3_BUCKET first, and then moved automatically (using
``s3cmd``) to CUSTOM_S3_PATH during a later step before finally being
registered. While it's generally safe to execute debian-image-builder
multiple times simultaneously for quickly building a large number of
AMIs, you will need to make sure no two builds are using S3_BUCKET at
the same time (and debian-image-builder will fail to start if it
detects this condition).

Also note that all included templates make use of the ``move-s3-path``
plugin, so you will need to set CUSTOM_S3_PATH if using those, or
delete the plugin reference from the templates otherwise. Modifying
template files is a trivial process.

```
export S3_BUCKET="my-temporary-build-bucket"
export CUSTOM_S3_PATH="my-${EC2_REGION}-images/debian-gnu_linux/jessie"
```

The next part of the setup process (if generating instance-store AMIs
using the move-s3-path plugin) is to install and configure
``s3cmd``. The s3cmd configure step will run you though a quick setup
wizard, since the tool does not recognise the EC2_* environment
variables.

```
apt-get install s3cmd
s3cmd --configure
```

If using the grant-launch-permission-tasks plugin, you will also need
to set the following environment variable:

```
export LAUNCH_ACCOUNTS="1234-5678-9012 4321-8765-2190 9876-5432-1098"
```

This will allow the grant-launch-permission-tasks plugin to grant
launch authorizatinon to accounts specified in the space-separated
string (with optional dashes). This can come in hand when, for
example, you want to share access to your AMIs with a separate account
for staging or development.

Getting the environment into a good state to generate images can take
some time and it's easy to overlook something, so you may want to run
the included ``envcheck`` script to verify that everything appears to
be in place. However debian-image-builder generally does a good job of
failing gracefully when something is amiss.


Usage
-----

Now we are ready to start creating AMIs. Using one of the included
templates is the easiest way to get started. This example creates a
Jessie HVM AMI with an instance-store root volume of 10Gb and systemd
replaced with sysvinit:

```
./debian-image-builder ec2 --template \
    templates/debian-jessie-hvm-amd64-instance-10-sysvinit.cfg
```

This next example creates a 50G EBS-backed HVM Jessie instance. Many
defaults are used:

```
./debian-image-builder ec2 --arch amd64 --codename jessie \
    --volume-size 50 \
    --plugin plugins/standard-packages --virt hvm \
    --name-suffix "$(date +%Y%m%d%H%M)" \
    --description "Debian 8 (Jessie) 50Gb, HVM, EBS"
```

This final example creates a Wheezy x86_64 paravirtual image with a 5G
instance-backed root volume, formatted to have 5000000 inodes. The
image time-zone and locales have been set, and the image name suffix
is the date and time of execution:

```
./debian-image-builder ec2 --arch amd64 --codename wheezy \
    --volume-type instance \
    --filesystem ext4 --volume-size 5 --volume-inodes 5000000 \
    --plugin plugins/standard-packages \
    --plugin plugins/move-s3-path \
    --timezone Australia/Melbourne --locale en_AU --charmap UTF-8 \
    --virt paravirtual --name-suffix "$(date +%Y%m%d%H%M)" \
    --description "Debian 7 (Wheezy) 5Gb, paravirtual, instance-store"
```

Bugs, suggestions, patches and plugins are all welcome. Have fun!
