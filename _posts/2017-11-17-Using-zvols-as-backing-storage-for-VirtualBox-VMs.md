Using ZFS volumes as backing storage for VirtualBox VMs needs some additional work, due to some assumptions VB is making and the way geom is handling them.
 
 
### ZFS specific settings
 
- set "volmode=dev" to expose the zvol as raw block storage and prevent e.g. geom or automounter to try accessing slices/filesystems the VM might create.
- use volblocksize >64k to increase performance
- create (or clone) the dataset as a child of the users /usr/home dataset to simplify delegation of zfs operations on the volume (snapshots, rollback...)
 

### host configuration
 
To allow users to access the zvol, we need to transfer ownership of the volumes:
 
 
```

[/etc/devfs.conf]
perm	/dev/zvol/zroot/usr/home/<user>/vbox			0770
own	/dev/zvol/zroot/usr/home/<user>/vbox			<user>:<user>
own	/dev/zvol/zroot/usr/home/<user>/vbox/<volname>		<user>:<user>


[/etc/devfs.rules]
add path 'zvol/zroot/usr/home/<user>	mode 0700 user <user> group <user>

```
 
 
### create VirtualBox disk
 
VirtualBox needs a vdmk-file that points to the raw device. This file has to be created by the user!
 
```
virtualbox internalcommands createrawvmdk -filename <file>.vmdk -rawdisk /dev/zvol/zroot/usr/home/<user>/vbox/<volname>
```
 
 
This vmdk can now be used by VirutalBox as a virutal disk.
 
 
_Thanks_  
_sko_
 
----
