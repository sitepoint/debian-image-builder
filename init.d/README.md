## init.d scripts ##
This script is added to generated images and executed on boot.

* `generate-ssh-hostkeys`
  Generate host keys if they do not exist.

These additioanl scripts are added to generated EC2 images.

* `ec2-get-credentials`
  Retrieve and install the SSH public key to /root/authorized_keys.
* `ec2-run-user-data`
  Run instance user-data if it looks like a script, on first launch only.
* `expand-volume`
  Expand the filesystem of the mounted root volume to its maximum possible size.
