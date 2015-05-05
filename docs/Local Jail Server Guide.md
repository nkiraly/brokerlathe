# Local Jail Server Guide

Local Jail Apollo Server setup notes. This guide is a crash course in setting up a local FreeBSD jail running Apollo for research and development.

# Install Host management ports

Install jail host management packages, catch up any other port-installed packages to be consistent with ports head

```bash
$ portsnap fetch update
$ cd /usr/ports/ports-mgmt/portupgrade/ && make install clean
$ portupgrade -a
$ cd /usr/ports/shells/bash/ && make install clean
$ cd /usr/ports/sysutils/ezjail/ && make install clean
$ cd /usr/ports/sysutils/jailme/ && make install clean
$ portupgrade -a -f
```

# Enable ezjail

```bash
$ vi /etc/rc.conf
ezjail_enable="YES"
```

# Configure a ZFS RAID-Z zpool for all jail storage

## Enable ZFS in rc.conf

```bash
$ vi /etc/rc.conf
zfs_enable="YES"
```

## Create the pool

```bash
$ zpool create bludgeon-storage raidz ada3 ada4
```

## Create filesystem from pool for jails

```bash
$ zfs create bludgeon-storage/jails
$ zfs set checksum=on bludgeon-storage/jails
$ zfs set copies=1 bludgeon-storage/jails
$ zfs set compression=gzip bludgeon-storage/jails
$ zfs set recordsize=16k bludgeon-storage/jails
```

## Mount jail volume for use as /opt/jails

Mount jail filesystem at /opt/jails

```bash
$ mkdir /opt/jails
 
$ zfs set mountpoint=/opt/jails bludgeon-storage/jails
```

Check zpool status and lists

```bash
$ zpool list
$ zpool status
```

If you need to clean up a previously provisioned jail, you will need to umount destroy the jail volume by that name

```bash
[root@bludgeon ~]$ zfs umount /opt/jails/bludgeon-dev00

[root@bludgeon ~]$ zfs destroy bludgeon-storage/jails/bludgeon-dev00
```

To get a list of current zfs volumes and their mount points

```bash
[root@bludgeon ~]$ zfs mount
bludgeon-storage                /bludgeon-storage
bludgeon-storage/jails          /opt/jails
bludgeon-storage/jails/basejail  /opt/jails/basejail
bludgeon-storage/jails/newjail  /opt/jails/newjail
```


# EZJail Setup

## Configure ezjail to use jails zfs volume mount

```bash
$ vi /usr/local/etc/ezjail.conf

ezjail_jaildir=/opt/jails

ezjail_use_zfs="YES"

ezjail_use_zfs_for_jails="YES"

ezjail_jailzfs="bludgeon-storage/jails"
```


## Install basejail

Include ports and source tree

```bash
[root@bludgeon ~]# ezjail-admin install -ps

.....

136316 blocks
Note: a non-standard /etc/make.conf was copied to the template jail in order to get the ports collection running inside jails.

[root@bludgeon ~]# ls -l /opt/jails/
total 9
drwxr-xr-x   9 root  wheel   9 Feb 26 10:57 basejail
drwxr-xr-x   3 root  wheel   3 Feb 26 10:58 flavours
drwxr-xr-x  12 root  wheel  22 Feb 26 10:58 newjail

[root@bludgeon ~]# df -h
Filesystem                                Size    Used   Avail Capacity  Mounted on
/dev/ada0p2                               3.9G    835M    2.7G    23%    /
devfs                                     1.0K    1.0K      0B   100%    /dev
/dev/ada0p3                               1.9G    1.5G    312M    83%    /var
/dev/ada0p5                               1.9G     27M    1.8G     1%    /tmp
/dev/ada1p1                                29G    6.9G     20G    26%    /usr
fdescfs                                   1.0K    1.0K      0B   100%    /dev/fd
procfs                                    4.0K    4.0K      0B   100%    /proc
bludgeon-storage/jails                     39G     45K     39G     0%    /opt/jails
bludgeon-storage/jails/basejail            39G    139M     39G     0%    /opt/jails/basejail
bludgeon-storage/jails/newjail             39G    815K     39G     0%    /opt/jails/newjail
```

If you need to update your basejail, use freebsd-update within the jail with

```bash
$ ezjail-admin update -u
```


# Creating a local Apollo server jail

## Create the jail with ezjail-admin

Create jail in 10.51.4.0/24 space - a fictional development network

```bash
$ ezjail-admin create  apollo-local  10.51.4.133

Configure jail IP and to bind on host interface em0. This may need to change based on you specific environment.

```bash
$ vi /usr/local/etc/ezjail/apollo_local
 
