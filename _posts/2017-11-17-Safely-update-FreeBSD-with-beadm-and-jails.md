This procedure allows to safely and fully update a running FreeBSD installation and switching to the updated version with a single reboot.
  
### steps involved
 
- snapshot the whole zroot (just in case and because it's cheap with ZFS anyways)
- create BE for the new version (e.g. 11.1-RELEASE)
- mount BE + /usr/src (snapshot it first)
- start a jail within that mountpoint
- fetch & install updates
- rebuild & install kernel if needed
- activate new BE
- reboot
- update packages if necessary
 

### step-by-step

 

```shell
host# zfs snapshot -r zroot@11.0-RELEASE
host# beadm create 11.1-RELEASE
host# beadm mount 11.1-RELEASE /usr/jails/upgrade
host# zfs umount zroot/usr/src
host# zfs snapshot zroot/usr/src@11.0-RELEASE
host# zfs mount -o mountpoint=/usr/jails/upgrade/usr/src zroot/usr/src
host# jail -c path=/usr/jails/upgrade mount.devfs host.hostname=upgrade ip4.addr=vlan4|10.50.51.90/24 command=/bin/tcsh
upgrade# freebsd-update -r 11.1-RELEASE upgrade
upgrade# freebsd-update install
# (restart jail?)
upgrade# freebsd-update install --currently-running 11.1-RELEASE
# (rebuild & install custom kernel if necessary)
upgrade# cd /usr/src
upgrade# make -j8 buildkernel KERNCONF=VIMAGE && make installkernel
upgrade# exit
host# zfs umount zroot/usr/src
host# beadm umount 11.1-RELEASE
host# beadm activate 11.1-RELASE
# (reboot host)
(host# pkg-static install -f pkg)  # for major release upgrades
host# pkg upgrade
```

 
 
If the reboot fails, just select the previous BE. Following this procedure, BEs will always be named after the version they have installed and the 'default' BE will be the oldest one back from initial installation.
 
### caveats
 
The GENERIC kernel still needs to be in /boot for the update. If no longer present, it can be rebuilt from source within the jail (KERNCONF=GENERIC) or downloaded from <https://download.freebsd.org/ftp/releases/>.
 
 
In case of fire, don't forget to rollback /usr/src
 
 
_Thanks_  
_sko_
 
----
