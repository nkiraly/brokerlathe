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


# Configure a ZFS RAID-Z zpool of filesystems for each node's db storage

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

## Check zpool status and lists

```bash
$ zpool list
$ zpool status
```



# EZ Jail Setup

## Configure ezjail to use jails zfs volume mount

```bash
$ vi /usr/local/etc/ezjail.conf

ezjail_jaildir=/opt/jails

ezjail_use_zfs="YES"

ezjail_use_zfs_for_jails="YES"

ezjail_jailzfs="bludgeon-storage/jails"
```


## Install basejail

```bash
[root@bludgeon ~]# ezjail-admin install

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


# Creating a dev jail

## Create a dev jail with ezjail-admin

```bash
$ ezjail-admin create  bludgeon-dev00  10.50.2.91

Configure jail IPs and talk on host main interface

```bash
$ vi /usr/local/etc/ezjail/bludgeon_dev00
 
export jail_bludgeon_dev00_ip="10.50.2.91"
export jail_bludgeon_dev00_interface="em1"
```

## Configure dev jail rc.conf

```bash
$ vi /opt/jails/bludgeon-dev00/etc/rc.conf

syslogd_enable="NO"
sendmail_enable="NONE"
cron_enable="YES"
```
 
## Configure jail DNS resolution

Set DNS resolution to be done as is on the host

```bash
$ cp /etc/resolv.conf /opt/jails/bludgeon-dev00/etc/
```

## Configure jail time zone and test jail binary execution

This tests that you can start and execute binaries in the jail, while also configuring and confirming jail timezone

```bash
$ cp /usr/share/zoneinfo/America/New_York /opt/jails/bludgeon-dev00/etc/localtime
 
$ sudo /usr/local/etc/rc.d/ezjail start bludgeon-dev00
 
$ sudo -H jailme 1 date
```

