## TODO ##

### In order of priority: ###

1. Validation checks for provider inputs
    * Gather inputs via getopt/getopts instead of a massive case statement?
2. Automate moving the bundle to a S3 bucket subfolder.
    * Maybe euca2ools can now handle this without a plugin?
3. Partitioned images support to avoid GRUB warnings.
4. Support cloud-init.
    * `grub.d/ec2-get-credentials` to be replaced by `cloud-init`.
    * `grub.d/ec2-run-user-data` to be replaced by `cloud-init`.
    * `grub.d/expand-volume` to be replaced by `cloud-initramfs-growroot`
    * `grub.d/generate-ssh-hostkeys` to be replaced by `cloud-init`.
    * Evaluate the possibility of adding cloud-initramfs-rescuevol to the generated initramfs.
5. New plugin to automatically share images with other specific accounts.
6. See if we can avoid using the custom grub.d file.
