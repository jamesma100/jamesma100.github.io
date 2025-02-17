---
layout: post
title: "Configuring external storage devices for Jellyfin"
---
This post serves as a personal guide to running a [Jellyfin](https://jellyfin.org/) server locally using an external hard drive. The following assumes:
- Debian GNU/Linux 12+ (I'm using a Raspberry Pi 4)
- External hard drive (e.g. `/dev/sda1`) mounted at something like `/media/<username>/<hd name>`

## Mounting
To mount your drive manually, run:
```
$ sudo mount -o rw /dev/sda1 /media/<username>/<hd name>
```
Make sure to mount as read-write, as we need to modify filesystem permissions later on for Jellyfin to access files on the disk.

Note that if any of the `-o` options failed, `mount` will not notify you!
As an example, I ran into an issue where the filesystem on my hard drive was mounted as read-only, despite specifiying the `rw` option.
```
$ mount | grep "/dev/sda1"
/dev/sda1 on /media/jamesma100/TOSHIBA_EXT type hfsplus (ro,relatime,umask=22,uid=0,gid=0,nls=utf8)
```

I had to resort to `dmesg(1)` to figure out what was going on.
```
$ dmesg | tail -n 1
[13719.862245] hfsplus: write access to a journaled filesystem is not supported, use the force option at your own risk, mounting read-only.
```

### Persisting mount options
Now if your system automounts but you would like to configure the options, you can add an entry to `/etc/fstab`.
First, to get the uuid, filesystem type, and other information about your block devices:

```
$ blkid | grep /dev/sda1
/dev/sda1: UUID="93033593-60b5-38ef-8086-e434e74192b0" BLOCK_SIZE="4096" LABEL="TOSHIBA_EXT" TYPE="hfsplus" PARTUUID="fea07a4a-01"
```

Then, per the output above, add the following to `/etc/fstab` (replacing the mount point with your own). E.g.
```
$ echo "UUID=93033593-60b5-38ef-8086-e434e74192b0  /media/jamesma100/TOSHIBA_EXT  hfsplus force,rw,relatime,umask=22,uid=0,gid=0,nls=utf8" >> /etc/fstab
```

To test the newly configured `fstab` without rebooting, run the following:
```
$ systemctl daemon-reload
$ sudo mount -a -v
```
This should remount all your partitions, and the `-v` (verbose) option should tell you which, if any, were unsuccessful.

### Filesystem corruption
If your hard drive was improperly unmounted for any reason, such a crash, it may mount as read-only (a sort of "maintenance mode", if you will) and prompt the user to perform a consistency check. We need to do this and then remount it as read-write. Once again, it's necessary to dig through logs and the `mount` output to figure this out; the system does not notify us.

```
$ dmesg | tail -n 2
[  137.881974] hfsplus: filesystem was not cleanly unmounted, running fsck.hfsplus is recommended.  leaving read-only.
[  170.342788] hfsplus: Filesystem was not cleanly unmounted, running fsck.hfsplus is recommended.  mounting read-only.
```
Now that we've confirmed the issue, we can use `fsck` to do a repair. Keep in mind that you will need a `fsck` implementation tailored to the filesystem type of your storage device, since `fsck` operates directly on disk-specific data structures. In my case, I need a `fsck.hfsplus` for the HFS+ filesystem, which is included in the [hfsprogs](https://packages.debian.org/sid/hfsprogs) package.

```
$ sudo apt-get -y install hfsprogs
$ sudo fsck.hfsplus /dev/sda1
$ sudo mount -o rw,remount,<other options> /dev/sda1 /media/<username>/<hd name>
```
Now that the disk is repaired and remounted, we can again use `mount` to verify that it is indeed read-write.

## File permissions
Jellyfin runs as a systemd service with its own user and group, both of which are usually just `jellyfin`. To verify this:
```
$ cat /lib/systemd/system/jellyfin.service | grep "User\|Group"
User = jellyfin
Group = jellyfin
```
To add a media library to your instance, you'll need to add the path to your mount point. Jellyfin shouldn't need any write permissions, but it will need execute permissions since it does need to traverse the directory. We can give out these permissions at the group level:
```
$ sudo chown -R :jellyfin /media/<username>
$ sudo chmod -R 755 /media/<username> 
```
Alternatively, you can make a new group, e.g. `media`, add jellyfin to it, and use that instead.
This is probably the correct approach, as you can add multiple users to the `media` group, including yourself.

## Server management
After changing any configuration, you should run
```
$ systemctl daemon-reload
$ systemctl restart jellyfin
```

To stop the server:
```
$ systemctl stop jellyfin
```

Jellyfin logs go under `/var/log/jellyfin`. To view status:
```
$ systemctl status jellyfin
```
