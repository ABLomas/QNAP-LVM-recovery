## Why this tool?
QNAP storage has "thin-volume" LVM feature, which is not working properly and can lead to metadata (and data, in some cases) corruption (most likely - when volumes are full and system is doing automated snapshots). It is not merged into mainline kernel.

This VM image can help you to recover data from corrupted thin LVM volume. Well, at least it worked in two cases i had, maybe it will work for you. It allows to repair and access thin (and thick) provisioned LVM volumes (QNAP edition).

## Why separate VM image?
QNAP sources (modified GPL stuff) were released, kind of, but when i was trying to build kernel - i had so many missing references, undefined macros (linked into some non-existing includes), etc - in other words, kernel does not build. Modified LVM binaries work, but without kernel support they are useless. I did this VM long ago so do not remember all steps i did, but in short - i extracted old QNAP recovery image (old for some serious reason - probably changed kernel compression), applied patches, swapped LVM binaries from new qnap release and installed few binaries. Also used hints from https://github.com/mkke/qnap-recovery - something like that. 

I did not found any commercial tool capable of fixing (!) corrupted thin LVM. Even when some claim that they should be able to read data - they could do that from "clean" LVM volume (which may be useful if you had partial disk failure, recovered with corrupted data on disk, QNAP storage refuses to use full volume, but metadata is intact), but any corruption - and all tools become useless (well, sure they can do full data scan, looking for file patterns, but this is not what i want).

