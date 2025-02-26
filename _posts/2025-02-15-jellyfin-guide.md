---
layout: post
title: "Configuring external storage devices for Jellyfin"
---
This post serves as a personal guide to running a [Jellyfin](https://jellyfin.org/) server locally using an external hard drive. The following assumes:
- Debian GNU/Linux 12+ (I'm using a Raspberry Pi 4)
- External hard drive (e.g. `/dev/sda1`) mounted at something like `/media/<username>/<hd name>`

## 1. User/group setup
Jellyfin runs as a systemd service with its own user and group, both of which are usually just `jellyfin`. To verify this:
```
$ cat /lib/systemd/system/jellyfin.service | grep "User\|Group"
User = jellyfin
Group = jellyfin
```
To give jellyfin access to your content, you could of course make it the owner and `chmod` away. But you'll probably want to the ability to create new users in the future and given them access as well. Furthermore, manually setting ownership and permissions each time can be cumbersome and error-prone, not to mention you'll need to mount your drive as read-write. We'll see later on how to set permissions automatically at mount-time.

The alternate way to do this is to create a group. New users that need to access media can just be added to that group.

```
$ sudo groupadd media              # create group named media
$ sudo usermod -aG media jellyfin  # add jellyfin to group
```

## 2. Mounting your drive
You can mount your drive manually with `mount(8)`, but the better way is to have it done automatically at boot time.
The simplest way to do this is to add an entry to `/etc/fstab`, which is read by `mount(8)` to determine the overall filesystem structure.
(Note that this approach is quite primitive, and there are more modern tools such as `udev`, which can handle userspace events as well, but for now we'll stick to `fstab`, as it is the de facto option and available on most Unix systems.)

An entry in `/etc/fstab` looks like:
```
<device> <dir> <type> <options> <dump> <fsck>
```
First, to get the `<device>` uuid and filesystem `<type>`, we can search for our drive in `blkid(8)`:
```
$ blkid | grep "/dev/sda1"
/dev/sda1: UUID="93033593-60b5-38ef-8086-e434e74192b0" BLOCK_SIZE="4096" LABEL="TOSHIBA_EXT" TYPE="hfsplus" PARTUUID="fea07a4a-01"
```
Next, `<dir>` is the mount point, e.g. `/media/jamesma100/TOSHIBA_EXT`. For `<options>`, we'll want the following:
- `ro` to mount as read-only
- `uid` and `gid` are the user/group for the device when mounted. We'll want `uid` to be yourself and `gid` to be set to the `media` group we created earlier. To get these ids, you can run
```
$ id
uid=1000(jamesma100) gid=1000(jamesma100) groups=1000(jamesma100),...,1001(media)...
```
- `umask=022` to give read-write permissions to yourself and read-only permissions to others. The permissions will now be `666 - 022 = 644` for files and `777 - 022 - 755` for directories.

The last two fields are filesystem checks - `<dump>` is for backups and `<fsck>` checks for filesystem corruption. I usually set these to off (0) and on (1), respectively.

Putting it all together:
```
$ echo "UUID=93033593-60b5-38ef-8086-e434e74192b0  /media/jamesma100/TOSHIBA_EXT  hfsplus ro,umask=22,uid=1000,gid=1001 0 1">> /etc/fstab
```

To test the newly configured `fstab` without rebooting, run the following:
```
$ systemctl daemon-reload
$ sudo mount -a -v
```
This should remount all your partitions, and the `-v` (verbose) option should tell you which, if any, were unsuccessful.

One last thing to note here: if any of the `-o` options failed, `mount` will not notify you!
As an example, I ran into an issue where the filesystem on my hard drive was mounted as read-only, despite specifiying the `rw` option since I had to make some modifications.

I had to resort to `dmesg(1)` to figure out what was going on.
```
$ dmesg | tail -n 1
[13719.862245] hfsplus: write access to a journaled filesystem is not supported, use the force option at your own risk, mounting read-only.
```
Sneaky sneaky!

## 3. Managing your server
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

## 4. Dealing with crashes
If your hard drive was improperly unmounted for any reason, such a crash, it will usually still remount but prompt the user to perform a consistency check. You should do that and then remount it. Once again, it's necessary to dig through logs; you will not be notified!

```
$ dmesg | tail -n 2
[  137.881974] hfsplus: filesystem was not cleanly unmounted, running fsck.hfsplus is recommended.  leaving read-only.
[  170.342788] hfsplus: Filesystem was not cleanly unmounted, running fsck.hfsplus is recommended.  mounting read-only.
```
Now that we've confirmed the issue, we can use `fsck` to do a repair. Keep in mind that you will need a `fsck` implementation tailored to the filesystem type of your storage device, since `fsck` operates directly on disk-specific data structures. In my case, I need a `fsck.hfsplus` for the HFS+ filesystem, which is included in the [hfsprogs](https://packages.debian.org/sid/hfsprogs) package.

```
$ sudo apt-get -y install hfsprogs
$ sudo fsck.hfsplus /dev/sda1
$ sudo mount -a -v
```
