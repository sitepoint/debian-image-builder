# Regression checklist #
This is a checklist, to keep a record of subtle bugs in the AMI that
can be hard to spot. Before publishing any AMI, this list should be
consulted first.

*Note that the bootstrapper does all the listed things for
 you. However, regressions are always a possibility.*

* EBS root volume, should be set to "Delete on Termination".
* The AWS credentials should not be obtainable from the root volume
  (see below for instructions).
* The ssh hostkeys of the bootstrapping machine are copied into the
  image by debootstrap. To remove them properly they should be
  shredded with `shred --remove FILENAME`.
* There should be no `.*_history` files in any home directories.
* The default locale should be generated.
* A timezone should be set.
* The apt sources list should at least contain a security update
  mirror and a main repository mirror.
* There should be no broken dependencies or un-upgraded packages.
* The apt package cache should be deleted.


To check a volume for any sensitive data run the following commands:

```
apt-get update
apt-get install binutils
strings /dev/xvda | grep -A10 -B10 'AWS_|EC2_'
```
