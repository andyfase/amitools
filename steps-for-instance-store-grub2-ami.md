# Steps for creating clean instance store backed grub2 AMI

## Assumptions

1. You have a base AMI (either instance store or EBS backed) using GRUB2 as a bootloader
2. You can spin up a EC2 instance based on the above and have root access to it through sudo
3. Your aiming to not include any of the required tooling into the new AMI that is produced

## Known package dependencies

* `parted`
* `ruby`

## Instructions 

1. Boot EC2 instance
2. Perform modifications needed to instance you want baked into AMI
3. Install package dependencies into `/tmp/build` directory, this directory is skipped when creating the image file

```
mkdir -p /tmp/build/ruby
sudo yum --installroot /tmp/build/ruby/ --releasever=7 -y instal

mkdir =p /tmp/build/utils
sudo yum --installroot /tmp/build/utils/ --releasever=7 -y install parted
```

4\. Fetch modified AMI-tools

```
mkdir -p /tmp/build/amitools
wget -qO- https://github.com/andyfase/amitools/raw/master/ami-tools.tar | tar --warning=no-unknown-keyword -x -C /tmp/build/amitools/
```

5\. Generate and copy accross needed [EC2 signing certificates](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/set-up-ami-tools.html?icmpid=docs_iam_console#ami-tools-managing-certs) to `/tmp/build/` directory

5\. Use `ec2-bundle-vol` to create image file



* Because `ec2-bundle-vol` is a hard-coded sym-link to `/usr/local/ec2/amitools/bundlevol` which itself is a simple call to ruby (again hard-coded to /usr/local) instead we call the `/tmp/build/amitools` ruby file directly.
* We pass in a new parameter `--grub2` to to build the new image using grub2 tooling vs grub legacy
* We over-ride the location of `ec2cert` which defaults to look within `/etc/amitools`
* We add `/tmp/build` as a excluded directory for the rsync which occurs to the loopback mounted volume.
* We include a bunch of paths to various ruby gems within the `/tmp/build` directory hierarchy as we are using the tooling outside of its normal install location


```
sudo PATH=$PATH:/tmp/build/utils/usr/sbin LD_LIBRARY_PATH=/tmp/build/ruby/usr/lib64 RUBYLIB=$RUBYLIB:/tmp/build/ruby/usr/share/ruby/:/tmp/build/ruby/usr/lib64/ruby/:/tmp/build/ruby/usr/share/rubygems:/tmp/build/amitools/usr/lib/ruby/site_ruby/:/tmp/build/amitools/usr/lib/tmp/build/ruby/bin/ruby /tmp/build/ruby/bin/ruby /tmp/build/amitools/usr/lib/ruby/site_ruby/ec2/amitools/bundlevol.rb -k /tmp/build/iam_tools.key -c /tmp/build/iam_tools_cert.pem -u <account#> -r x86_64 -e /tmp/build  -P mbr --ec2cert /tmp/build/amitools/etc/ec2/amitools/cert-ec2.pem --grub2
```

6\. Upload created image to S3

```
sudo PATH=$PATH:/tmp/build/utils/usr/sbin LD_LIBRARY_PATH=/tmp/build/ruby/usr/lib64 RUBYLIB=$RUBYLIB:/tmp/build/ruby/usr/share/ruby/:/tmp/build/ruby/usr/lib64/ruby/:/tmp/build/ruby/usr/share/rubygems:/tmp/build/amitools/usr/lib/ruby/site_ruby/:/tmp/build/amitools/usr/lib/tmp/build/ruby/bin/ruby /tmp/build/ruby/bin/ruby /tmp/build/amitools/usr/lib/ruby/site_ruby/ec2/amitools/uploadbundle.rb -b <bucket>/path/for/image -a <iam_access_key> -s <iam_access_key_secret> -m /tmp/image.manifest.xml --region <region>
```

7\. On any machine with aws-cli installed register the new ami

```
aws ec2 register-image --image-location <bucket>/path/for/image/image.manifest.xml --name <name_of_image> --description "description of image" --architecture x86_64 --virtualization-type hvm --sriov-net-support simple --ena-support --region <region>
```