debian-image-builder
====================

Debian GNU/Linux image builder for multiple IaaS providers.

This script bootstraps a basic Debian GNU/Linux installation to create
a Amazon Machine Image. The image contains no latent logfiles, no
.bash\_history or even the apt package cache. In fact, the image is
created via debootstrap and never booted during the creation process,
providing a very clean setup without surprises.

The machine configuration this script creates has been thoroughly
tested on EC2. I am open to accepting patches to support other
providers, if they will be maintained. Adding new providers should be
quite straighforward.


Features
--------

* Both HVM and PVM EC2 instance types can be created, with either an
  instance store or EBS backed root volume.

* Template support makes recreating newer versions of AMIs quick and
  easy.

* This script is currently tested on AWS to build Stretch (stable) and
  Jessie (oldstable) images.

Note: To create an AMI, debian-image-builder needs to be run on an
Amazon EC2 instance - we'll be attaching a new EBS volume temporarily
during execution regardless of the type of AMI being created, which
will be deleted afterwards.


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
  Debian (aside from euca2ools on Debian Jessie and below).

* A commitment to using only free software tools.
  debian-image-builder uses only free software in accordance with the
  [Debian Social Contract](http://www.debian.org/social_contract).

* The ``no-systemd`` plugin. Keep your AMIs as free from Systemd as is
  reasonably possible (still using only official Debian packages).

* The ``pause-before-umount`` plugin. Allows you to inspect the root
  filesystem for problems (and optionally make manual changes) just
  prior to the AMI registration step. Great if you don't want to make
  a dedicated plugin for a once-off custom AMI, or if you are just
  trying to debug something.

* Have all of your instance-backed AMIs stored safely in the same
  bucket with different prefixes, using your preferred directory
  structure!

* The ``grant-launch-permission`` plugin. Automatically add launch
  permissions to other AWS accounts for generated AMIs.

* Support for more EC2 AMI types. Want Jessie HVM AMIs with a
  instance-store volume for the root device (for example)? We've got
  you covered.

* Uses the ``cloud-init`` package by default.


Setup
-----

The script is started with ``./debian-image-builder``.  The first
argument should be the provider. Currently only ec2 is supported
(``./debian-image-builder ec2``) however additing additional providers
should be straightforward (and patches are welcome). Supported
environments should have sensible defaults and can be configured with
options and plugins. To see a list of options use ``--help``. When
creating an AMI, the script needs to be provided with appropriate AWS
credentials (see below).

As there are no interactive prompts, the bootstrapping can run
entirely unattended from start to finish. Plugins can optionally
change this behaviour (eg. ``pause-before-umount``).

A number of plugins are included in the plugins directory. A list of
external plugins is also provided there in the
[README.md](./plugins/README.md) file. If none of those scratch your
itch, you can of course very easily write your own plugin (see
[HOWTO.md](./plugins/HOWTO.md) in the plugins directory).


### EC2 usage examples ###

Start by switching to the root user, and exporting the environment
variables required by awscli:

```
export AWS_ACCESS_KEY_ID='access_key'
export AWS_SECRET_ACCESS_KEY='secret_key'
```

Also be sure to export the following:

```
export AWS_DEFAULT_REGION="us-west-2"
```

If creating an instance store AMI, you will also require the
following:

```
export EC2_CERT="${HOME}/x.509/cert.pem"
export EC2_PRIVATE_KEY="${HOME}/x.509/pk.pem"
export EC2_USER_ID="5555-5555-5555"
```

If creating AMIs with instance-store volumes, you will need to set
`S3_BUCKET` using the `BUCKET[/PREFIX]` format. `S3_BUCKET` is used to
specify the S3 bucket (minus the `s3://` prefix) you wish to bundle
and upload your AMI to.

Be careful to ensure that S3 buckets specified by `S3_BUCKET` are
located in the same region as the instance running
debian-image-builder. Failure to do so will result in a "Bucket is not
available from endpoint" error near the end of the build process.

If you would like to have an organised path like
`s3://my-company-region-ami/debian-gnu_linux/stretch/x86_64/201804201821/`
where you can consolidate multiple AMIs into a single bucket, you
could specify the following `S3_BUCKET` value:

```
export S3_BUCKET="my-company-${AWS_DEFAULT_REGION}-ami/debian-gnu_linux/stretch/x86_64/$(date +'%Y%m%d%H%M')"
```

Note there is currently a bug running the `euca-upload-bundle` command
(which is called automatically if creating instance-backed AMIs) on
Stretch that results in an `InvalidHeader` exception. For now this can
be avoided by temporarily editing
`/usr/lib/python2.7/dist-packages/requests/models.py` and making the
changes shown
[here](https://github.com/eucalyptus/euca2ools/pull/83/commits/4d3f5dddcb019f1ce02e9352fbea7a6e86ba8e3c). This
bug doesn't affect the creation of the more common EBS backed AMI
types. For details see
[this](https://github.com/requests/requests/issues/3477)
python-requests issue and
[this](https://github.com/eucalyptus/euca2ools/pull/83) euca2ools
PR. The included envcheck tool will try to detect hosts affected by
this issue.

If using the `grant-launch-permission` plugin, you will also need
to set the following environment variable:

```
export LAUNCH_ACCOUNTS="1234-5678-9012 4321-8765-2190 9876-5432-1098"
```

This will allow the `grant-launch-permission` plugin to grant launch
authorizatinon to accounts specified in the space-separated string
(with optional dashes). This can come in handy when, for example, you
want to share access to your AMIs with a separate account for staging
or development.

Getting the environment into a good state to generate images can take
some time and it's easy to overlook something, so you may want to run
the included `envcheck` script to verify that everything appears to
be in place. However debian-image-builder generally does a good job of
failing gracefully when something is amiss.

debian-image-builder aims to be safe to execute multiple times
simultaneously for quickly building a large number of AMIs.


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

This next example creates a 50G EBS-backed HVM Stretch instance. Many
defaults are used:

```
./debian-image-builder ec2 --arch amd64 --codename stretch \
    --volume-size 50 \
    --plugin plugins/standard-packages --virt hvm \
    --name-suffix "$(date +%Y%m%d%H%M)" \
    --description "Debian 9 (Stretch) 50Gb, HVM, EBS"
```

This final example creates a Jessie x86_64 paravirtual image with a 5G
instance-backed root volume, formatted to have 5000000 inodes. The
image time-zone and locales have been set, and the image name suffix
is the date and time of execution:

```
./debian-image-builder ec2 --arch amd64 --codename jessie \
    --volume-type instance \
    --filesystem ext4 --volume-size 5 --volume-inodes 5000000 \
    --plugin plugins/standard-packages \
    --timezone Australia/Melbourne --locale en_AU --charmap UTF-8 \
    --virt paravirtual --name-suffix "$(date +%Y%m%d%H%M)" \
    --description "Debian 8 (Jessie) 5Gb, paravirtual, instance-store"
```

Bugs, suggestions, patches and plugins are all welcome. Have fun!
