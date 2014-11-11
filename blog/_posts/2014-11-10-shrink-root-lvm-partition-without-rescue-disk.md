---
layout: post
title: 'Resize root LVM partition'
meta:
 keywords: howto, resize, lvm, root, partition, rootfs
---

I like to use LVM on my personal laptop to avoid the crazy partition sizing step. When I set up my system I just create some main partitions (`swap`, `/` and `/home`) then I grow them as I need more space.

Lately I installed docker and instead of creating a logical volume for docker data in `/var/data/docker` I lazily growed my `/` partition. In fact, at the time I knew it was a mistake. Few weeks later I got sick of it.

Here is how to resize an unmountable parition, e.g. your system root partition. 

1. create a snapshot SNAP_ROOT of volume ROOT
2. shrink the filesystem on SNAP_ROOT
3. ask LVM to merge SNAP_ROOT to ROOT, ie. replace the volume with the original filesystem by the shrinked one
4. reboot so LVM can do the merge
5. reduce ROOT volume size with option `--resizefs`

⚠ First you should remount the partition in read-only mode. I wasn't able to do it because of all the stuff running and I decided to take the risk to continue. If you do so, **you're responsible**.

### the short way

    lvcreate -s ssd/root -L 40G -n root_backup
    e2fsck -f /dev/mapper/ssd-root_backup \
        && resize2fs -p -M /dev/mapper/ssd-root_backup
    lvconvert --merge ssd/root_backup
    reboot
    lvresize -L 20G -r ssd/root

### the long way

#### 1. create a snapshot SNAP_ROOT

Create a snapshot of the partition, I used a size of 40G (same size of the original partition) but a such value shouldn't be necessary.

`lvcreate -s ssd/root -L 40G -n root_backup`

Running `lvdisplay` shows the original volume and its snapshot. We can see:

* original volume size is 40GB
* snapshot as _Copy-On-Write-table size_ of 40GiB
* 0% of the _COW-table_ is used for now

`lvdisplay ssd/root ssd/root_backup`

    --- Logical volume ---
    LV Path                /dev/ssd/root
    LV Name                root
    VG Name                ssd
    …
    LV snapshot status     source of
                           root_backup [active]
    …
    LV Size                40,00 GiB
    …

    --- Logical volume ---
    LV Path                /dev/ssd/root_backup
    LV Name                root_backup
    VG Name                ssd
    …
    LV snapshot status     active destination for root
    LV Status              available
    …
    COW-table size         40,00 GiB
    COW-table LE           10240
    Allocated to snapshot  0,00%
    …

#### 2. shrink the filesystem on SNAP_ROOT

Firstly, a filesystem check must be forced:

`e2fsck -f /dev/mapper/ssd-root_backup`

    e2fsck 1.42.12 (29-Aug-2014)
    Passe 1 : vérification des i-noeuds, des blocs et des tailles
    Passe 2 : vérification de la structure des répertoires
    Passe 3 : vérification de la connectivité des répertoires
    Passe 4 : vérification des compteurs de référence
    Passe 5 : vérification de l'information du sommaire de groupe
    root : 290516/2621440 fichiers (0.2% non contigüs), 4227390/10485760 blocs

Then we are able to resize it:

`resize2fs -p -M /dev/mapper/ssd-root_backup`

    resize2fs 1.42.12 (29-Aug-2014)
    En train de redimensionner le système de fichiers sur /dev/mapper/ssd-root_backup à 4527614 (4k) blocs.
    The filesystem on /dev/mapper/ssd-root_backup is now 4527614 (4k) blocks long.

Shrinking the filesystem may cause a lot of changes to the volume. During this operation the allocated space of the _COW-table_ will increase.
We can run `lvdisplay` to observe how many space the resize operation taken from the snapshot COW-table.

`lvdisplay ssd/root_backup`

    --- Logical volume ---
    LV Path                /dev/ssd/root_backup
    LV Name                root_backup
    ...
    Allocated to snapshot  17,13%
    ...

#### 3. merge SNAP_ROOT to ROOT

`lvconvert --merge ssd/root_backup`

    Logical volume ssd/root contains a filesystem in use.
    Can't merge over open origin volume.
    Merging of snapshot ssd/root_backup will occur on next activation of ssd/root.

#### 4. reboot

LVM can't merge a snapshot to an active partition: the partition has te be deactivated. This is done by rebooting the system.

`reboot`

#### 5. reduce ROOT volume

Let's see the new partition size we have:

`df -h`

    Sys. de fichiers       Taille Utilisé Dispo Uti% Monté sur
    /dev/mapper/ssd-root      17G     16G  742M  96% /

`lvdisplay ssd/root`

    --- Logical volume ---
    LV Path                /dev/ssd/root
    LV Name                root
    VG Name                ssd
    …
    LV Size                40,00 GiB

The filesystem has a size of 17G, but the volume size is always 40G. We must use `lvm` to resize the volume. We use option `-r`/`--resizefs` which tell LVM to resize the underlying filesystem to the new size of the volume.

`lvresize -L 20G -r ssd/root`

    resize2fs 1.42.12 (29-Aug-2014)
    Le système de fichiers de /dev/mapper/ssd-root est monté sur / ; le changement de taille doit être effectué en ligne
    old_desc_blocks = 2, new_desc_blocks = 2
    The filesystem on /dev/mapper/ssd-root is now 5242880 (4k) blocks long.
    Size of logical volume ssd/root changed from 40,00 GiB (10240 extents) to 20,00 GiB (5120 extents).
    Logical volume root successfully resized


Now our partition and our volume size are reduced:

`df -h`

    Sys. de fichiers       Taille Utilisé Dispo Uti% Monté sur
    /dev/mapper/ssd-root      20G     16G  3,4G  83% /

`lvdisplay ssd/root`

    --- Logical volume ---
    LV Path                /dev/ssd/root
    LV Name                root
    VG Name                ssd
    …
    LV Size                20,00 GiB