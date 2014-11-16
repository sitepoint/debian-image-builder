## init.d scripts ##
These scripts are added to the generated EC2 image and executed on boot.

The scripts are required for the following tasks:

* `ec2-get-credentials`
  Retrieve and install the SSH public key to /root/authorized_keys.
* `ec2-run-user-data`
  Run instance user-data if it looks like a script, on first launch only.
* `expand-volume`
  Expand the filesystem of the mounted root volume to its maximum possible size.
* `generate-ssh-hostkeys`
  Generate host keys if they do not exist.
