# Cloud init

https://cloudinit.readthedocs.io/en/latest/tutorial/qemu.html

Download sample ubuntu image
```
cd test
wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img
```

## Qemu

Quck EMUlator is capable of running Xen and KVM Kernel based virtual machines on
linux, and Hypervisor.framework HVF on macOS.
It is in core of libvirt, LXD and vagrant.
Install on Ubuntu
```
sudo apt install qemu-system-x86
```

Create sample files for user-data meta-data
```
# test/user-data
#cloud-config
password: password
chpasswd:
  expire: False
users:
  - name: ubuntu
    ssh-authorized-keys:
      - ssh-rsa AAAAB3....
```
and run server for Instance Metadata
Service IMDS
```
cd test
python3 -m http.server --directory .

# check that you can access files eq:
curl localhost:8000/user-data
```

and start emulator on ubuntu using kvm
```
qemu-system-x86_64                                            \
    -net nic                                                    \
    -net user                                                   \
    -machine accel=kvm:tcg                                      \
    -cpu host                                                   \
    -m 512                                                      \
    -nographic                                                  \
    -hda jammy-server-cloudimg-amd64.img                        \
    -smbios type=1,serial=ds='nocloud;s=http://10.0.2.2:8000/'
```

on macos we are using default
```
qemu-system-x86_64 -net nic -net user,hostfwd=tcp::2222-:22 -m 512 -nographic -hda jammy-server-cloudimg-amd64.img -smbios type=1,serial=ds='nocloud;s=http://10.0.2.2:8000/'
```

connect to it
```
ssh -p 2222 user@localhost
```

Login with ubuntu/password.
Exit qemu with Ctrl-A + x and write `quit`

## LXD

```
sudo snap install lxd
```
create sample userdata
```
cat > /tmp/my-user-data << HERE_DOC
#cloud-config
runcmd:
  - echo 'Hello, World!' > /var/tmp/hello-world.txt
HERE_DOC
```
start machine
```
lxc launch ubuntu:focal i1 --config=user.user-data="$(cat user-data)"

# or in three steps
lxc init ubuntu:focal i1
lxc config set i1 user.user-data - < user-data
# check that the right file was adde to userdata
lxc config show i1
lxc start i1
```
if you want to update you need to remove the instance since stop will not help,
or clean the state/cache since cloud-init will not rerun if already completed
```
lxc rm -f i1
```
connect with
```
lxc shell i1
```

## Cloud init

There are two steps
https://cloudinit.readthedocs.io/en/latest/explanation/introduction.html#how-does-cloud-init-work
* early boot before networking configuration: identify datasource for all
  congifuration data (metadata like instance ID and network config, or vendor
  and user data like ssh keys, hardware optimisation)
* late boot is after network is configurated, interact with tools like ansible
  for more complex configuration, user accounts, execute user scripts like
  inject ssh keys to authorized_keys file

### Status

Check the status
```
cloud-init status --wait

# print cloudinit progress
status: done
```

### Query

Verify user data content
```
cloud-init query userdata

# this will show file for userdata
```

### Schema

You can see the logs on
```
cat /var/log/cloud-init-output.log
cat /var/log/cloud-init.log
```

Assert valid config, and show files where it is stored

1. user-data at /var/lib/cloud/instance/cloud-config.txt
  but original is on /var/lib/cloud/instance/user-data.txt
2. vendor-data at /var/lib/cloud/instance/vendor-cloud-config.txt
3. network-config at /var/lib/cloud/instance/network-config.json

/var/lib/cloud/instance is a link to /var/lib/cloud/instances/i1

Main cloud init configuration is /etc/cloud/cloud.cfg 
(provided by os, install fresh with `apt intall cloud-init`)

```
cloud-init schema --system --annotate

# this will show file paths and errors, eg
Found cloud-config data types: user-data, network-config

1. user-data at /var/lib/cloud/instances/someid_somehostname/cloud-config.txt:
#cloud-config

# from 1 files
# part-001

# E1: Additional properties are not allowed ('ssh-authorized-keys' was unexpected)
```

Not sure how to solve this issue, but `cloud-init clean && cloud-init init`
will add the keys, but I see empty `/var/lib/cloud/instance/user-data.txt`

### Clean

Remove the state, logs and cache and reboot
```
cloud-init clean --logs --reboot
```
rerun cloudinit init, configuration and final modules
```
cloud-init init
cloud-init modules -m config
cloud-init modules -m final
```

## Modules

https://cloudinit.readthedocs.io/en/latest/reference/modules.html#modules


## Netplan

https://cloudinit.readthedocs.io/en/latest/reference/network-config-format-v2.html#network-config-v2

