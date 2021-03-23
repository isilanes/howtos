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

# Disk quotas

Btrfs can impose disk quotas on subvolumes, effectively limiting the space they can grow to arbitrarily. You can enable quotas for a subvolume (in my tests, this command enabled quotas for all subvolumes within the same partition), with:

```bash
$ btrfs quota enable <mounted-subvolume-path>
```

You should run this command the first time:

```bash
$ btrfs quota rescan <mounted-subvolume-path>
```

After that, you can check the status of the quotas with:

```bash
$ btrfs qgroup show -reF /
```

The above will show a list of all subvolumes in `/`, and their size (`rfer` column), and quota (`max_rfer` column).

To set a quota on a subvolume, for example `/home` to 20 GB:

```bash
$ btrfs qgroup limit 20G /home
```

# Free space

There are a number of commands to check the disk usage of a Btrfs filesystem[1]. Assuming we want to check a filesystem mounted at `/btrfs`:

```bash
$ btrfs filesystem show
$ btrfs filesystem df /btrfs
$ btrfs filesystem usage /btrfs
```

The main thing to consider in order to understand disk space usage by a Btrfs system is that Btrfs allocates space in chunks before putting data into them. Depending on how full the chunks are, the total allocated space can be significantly larger than the data inside it. For example, in a 500 GB partition, we could have 100 GB of data on it, and Btrfs might have allocated 120 GB. This means that the disk theoretically has 400 GB of free space, but actually only 380 GB of data can be added to it. Tools such as `df` will tell us that the full 400 GB are free, while 20 of those are allocated (unusable) space.

One way to close the gap between free and actually usable space is to run the balance Btrfs tool[1]. The syntax is:

```bash
$ btrfs balance start -dusage=x /
$ btrfs balance start -musage=x /
```

where `x` is a number from 0 to 100. The first command acts upon data, and the second one upon metadata. Each one analyzes the disk chunks where Btrfs stores data, and relocates it if the occupation of the chunk is below `x`. This way data is stored more efficiently in chunks, and allocated space is closer to actual data size.

[1] https://ohthehugemanatee.org/blog/2019/02/11/btrfs-out-of-space-emergency-response/