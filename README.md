# Cloud init

## Qemu

https://cloudinit.readthedocs.io/en/latest/tutorial/qemu.html

Download sample ubuntu image
```
cd test
wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img
```

Quck EMUlator is capable of running Xen and KVM Kernel based virtual machines on
linux, and Hypervisor.framework HVF on macOS.
It is in core of libvirt, LXD and vagrant.
Install on Ubuntu
```
sudo apt install qemu-system-x86
```

Create sample script for user-data meta-data
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

# check that you can access web eq:
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

Boot stages
https://cloudinit.readthedocs.io/en/latest/explanation/boot.html
* datect: `ds-identify` will determine the platform
* local: `cloud-init-local.service` finds the datasource and apply network
  configuration (from datasource metadata) or fallback (dhcp on eth0) or no
  network when configuration exists `network: {config: disabled}` in
  `/etc/cloud/cloud.cfg` or
  `/etc/cloud/cloud.cfg.d/99-disable-network-config.cfg`. This stage must block
  network bring-up or stale configuration that might been applied, instead,
  cloud-init exits and expect the continued boot of the operating system to
  bring network configuration up as it is configurated.
* network: `cloud-init.service` after local stage and configurated networking is
  up. It retrieve any `#include`, runs `disk_setup` and `mounts` modules,
  part-handler and boothooks
* config: `cloud-config.service` runs `cloud_config_modules` modules (do not
  have effect on other stages, such as `runcmd`). User data yml can be provided
  starting with `#cloud-config` so it become cloud config data for example
  https://cloudinit.readthedocs.io/en/latest/reference/examples.html#including-users-and-groups
  and there are other type of user data formats: script `#!`, `#include`
  https://cloudinit.readthedocs.io/en/latest/explanation/format.html
* final: `cloud-final.service` runs `cloud_final_modules` (traditional
  `rc.local`) scripts after logging into a system such as package installation,
  ansible and user-defined scripts from user data. For external scripts looking
  to wait untill cloud-init is finished, can use `cloud-init status --wait`

First boot runs all "per-instance" configuration, whereas subsequent boot run
only "per-boot" configuration (ex on reboot). Also, instance could be launched
from an image captured from a launched instance, and cloud-init check the
instance ID in cache againts the instance ID it determines at runtime (if they
do not match, it is first boot). If you want, it can also `trust` the instance
ID that is present in the system unconditionally using `manual_cache_clean:
true` (if it is false (default) it will clean the cache if instance IDs do
not match). If your image had `manual_cache_clean: true` ie in trust mode, than
new instances from that image will be `trust` mode, unless you manually clean
the cache.
So we have following event types:
BOOT_NEW_INSTANCE (first boot) and BOOT (any boot other than first boot),
BOOT_LEGACY (similar to BOOT, but applies network config during local and
network stage, exists only for regression reasons), HOTPLUG (dynamic add of a
system device). In future we will have METADATA_CHANGE (instance's metadata has
changed, USER_REQUEST directed request to update).



### Status

Check the status print cloudinit progress
```
cloud-init status
status: running

cloud-init status --wait
.....
# when it is finished
status: done

cloud-init status --long
```

https://cloudinit.readthedocs.io/en/latest/howto/debugging.html#cloud-init-did-not-run
or debug services with
```
systemctl status cloud-init-local.service cloud-init.service\
   cloud-config.service cloud-final.service
```

debug with analize
https://cloudinit.readthedocs.io/en/latest/explanation/analyze.html
```
cloud-init analyze blame
cloud-init analyze show
cloud-init analyze dump
cloud-init analyze boot
```

### Query

Verify user data content
```
cloud-init query --all

# this will show file for userdata
cloud-init query userdata

# query keys
cloud-init query --list-keys
# show the value
cloud-init query ds.meta_data
```

also you can check if file exists and its timestamp when cloud-init completed
```
/var/lib/cloud/instance/boot-finished
```

### Schema

You can see the logs on
https://cloudinit.readthedocs.io/en/latest/reference/user_files.html
```
cat /var/log/cloud-init-output.log
cat /var/log/cloud-init.log
```

Assert valid config of user data and show files where it is stored

Datasources from cloud provider are called metadata and includes instance id,
server name, display name and are stored in `/run/cloud-init/instance-data.json`
https://cloudinit.readthedocs.io/en/latest/reference/datasources.html#datasources
Also it searches for network configuration in those metadata and write to
`/var/cloud-init/network-config.json`
https://cloudinit.readthedocs.io/en/latest/reference/network-config.html

1. user-data at /var/lib/cloud/instance/cloud-config.txt
  but original is on /var/lib/cloud/instance/user-data.txt
2. vendor-data at /var/lib/cloud/instance/vendor-cloud-config.txt
3. network-config at /var/lib/cloud/instance/network-config.json

`/var/lib/cloud/instance` is a link to `/var/lib/cloud/instances/i1`

semaphores in `/var/lib/cloud/sem` and `/var/lib/cloud/instance/sem`

Main cloud init configuration is `/etc/cloud/cloud.cfg` and
`/etc/cloud/cloud.cfg.d/*`
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

Not sure how to solve this error, but the keys are added, birth_certificate
created.

### Clean & init to rerun

Remove the state, logs and cache and reboot
for example after updating terraform scripts and t apply
```
# this will remove whole folder `/var/lib/cloud`
cloud-init clean

# remove also the logs
cloud-init clean --logs --reboot
```

