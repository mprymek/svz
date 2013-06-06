SVZ
===
Script-set for easy testing of server clusters in VirtualBox. You can use it as 
a much simpler, dependency-free, lightning-speed-fast alternative to Vagrant.

This version uses Salt, ZFS and CentOS guests but you can easily adapt it to 
other technologies.

Requisites
----------
1. ZFS is used for lightning-speed cloning of virtual machines. It's available
for FreeBSD (native), Solaris derivatives (native), MacOS (MacZFS, ZEVO), Linux (zfsonlinux).
Install it before using this scripts. 

2. Base virtual machine image[s]. For CentOS, you can look at misc/prepare_centos script.
If you want to use other guest systems, you must prepare it on your own and also
adapt update-all script (it edits CentOS-specific /etc/sysconfig/network file).

Description
-----------
All data are kept in one "ZFS root", e.g. tank/VMs.
Base images are kept in separated datasets under it.

Only disk image is to be present in the base image directory. Other VM settings are retrieved from
settings file.

Base images must be ZFS *snapshots* and virtual machines are cloned from this snapshot as datasets
$zfs_root/VMNAME_hdd

up-all script clones the base image, creates all the VMs and powers them up.
update-all script sets the machines' hostnames and updates the machines state using Salt.
sshvm script can be used to connect to the particular machine or run commands on it.
destroy-all script destroys all the machines.

*Thanks to ZFS, cloning of the machines is super-fast: a new machine is prepared in fractions of second.
The cloning time does NOT depend on the size of the base image. Images of any size are cloned in the same
amount of time. (Unlike in e.g. Vagrant where images are simply copied by default)*

Settings
--------
VMs definition in the `settings` file MUST have this form:

vmX_name=NAME
vmX_base=BASE_IMG_NAME
vmX_ram=RAM
vmX_ssh_port=SSH_PORT

X is the number of the machine. Machines MUST be numbered from 1 succesivelly. NAME is the
name of the machine. BASE_IMG_NAME is the name of the base image snapshot to be cloned. 
RAM is the amount of RAM the machine should have. SSH_PORT is the port which will be
opened on the HOST localhost to communicate with the particular machine. Hence the port
must be unique and must not be used by any application on the host machine.

Salt
----
If you prepare your base image using misc/prepare_centos script:
1. the vbox directory is mounted as /vbox in all the machines
2. the vbox/salt/etc/minion file is symlinked to /etc/salt/minion
3. in this minion config file, file_roots is set to /vbox/salt/states and pillar_roots to /vbox/salt/pillar

Hence, you can create your salt configuration in vbox/salt/states/top.sls and
it will be automatically forced by update-all script.

Preparation
-----------
1. adapt settings file

```shell
# cat settings
vm1_name=test1
vm1_base=base_centos6_hdd@base
vm1_ram=400
vm1_ssh_port=2223

# "root dataset"
zfs_root="tank/VMs"
# where the root dataset is mounted
root="/Volumes/$zfs_root"
# domain to be set on the machines
lab_domain="example.com"
```
2. prepare base image
3. create zfs datasets and base-image snapshot
```shell
# . settings
# zfs create $zfs_root
# zfs create $zfs_root/base_centos6_hdd
# cp /path/to/base_image.vmdk $root/base_centos6_hdd/box-disk1.vmdk
# zfs snapshot $zfs_root/base_centos6_hdd@base
```
4. insert Salt state configuration to vbox/salt/states/top.sls

Usage
-----
1. bring the machines up
```shell
#time ./up-all 
Creating VM test1
Virtual machine 'test1' is created and registered.
UUID: 314da040-6ceb-4934-a4d5-ae4fac0f532f
Settings file: '/Volumes/tank/VMs/test1/test1.vbox'

real  0m0.754s
user  0m0.158s
sys   0m0.130s
```
The machine is created and powered up in 0.754s.

2. wait for machines to boot up and force the desired state
```shell
# ./update-all
Updating config of test1
local:
----------
    State: - file
    Name:      /root/.inputrc
    Function:  managed
        Result:    True
        Comment:   File /root/.inputrc updated
        Changes:   diff: New file
```

3. ssh into particular machine
```shell
# ./sshvm test1
```

4. tear everything down
```shell
# ./destroy-all
```
