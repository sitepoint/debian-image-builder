## TODO ##

### In order of priority: ###

1. Validation checks for provider inputs
    * Gather inputs via getopt/getopts instead of a massive case statement?
2. Support cloud-init.
    * `grub.d/ec2-get-credentials` to be replaced by `cloud-init`.
    * `grub.d/ec2-run-user-data` to be replaced by `cloud-init`.
    * `grub.d/expand-volume` to be replaced by `cloud-initramfs-growroot`
    * `grub.d/generate-ssh-hostkeys` to be replaced by `cloud-init`.
    * Evaluate the possibility of adding cloud-initramfs-rescuevol to the generated initramfs.
3. Figure out what the problem is with isc-dhcp-client and see if it's fixable. We should use the default Debian packages wherever possible to improve consistency. Reportedly, isc-dhcp-client is only incompatible with Wheezy.
4. Template support need not be EC2-specific.