rerun cloudinit init, configuration and final modules
On AWS it will fetch user data from AWS IMDS (instance metada service)
```
# check latest version
curl http://169.254.169.254/latest/user-data

# this will fetch user-data and store locally, mount filesystem, regenerate keys
# this will recreate instance folder
# note that it will create semaphores in /var/lib/cloud/instance/sem
cloud-init init

### THIS IS A PLACE TO UPDATE /var/lib/cloud/instance/cloud-config.txt

# this will show file for userdata
# from /var/lib/cloud/instance/user-data.txt
# not from /var/lib/cloud/instance/cloud-config.txt
cloud-init query userdata
# but the following command will use /var/lib/cloud/instance/cloud-config.txt

# process #cloud-config, create scripts/runcmd
cloud-init modules -m config
# since config is default module, we can simple run
cloud-init modules

### IF YOU WANT TO UPDATE /var/lib/cloud/instance/cloud-config.txt recreate runcmd
### without previous clean you can do with manually remove but also a file from sem
### rm /var/lib/cloud/instance/scripts/runcmd
### rm /var/lib/cloud/instance/sem/config_runcmd
### cloud-init modules # to recreate /var/lib/cloud/instance/scripts/runcmd
### manually run runcmd since final is performed only after clean
### /var/lib/cloud/instance/scripts/runcmd
### to run final you need to rm -rf /var/lib/cloud/instance/sem/*

# run runcmd step, logs in /var/log/cloud-init-output.log
cloud-init modules -m final
# see the output
cat /var/log/cloud-init-output.log
# also check the status
cloud-init status
cat /var/lib/cloud/data/status.json
```

Run specific section or manually run generated scripts
```
cloud-init single --name=write_files
cloud-init single --name=runcmd
/var/lib/cloud/instance/scripts/runcmd
```

Easier could be to use alias and link to log
```
alias c=cloud-init
c init

alias cl="cat /var/log/cloud-init-output.log"
cl
```
https://cloudinit.readthedocs.io/en/latest/howto/rerun_cloud_init.html#run-a-single-cloud-init-module
You can run single module
```
sudo cloud-init single --name cc_ssh --frequency always
```

## Modules

https://cloudinit.readthedocs.io/en/latest/reference/modules.html#modules


## Netplan

https://cloudinit.readthedocs.io/en/latest/reference/network-config.html
It searches for network configurtion in metadata datasources, system config
(default `/etc/cloud/cloud.cfg.d/`) and kernel command line (`ip=` or
`network-config=`).

https://cloudinit.readthedocs.io/en/latest/reference/network-config-format-v2.html#network-config-v2

```
# network-config
network:
  version: 2
  ethernets: []
```

Supported devide types are:
* `ethernets:`
* `bonds:`
* `bridges:`
* `vlans:`

Each type block contains device definition as a map, keys are called
Configuration IDs. There are two classes of device definitions:
* Physical devices (ethernet, wifi) can be selected by `match:` rule based on
  name, mac address or driver, so it can be a group of device definitions. When
  no `match:` rule is specified, ID is used to match interface name, otherwise
  ID is only a name of a group.
* Virtual devices (veth, bridge, bond)

`match:`
* `name: enp2*` only networkd supports globbing
* `macaddress: "11:22:33:aa:bb:cc"` does not support globs
* `driver: ixgbe` match on driver is only supported with networkd
* `set-name: my` first matched devices will have this name instead of udev's
* `wakeonlan: true` default is false

`renderer:` this can be specificed globaly `networks:`, per device type
`ethernets:` or for particual device definition.

`dhcp4: true` default is false
`dhcp6: true` default is false
`dhcp4-overrides:`
https://netplan.readthedocs.io/en/latest/netplan-yaml/#dhcp-overrides

`addresses: [192.168.1.2/24]` add static addresses in addition to the one
received through DHCP or RA.

`gateway4:` deprecated, use routes:

`mtu: 1280` maximum transfer unit in bytes, optional

`nameservers: { search: [], addresses: [1.1.1.1] }` is map for DNS servers and
search domains

```
routes:
  - to: 0.0.0.0/0
    via: 10.23.2.1
    metric: 3
```


TODO: Apply network configuration found in the datasource on every boot
https://cloudinit.readthedocs.io/en/latest/explanation/events.html#apply-network-config-every-boot

TODO: User data cannot change an instance’s network configuration.
https://cloudinit.readthedocs.io/en/latest/reference/network-config.html


# Netplan

https://youtu.be/zvbd64ORw8k?t=1414
Generated config
`/run/systemd/network/10-netplan-eth0.network`
can be overriden with native extensions
`/etc/systemd/network/10-netplan-eth0.network.d/override.conf`
for example when you want to add routes to aws instance
```
#/etc/systemd/network/10-netplan-eth0.network.d/override.conf
[Route]
Destination=172.17.0.0/16
Gateway=172.16.1.173
```
so `netplan apply` or creating new instance from the image will use this routes

TODO:
https://blog.slyon.de/2023/07/10/netplan-and-systemd-networkd-on-debian-bookworm/
TODO: https://www.youtube.com/watch?v=2_m6EUo6VOI cloud init talk
TODO: https://www.youtube.com/watch?v=shiIi38cJe4 proxmox template with cloud
init
TODO: https://www.youtube.com/watch?v=1joQfUZQcPg