export jail_apollo_local_ip="10.51.4.133"
export jail_apollo_local_interface="em0"
```

## Configure jail rc.conf

Turn off things we know we don't need like syslog and sendmail in the jail

```bash
$ vi /opt/jails/apollo-local/etc/rc.conf

syslogd_enable="NO"
sendmail_enable="NONE"
cron_enable="YES"
```
 
## Configure jail DNS resolution

Set DNS resolution to be done the same as the jail host

```bash
$ cp /etc/resolv.conf /opt/jails/apollo-local/etc/
```

## Configure jail time zone and test jail binary execution

This tests that you can start and execute binaries in the jail, while also configuring and confirming jail timezone

```bash
$ cp /usr/share/zoneinfo/America/New_York /opt/jails/apollo-local/etc/localtime
 
$ /usr/local/etc/rc.d/ezjail start apollo-local
 
$ sudo -H jailme 1 date
```


# Jail Ports Management

pkg ng allows you to do jail ports management from the host. This greatly streamlines packaging.

Remember you need to have ezjail-admin update the basejail ports tree and manage the non-standard /etc/make.conf

Update pkg and then packages in the jail
```bash
[root@bludgeon ~]$ jls
   JID  IP Address      Hostname                      Path
     1  10.51.4.133     apollo-local                  /opt/jails/apollo-local

[root@bludgeon ~]$ sudo pkg -j 1 update
Updating FreeBSD repository catalogue...
[apollo-local] Fetching meta.txz: 100%    944 B   0.9kB/s    00:01
[apollo-local] Fetching packagesite.txz: 100%    5 MiB   5.3MB/s    00:01
Processing entries: 100%
FreeBSD repository update completed. 23912 packages processed.
```

## Installing Packages in the jail

Now you have your dependencies, you can provision the jail with the provision_broker.yml playbook.

```bash
[root@bludgeon ~]# sudo pkg -j 1 install java/openjdk7
Updating FreeBSD repository catalogue...
FreeBSD repository is up-to-date.
All repositories are up-to-date.
The following 3 package(s) will be affected (of 0 checked):

New packages to be INSTALLED:
        bash: 4.3.33
        indexinfo: 0.2.3
        gettext-runtime: 0.19.4

The process will require 7 MiB more space.
1 MiB to be downloaded.

Proceed with this action? [y/N]:
```

```bash
[root@bludgeon ~]# sudo pkg -j 1 install java/openjdk7
Updating FreeBSD repository catalogue...
FreeBSD repository is up-to-date.
All repositories are up-to-date.
The following 32 package(s) will be affected (of 0 checked):

New packages to be INSTALLED:
        openjdk: 7.80.15,1
        libXtst: 1.2.2_3
        recordproto: 1.14.2
        libXi: 1.7.4_1,1
        xproto: 7.0.27
        libXfixes: 5.0.1_3
        libX11: 1.6.2_3,1
        libxcb: 1.11_1
        libXdmcp: 1.1.2
        libXau: 1.0.8_3
        libxml2: 2.9.2_2
        libpthread-stubs: 0.3_6
        kbproto: 1.0.6
        fixesproto: 5.0
        libXext: 1.3.3_1,1
        xextproto: 7.3.0
        expat: 2.1.0_2
        inputproto: 2.3.1
        libXrender: 0.9.8_3
        renderproto: 0.11.1
        libXt: 1.1.4_3,1
        libSM: 1.2.2_3,1
        libICE: 1.0.9_1,1
        fontconfig: 2.11.1,1
        freetype2: 2.5.5
        dejavu: 2.34_6
        mkfontscale: 1.1.2
        libfontenc: 1.1.2_3
        mkfontdir: 1.0.7
        javavmwrapper: 2.5
        java-zoneinfo: 2015.b
        alsa-lib: 1.0.29

The process will require 187 MiB more space.
57 MiB to be downloaded.

Proceed with this action? [y/N]:
```


## Building custom packages

If you need to, you can build custom packages to install in your jails with
```bash
[root@bludgeon /usr/ports/sysutils/ansible]# make package
===>  Building package for ansible-1.9.0.1

[root@bludgeon /usr/ports/sysutils/ansible]# ls -lah work/pkg
total 2512
drwxr-xr-x  2 root  wheel   512B May  5 16:46 .
drwxr-xr-x  6 root  wheel   512B May  5 16:46 ..
-rw-r--r--  1 root  wheel   1.2M May  5 16:46 ansible-1.9.0.1.txz
```

You can use make-config to do all your config options up front so you can just let the build chain run.

```bash
[root@bludgeon /usr/ports/shells/bash]# make config-recursive
===> Setting user-specified options for bash-4.3.33 and dependencies
```


# References
https://erdgeist.org/arts/software/ezjail/
https://www.freebsd.org/doc/en/books/handbook/ports-using.html