## Steps to recover data
(at least i would do this way, many things may differ depending on situation. I also did that long time ago, so do not remember all details):
- Take out disks from QNAP storage. This MAY be not required if you are sure that array is in good shape (see steps below). Usually it's better to work with data copy, even in read-only mode
- Attach them to linux system (have kpartx, lvm2, mdadm packages ready), make disk images (again, optional - i would not work with original client data - maybe your data is not worth it) using dd/ddrescue
- Detach original disks (they are your master copy now, keep them in safe place). Setup RAID images (maybe not required if you have single disk, there are such devices with thin provisioning and no RAID? I don't know any):

       kpartx -a -v /fimg/m2_1.iso
       kpartx -a -v /fimg/m2_2.iso
       kpartx -a -v /fimg/sata_1.iso
       kpartx -a -v /fimg/sata_2.iso
       kpartx -a -v /fimg/sata_3.iso
       # <...> repeat as needed for all disk images
       # note loop names in previous command output
       mdadm --assemble --readonly --force /dev/md0 /dev/mapper/loop[12345]p3
       # Your loop names may differ, device count also
       # p3 is RAID (with parity) with user data on QNAP devices i saw, but may differ
       # check with `fdisk -l /dev/mapper/loop1` for example and notice which partition is biggest
       # or see header/disk name in `hexdump -C /dev/mapper/loop5p3 | head`
       # or just use mdadm: `mdadm --examine /dev/mapper/loop[12345]p3`
       
- inspect previous command output and RAID health (`cat /proc/mdstat`), fix issues if there are any
- Again, optional, depends on data cost/data amount - make full data image: `ddrescue /dev/md126 /lots_of_space/raid_content.iso` (name md126 may differ)
So, you got your data in single file, this is good thing, but if you try to use it on typical linux system, for example using `lvscan` - you will probably get output:
    
        WARNING: Unrecognised segment type tier-thin-pool
        <...>
        Cannot process volume group vg1
        	

 If you play with metadata and force reading data, populating snapshot states and VG's/LV's in image. Then you still can't get your volume/snapshots recognized:
    
        # lvchange -ay -K -v vg1/snap10050
            Using logical volume(s) on command line.
            Activating logical volume "snap10050" exclusively.
            activation/volume_list configuration setting not defined: Checking only host tags for vg1/snap10050.
            dev_open(/dev/loop0) called while suspended
            Creating vg1-tp1_tmeta
            Loading vg1-tp1_tmeta table (252:0)
            Resuming vg1-tp1_tmeta (252:0)
            Creating vg1-tp1_tierdata_0
            Loading vg1-tp1_tierdata_0 table (252:1)
            Resuming vg1-tp1_tierdata_0 (252:1)
            Creating vg1-tp1_tierdata_1
            Loading vg1-tp1_tierdata_1 table (252:2)
            Resuming vg1-tp1_tierdata_1 (252:2)
            Creating vg1-tp1_tierdata_2
            Loading vg1-tp1_tierdata_2 table (252:3)
            Resuming vg1-tp1_tierdata_2 (252:3)
            Executing: /sbin/thin_check -q --skip-mappings /dev/mapper/vg1-tp1_tmeta
            Creating vg1-tp1-tpool
            Loading vg1-tp1-tpool table (252:4)
        [ 6623.501490] device-mapper: thin metadata: __create_persistent_data_objects: block manger get correctly
            Resuming vg1-tp1-tpool (252:4)
        [ 6623.931462] device-mapper: thin: 252:4: growing the data device from 4913144 to 6594936 blocks
        [ 6624.256141] device-mapper: btree spine: node_check failed: blocknr 256 != wanted 41016
        [ 6624.258780] device-mapper: block manager: btree_node validator check failed for block 41016
        [ 6624.261257] BUG: unable to handle kernel paging request at ffffc90001a57440
        [ 6624.262211] IP: [<ffffffff813ad1b3>] bitmap_set+0x63/0x70
        [ 6624.262211] PGD 3dc36067 PUD 3dc37067 PMD 2368c067 PTE 0
        <...>
        [ 6624.262211] RIP  [<ffffffff813ad1b3>] bitmap_set+0x63/0x70
        [ 6624.262211]  RSP <ffff88003a977a58>
        [ 6624.262211] CR2: ffffc90001a57440
        [ 6624.262211] ---[ end trace c1e4e65f25ea35f7 ]---
        Killed
          
 So, kernel module cannot understand metadata. Here we need modified image.
  
- Install Hyper-V (it is probably in W10 installations already? If not, you can add it using "add-remove windows components"), start mmc, add Hyper-V snapin, import unpacked image from "Releases" tab, go to settings. Add disks and change settings as required:
  - attach RAID image from steps above (storage with iso file - we need raw content)
  - attach additional storage - you will need to copy data out
  - adjust serial port settings (see serial settings - QNAP has no monitor output, we use serial link - in image it is set as `\\.\pipe\com15` - i maybe also used tool named "COMpipe.exe" - it's free on internet)
- Start VM. You won't see anything if you use "Connect" (no video out) but you can connect to serial port (by default COM15) using putty or any other tool you like.
- Log in using `admin/admin` username/password, run `/etc/init.d/init_check.sh`, lsblk, check that all attached devices visible. Mount additional space if required
- I do not remember all details, but most likely i extended ISO image (dd few hundred megabytes from /dev/zero, merge it (to end) with raid image (using cat, for example), extend PV inside - or any other operation you know). Skip this step for a while, try with steps below, but most likely it will fail due to missing disk space, so you will need to extend it.
- Go with recovery, in my case:


      # /etc/init.d/init_check.sh
      mkdir /1
      mount -t ext4 /dev/sdb2 /1
      # now in /1 we have RAID data
      losetup /dev/loop0 /1/raid_content.iso
      ln -s /dev/loop0 /dev/md1
      # I do not remember details - why - but IIRC QNAP tools refused to work with loop images, but symlink to "md" device was ok
      pvscan --cache /dev/md1
      pvs
      lvchange -a y vg1/lv1 
      # <...> crash
      lvconvert --repair vg1/tp1
        WARNING: Sum of all thin volume sizes (71.75 TiB) exceeds the size of thin pools and the size of whole volume group (3.31 TiB)!
        For thin pool auto extension activation/thin_pool_autoextend_threshold should be below 100.
        WARNING: If everything works, remove "vg1/tp1_meta0".
        WARNING: Use pvmove command to move "vg1/tp1_tmeta" on the best fitting PV.
		
- Check if volumes visible (`lvdisplay`), activate (but IIRC not required), check status on it, if LVM is happy (see `dmesg` also) - do fsck on logical volume. Expect to see many errors, but fsck should be able to fix them
- Copy your data out to new storage
  
## What to expect:
In both cases i had - some data was lost, this is expected, but data loss was not huge. First one this was 0.08%, second case was a bit worse with ~0.34% of data lost. This is also counting stuff from lost+found (after fsck) - so some files lost their names (without it dataloss can be counted at approx 0.11% and 0.45% of data).
This looks not bad, but when you have terabytes of data - can hurt, so pre-fsck image could be used to pull even more data (`icat`, data read from unclean (before fixing!) filesystem, manually cutting overlapping data areas, etc) - but in my case clients did not noticed any important data loss (lost temporary files, few email attachments which can be recovered) and i decided not to dive deeper.

For file checking scripts (in lost+found) see MongoDB recovery repo, but in short - this is simple script looping trough files, using magic bytes (to rename file) and then file-specific content checker (to verify file integrity). In most cases you won't need them.
