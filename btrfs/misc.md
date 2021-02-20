# Mount options

For some reason (actually lack of implementation), all subvolumes in the same partition will be mounted with the mount options of the first such subvolume, and the rest will be ignored:

```bash
$ grep subvol /etc/fstab
UUID=7a97a9bf-7bf5-450e-a6c5-3b37e8979ba0 /       btrfs   subvolid=272,defaults,noatime,compress=lzo   0   1
UUID=7a97a9bf-7bf5-450e-a6c5-3b37e8979ba0 /home   btrfs   subvolid=257,defaults,noatime,compress=zstd  0   1

$ mount | grep @
/dev/nvme0n1p3 on / type btrfs (rw,noatime,compress=lzo,ssd,space_cache,subvolid=276,subvol=/@root-mint)
/dev/nvme0n1p3 on /home type btrfs (rw,noatime,compress=lzo,ssd,space_cache,subvolid=257,subvol=/@home)
```

# Subvolume creation

Any configuration is fine, but the following is the one I implement. I create two directories, one for subvolumes, one for snapshots:

```bash
$ mkdir /{.subvolumes,.snapshots}
```

Then say I want to create a specific subvolume for the "Documents" folder of user "isilanes" (because I will give it specific compression settings, see below, or because I want to give it a space quota so it doesn't fill the HOME). I would do the following:

```bash
$ mkdir /.subvolumes/@isilanes
$ btrfs subvolume create /.subvolumes/@isilanes/Documents
$ vi /etc/fstab
$ mkdir ~isilanes/Documents
$ mount ~isilanes/Documents/
$ chown isilanes.isilanes ~isilanes/Documents
```

Above, editing `/etc/fstab`, I add the following line:

```
UUID=7a97a9bf-7bf5-450e-a6c5-3b37e8979ba0 /home/isilanes/Documents      btrfs   subvolid=281,defaults,noatime 0   1
```

The value of `subvolid` can be obtained from:

```bash
$ btrs subvolume list /
```

# Compression

Due to all subvolumes in a partition being mounted with the same mount options (see above), I recommend mounting with option `compress=lzo`, as it is the fastest compression method. This way both `/` and `/home` will be mounted with this compression, in general:

```bash
$ grep subvol /etc/fstab
UUID=7a97a9bf-7bf5-450e-a6c5-3b37e8979ba0 /       btrfs   subvolid=272,defaults,noatime,compress=lzo   0   1
```

Compression can be set on a subvolume after mounting, but unfortunately will not work (for me) for "root" subvolumes, such as `@` and `@home`. Those keep the compression settings in the mount options. Fortunately, other subvolumes can be configured independently. To set level 3 ZSTD compression on the "Documents" subvolume created above, we would do the following:

```bash
$ btrfs property set /home/isilanes/Documents compression zstd:3
```

Once compression is set on a subvolume it will be persistent until changed. It can be unset with:

```bash
$ btrfs property set /home/isilanes/Documents compression ""
```

Anything copied or created into that subvolume will be compressed (if Btrfs considers it worthwhile, see documentation). We can check that compression is being used with `compsize`:

```bash
$ apt install btrfs-compsize
$ compsize /home/isilane/Documents
Processed 4452 files, 2602 regular extents (2602 refs), 1716 inline.
Type       Perc     Disk Usage   Uncompressed Referenced  
TOTAL       97%      1.1G         1.2G         1.2G       
none       100%      1.1G         1.1G         1.1G       
zstd        33%       15M          44M          44M
```

In the example above, only 44 MB worth of data have been compressed with ZSTD (to a third of its size), while 1.1 GB have been considered uncompressable (probably because they were either binaries or already compressed).
