## Introduction

## Authors
luksipc was written by Johannes Bauer <JohannesBauer@gmx.de>. Please report all
bugs directly to his email address or file a issue at GitHub (URL is below). If
you do not wish to be named in the ChangeLog or this README file, please tell
me and I'll omit your name. Inversely, if I forgot to include you in this list
and you would like to appear, please drop me a note and I'll fix it.

There are several contributors to the project:

  - Eric Murray (cryptsetup status issue)
  - Christian Pulvermacher (cryptsetup status issue)
  - John Morrissey (large header issue)

The current version is maintained at GitHub at the moment: https://github.com/johndoe31415/luksipc

The project documentation can be found at: https://johndoe31415.github.io/luksipc

The projects main page is hosted at: https://johannes-bauer.com/linux/luksipc/

Please send issues and pull requests to GitHub if you would like to contribute.
I do have a horrible latency sometimes but I'll try to do my best, promise.


## Disclaimer
If you use luksipc and it bricks your disk and destroys all your data then
that's your fault, not mine. luksips comes without any warranty (neither
expressed nor implied). Please have a backup for really, really important data.


## Compiling
luksipc has no external dependencies, it should compile just fine if you have a
recent Linux distribution with GNU make and gcc installed. Just type:
```
$ make
```


That's it. At runtime, it needs access to the cryptsetup and dmsetup tools in
the PATH.


## luksipc vs. cryptsetup-reencrypt
luksipc has nothing to do with cryptsetup-reencrypt. It was simply created
because at the time that luksipc was written, cryptsetup-reencrypt hadn't been
written just yet. On modern systems, cryptsetup-reencrypt is already shipped
with the LUKS tools and it's also the supported tool by LUKS upstream.
Therefore, to be frank, it seems like the better choice to use nowadays. I've
never used cryptsetup-reencrypt myself so far, but will probably try it out on
the next best occasion (then I can also update this README with a more
insightful comment instead of just clueless jibber jabber).

## Usage
The following section will now cover the basic usage of luksipc. There are two
distinct cases, one is the conversion of a unencrypted (plain) volume to LUKS
and the other is the conversion of an encrypted (LUKS) volume to another LUKS
volume. Both will need the same preparation steps before performing them.


## Checklist
If you skip over everything else, **please** at least make sure you do these
steps before starting a conversion:

  - Resized file system size, shrunk size by at least 10 MiB
  - Unmounted file system
  - Laptop is connected to A/C power (if applicable)



## Preparation
The first thing you need to do is resize your file system to accomodate for the
fact that the device is going to be a tiny bit smaller in the end (due to the
LUKS header). The LUKS header size is usually 2048 kiB (it was 1028 kiB for
previous versions of cryptsetup), but you can safely decrease the file system
size by more (like 100 MiB) to be on the safe side.  If you decrease the size
too much you have no drawbacks (and you can easily increase after the
conversion has been performed).

*WARNING*
  Do not forget to shrink the file system before conversion

luksipc has no means of detecting wheter or not you have performed this step
and will not warn you if you haven't (it has no knowledge of the underlying
file system). This might lead to very weird file system errors in the case that
your volume ever wants to use the whole space and it might even render your
volume completely unmountable (depending on the check the file system driver
performs on the block device before allowing mounting).

For example, let's say you have a device at /dev/loop0 that has an ext4 file
system. You want to LUKSify it. We first resize our volume. For this we find
out how large the volume is currently:
```
# tune2fs -l /dev/loop0
tune2fs 1.42.9 (4-Feb-2014)
Filesystem volume name:   <none>
Last mounted on:          <not available>
Filesystem UUID:          713cc62e-b2a2-406a-a82a-c4c1d01464e1
Filesystem magic number:  0xEF53
Filesystem revision #:    1 (dynamic)
Filesystem features:      has_journal ext_attr resize_inode dir_index filetype extent flex_bg sparse_super large_file huge_file uninit_bg dir_nlink extra_isize
Filesystem flags:         signed_directory_hash
Default mount options:    user_xattr acl
Filesystem state:         clean
Errors behavior:          Continue
Filesystem OS type:       Linux
Inode count:              64000
Block count:              256000
Reserved block count:     12800
Free blocks:              247562
Free inodes:              63989
First block:              0
Block size:               4096
[...]
```


So we now know that our device is 256000 blocks of 4096 bytes each, so exactly
1000 MiB. We verify this is correct (it is in this case). So we now want to
decrease the file system size to 900 MiB. 900 MiB = 900 * 1024 * 1024 bytes =
943718400 bytes. With a file system block size of 4096 bytes we arrive at
943718400 / 4096 = 230400 blocks for the file system with decreased size. So we
resize the file system:
```
# resize2fs /dev/loop0 230400
resize2fs 1.42.9 (4-Feb-2014)
Resizing the filesystem on /dev/loop0 to 230400 (4k) blocks.
The filesystem on /dev/loop0 is now 230400 blocks long.
```


That was successful. Perfect. Now (if you haven't already), umount the volume.

*WARNING*
  Do not forget to unmount the file system before conversion


## Plain to LUKS conversion
After having done the preparation as described in the :ref:`preparation`
subsection, we can proceed to LUKSify the device. By default the initial
randomized key is read from /dev/urandom and written to
/root/initial_keyfile.bin. This is okay for us, we will remove the appropriate
keyslot for this random key anyways in the future.  It is only used for
bootstrapping. We start the conversion:
```
# ./luksipc -d /dev/loop0
WARNING! luksipc will perform the following actions:
   => Normal LUKSification of plain device /dev/loop0
   -> luksFormat will be performed on /dev/loop0

Please confirm you have completed the checklist:
    [1] You have resized the contained filesystem(s) appropriately
    [2] You have unmounted any contained filesystem(s)
    [3] You will ensure secure storage of the keyfile that will be generated at /root/initial_keyfile.bin
    [4] Power conditions are satisfied (i.e. your laptop is not running off battery)
    [5] You have a backup of all important data on /dev/loop0

    /dev/loop0: 1024 MiB = 1.0 GiB
    Chunk size: 10485760 bytes = 10.0 MiB
    Keyfile: /root/initial_keyfile.bin
    LUKS format parameters: None given

Are all these conditions satisfied, then answer uppercase yes:
```


Please, read the whole message thourougly. There is no going back from this. If
and only if you're 100% sure that all preconditions are satisfied, answer
"YES" and press return:
```
Are all these conditions satisfied, then answer uppercase yes: YES
[I]: Created raw device alias: /dev/loop0 -> /dev/mapper/alias_luksipc_raw_89ee2dc8
[I]: Size of reading device /dev/loop0 is 1073741824 bytes (1024 MiB + 0 bytes)
[I]: Backing up physical disk /dev/loop0 header to backup file header_backup.img
[I]: Performing luksFormat of /dev/loop0
[I]: Performing luksOpen of /dev/loop0 (opening as mapper name luksipc_7a6bfc08)
[I]: Size of luksOpened writing device is 1071644672 bytes (1022 MiB + 0 bytes)
[I]: Write disk smaller than read disk by 2097152 bytes (2048 kB + 0 bytes, occupied by LUKS header)
[I]: Starting copying of data, read offset 10485760, write offset 0
[I]:  0:00:  10.8%       110 MiB / 1022 MiB     0.0 MiB/s   Left:     912 MiB  0:00 h:m
[I]:  0:00:  20.5%       210 MiB / 1022 MiB     0.0 MiB/s   Left:     812 MiB  0:00 h:m
[I]:  0:00:  30.3%       310 MiB / 1022 MiB     0.0 MiB/s   Left:     712 MiB  0:00 h:m
[I]:  0:00:  40.1%       410 MiB / 1022 MiB     0.0 MiB/s   Left:     612 MiB  0:00 h:m
[I]:  0:00:  49.9%       510 MiB / 1022 MiB   412.0 MiB/s   Left:     512 MiB  0:00 h:m
[I]:  0:00:  59.7%       610 MiB / 1022 MiB   402.4 MiB/s   Left:     412 MiB  0:00 h:m
[I]:  0:00:  69.5%       710 MiB / 1022 MiB   401.5 MiB/s   Left:     312 MiB  0:00 h:m
[I]:  0:00:  79.3%       810 MiB / 1022 MiB   360.4 MiB/s   Left:     212 MiB  0:00 h:m
[I]:  0:00:  89.0%       910 MiB / 1022 MiB   350.0 MiB/s   Left:     112 MiB  0:00 h:m
[I]:  0:00:  98.8%      1010 MiB / 1022 MiB   344.8 MiB/s   Left:      12 MiB  0:00 h:m
[I]: Disk copy completed successfully.
[I]: Synchronizing disk...
[I]: Synchronizing of disk finished.
```


The volume was successfully converted! Now let's first add a passphrase that we
want to use for the volume (or any other method of key, your choice). You can
actually even do this while the copying process is running:
```
# cryptsetup luksAddKey /dev/loop0 --key-file=/root/initial_keyfile.bin
Enter new passphrase for key slot:
Verify passphrase:
```


Let's check this worked:
```
# cryptsetup luksDump /dev/loop0
LUKS header information for /dev/loop0

Version:        1
Cipher name:    aes
Cipher mode:    xts-plain64
Hash spec:      sha1
Payload offset: 4096
MK bits:        256
MK digest:      b2 34 b8 7b 70 e8 78 17 a4 12 00 41 dc a4 bc 70 a3 50 02 22
MK salt:        ee 25 b4 f0 11 94 25 d1 2b 97 42 6c a6 ff 3d 1d
                e7 6d 1e 15 dd a0 07 17 25 82 d1 f9 14 6c ab e9
MK iterations:  50125
UUID:           3e21bbe0-3d70-4189-8f19-04fb7d7c5bb9

Key Slot 0: ENABLED
    Iterations:             201892
    Salt:                   9d b6 a1 f5 0f 91 ee 24 be 49 0e f7 f9 62 a2 06
                            aa 45 79 7f 1a 56 5c 8c a3 03 15 a0 d2 9e ca e5
    Key material offset:    8
    AF stripes:             4000
Key Slot 1: ENABLED
    Iterations:             198756
    Salt:                   46 b4 21 fb e3 12 54 18 ff 8d 05 24 75 fc 3c 4b
                            3c 90 77 47 43 b6 0b 28 d9 b6 86 44 30 9e 20 d2
    Key material offset:    264
    AF stripes:             4000
Key Slot 2: DISABLED
Key Slot 3: DISABLED
Key Slot 4: DISABLED
Key Slot 5: DISABLED
Key Slot 6: DISABLED
Key Slot 7: DISABLED
```


You can see the initial keyfile (slot 0) and the passphrase we just added (slot
1). Let's scrub the initial keyslot so the initial keyfile becomes useless. We
do this by scrubbing slot 0. Don't worry, you cannot choose the wrong slot
here; cryptsetup won't permit you to remove the wrong slot since you must prove
that you still have at least access to one remaining slot (by entering your
passphrase):
```
# cryptsetup luksKillSlot /dev/loop0 0
Enter any remaining passphrase:
```


And check again:
```
# cryptsetup luksDump /dev/loop0
LUKS header information for /dev/loop0

Version:        1
Cipher name:    aes
Cipher mode:    xts-plain64
Hash spec:      sha1
Payload offset: 4096
MK bits:        256
MK digest:      b2 34 b8 7b 70 e8 78 17 a4 12 00 41 dc a4 bc 70 a3 50 02 22
MK salt:        ee 25 b4 f0 11 94 25 d1 2b 97 42 6c a6 ff 3d 1d
                e7 6d 1e 15 dd a0 07 17 25 82 d1 f9 14 6c ab e9
MK iterations:  50125
UUID:           3e21bbe0-3d70-4189-8f19-04fb7d7c5bb9

Key Slot 0: DISABLED
Key Slot 1: ENABLED
    Iterations:             198756
    Salt:                   46 b4 21 fb e3 12 54 18 ff 8d 05 24 75 fc 3c 4b
                            3c 90 77 47 43 b6 0b 28 d9 b6 86 44 30 9e 20 d2
    Key material offset:    264
    AF stripes:             4000
Key Slot 2: DISABLED
Key Slot 3: DISABLED
Key Slot 4: DISABLED
Key Slot 5: DISABLED
Key Slot 6: DISABLED
Key Slot 7: DISABLED
```


Perfect, only our slot 1 (passphrase) is left now, you can safely discard the
initial_keyfile.bin now.

Last step, resize the filesystem to its original size. For this we must first
mount the cryptographic file system and then call the resize2fs utility again:
```
# cryptsetup luksOpen /dev/loop0 newcryptofs
Enter passphrase for /dev/loop0:

# resize2fs /dev/mapper/newcryptofs
resize2fs 1.42.9 (4-Feb-2014)
Resizing the filesystem on /dev/mapper/newcryptofs to 255488 (4k) blocks.
The filesystem on /dev/mapper/newcryptofs is now 255488 blocks long.
```


You can see that the filesystem now occupies all available space (998 MiB).



## LUKS to LUKS conversion
There are situations in which you might want to re-encrypt your LUKS device.
For example, let's say you have a cryptographic volume and multiple users have
access to it, each with their own keyslot. Now suppose you forfeit the rights
of one person to the volume. Technically you would do this by killing the
appropriate key slot of the key that was assigned to the user. This means the
user can from then on not unlock the volume using the LUKS keyheader.

But suppose the user you want whose access you want to revoke had -- while
still in possession of a valid key -- access to the file system container
itself. Then with that LUKS header he can still (even when the slot was killed)
derive the underlying cryptographic key that secures the data. The only way to
remedy this is to reencrypt the whole volume with a different bulk-encryption
key.

Another usecase are old LUKS volumes: the algorithms that were used at creation
may not be suitable anymore. For example, maybe you have switched to some other
hardware platform that has hardware support for specific algorithms and you can
only take advantage of those when you choose a specific encryption algorithm.
Or maybe the alignment that was adequate a couple of years back is not adquate
anymore for you. For example, older cryptsetup instances used 1028 kiB headers,
which is an odd size. Or maybe LUKS gained new features that you want to use.

In any case, there are numerous cases why you want to turn a LUKS volume into
another LUKS volume. This process is called "reLUKSification" within luksipc
and it is something that is supported from 0.03 onwards.

Let's say you have a partition called /dev/sdh2 which you want to reLUKSify.
First let's see what the used encryption parameters are:
```
# cryptsetup luksDump /dev/sdh2
LUKS header information for /dev/sdh2

Version:        1
Cipher name:    aes
Cipher mode:    xts-plain64
Hash spec:      sha1
Payload offset: 4096
MK bits:        256
MK digest:      b1 44 6a 73 e3 06 27 27 a2 fe c2 59 e5 3a 39 2e 15 d7 d7 e0
MK salt:        09 6d 6a 24 66 28 43 f7 f3 55 a9 9d 0a 40 77 58
                e0 1f 7c 30 b9 63 96 eb 99 34 52 4f 72 ba 57 ac
MK iterations:  49750
UUID:           6495d24d-34ac-41f5-a594-c5058cc31ed3

Key Slot 0: ENABLED
    Iterations:             206119
    Salt:                   99 c8 48 50 c3 a6 83 0d f9 39 a4 4d 0a 35 b0 ab
                            13 83 ee fd 9f 91 8d 92 a6 cf 42 50 9b 89 a6 be
    Key material offset:    8
    AF stripes:             4000
Key Slot 1: DISABLED
Key Slot 2: DISABLED
Key Slot 3: DISABLED
Key Slot 4: DISABLED
Key Slot 5: DISABLED
Key Slot 6: DISABLED
Key Slot 7: DISABLED
```


We'll now open the device with our old key (a passphrase):
```
# cryptsetup luksOpen /dev/sdh2 oldluks
```


Just for demonstration purposes, we can calculate the MD5SUM over the whole
block device (you won't need to do that, it's just a demo):
```
# md5sum /dev/mapper/oldluks
48d9763be76ddb4fb990367f8d6b8c22  /dev/mapper/oldluks
```


For reLUKSification to work, you need to supply the path to the unlocked device
(from where data will be read) as well as the path to the underlying raw device
(which will be luksFormatted).

You currently have your (raw) disk at /dev/sdh2 and your (unlocked) read disk
at /dev/mapper/oldluks. It may be possible that a new LUKS header is even
larger than the old header as now, which will lead to truncation of data at the
very end of the partition. This will be the case, for example, if you reLUKSify
volumes that have a 1028 kiB LUKS header and recreate with a recent version
which writes 2048 kiB LUKS headers. You need to take all measures to decrease
the size of the contained file system, as shown in :ref:`preparation`.  These
steps will not be repeated here, but you **must** perform them nevertheless if
you want to avoid losing data.

After the disk is unlocked, you call luksipc. In addition to the raw device
which you want to convert you will also now have to specify the block device
name of the unlocked device. The raw device is the one that luksFormat and
luksOpen will be called on and the read device is the device from which data
will be read during the copy procedure. Here's how the call to luksipc looks
like. We assume that we want to change the underlying hash function to SHA256:
```
# luksipc --device /dev/sdh2 --readdev /dev/mapper/oldluks --luksparams='-h,sha256'
WARNING! luksipc will perform the following actions:
   => reLUKSification of LUKS device /dev/sdh2
   -> Which has been unlocked at /dev/mapper/oldluks
   -> luksFormat will be performed on /dev/sdh2

Please confirm you have completed the checklist:
    [1] You have resized the contained filesystem(s) appropriately
    [2] You have unmounted any contained filesystem(s)
    [3] You will ensure secure storage of the keyfile that will be generated at /root/initial_keyfile.bin
    [4] Power conditions are satisfied (i.e. your laptop is not running off battery)
    [5] You have a backup of all important data on /dev/sdh2

    /dev/sdh2: 2512 MiB = 2.5 GiB
    Chunk size: 10485760 bytes = 10.0 MiB
    Keyfile: /root/initial_keyfile.bin
    LUKS format parameters: -h,sha256

Are all these conditions satisfied, then answer uppercase yes: YES
[I]: Created raw device alias: /dev/sdh2 -> /dev/mapper/alias_luksipc_raw_60377226
[I]: Size of reading device /dev/mapper/oldluks is 2631925760 bytes (2510 MiB + 0 bytes)
[I]: Backing up physical disk /dev/sdh2 header to backup file header_backup.img
[I]: Performing luksFormat of /dev/sdh2
[I]: Performing luksOpen of /dev/sdh2 (opening as mapper name luksipc_dbb86eda)
[I]: Size of luksOpened writing device is 2631925760 bytes (2510 MiB + 0 bytes)
[I]: Write disk size equal to read disk size.
[I]: Starting copying of data, read offset 10485760, write offset 0
[I]:  0:00:   4.4%       110 MiB / 2510 MiB    43.5 MiB/s   Left:    2400 MiB  0:00 h:m
[I]:  0:00:   8.4%       210 MiB / 2510 MiB    34.1 MiB/s   Left:    2300 MiB  0:01 h:m
[I]:  0:00:  12.4%       310 MiB / 2510 MiB    21.9 MiB/s   Left:    2200 MiB  0:01 h:m
[...]
[I]:  0:02:  88.0%      2210 MiB / 2510 MiB    17.3 MiB/s   Left:     300 MiB  0:00 h:m
[I]:  0:02:  92.0%      2310 MiB / 2510 MiB    17.6 MiB/s   Left:     200 MiB  0:00 h:m
[I]:  0:02:  96.0%      2410 MiB / 2510 MiB    18.0 MiB/s   Left:     100 MiB  0:00 h:m
[I]:  0:02: 100.0%      2510 MiB / 2510 MiB    18.2 MiB/s   Left:       0 MiB  0:00 h:m
[I]: Disk copy completed successfully.
[I]: Synchronizing disk...
[I]: Synchronizing of disk finished.
```


After the process has finished, the old LUKS device /dev/mapper/oldluks will
still be open. Be very careful not to do anything with that device, however!
It's safe to close it:
```
# cryptsetup luksClose oldluks
```


Then, let's open the device with the new key:
```
# cryptsetup luksOpen /dev/sdh2 newluks -d /root/initial_keyfile.bin
```


And check that the conversion worked:
```
# md5sum /dev/mapper/newluks
48d9763be76ddb4fb990367f8d6b8c22  /dev/mapper/newluks
```


Which it did :-)

## Conversion problems
If anything goes wrong, you will find advice in this section. Again the two
distinct cases will be in different subsections. Errors with plain to LUKS
conversion will be discussed first and errors with LUKS to LUKS conversion
(reLUKSificiation) in a different subsection.


## Problems during plain to LUKS conversion
You may find yourself here because a luksipc process has crashed mid-conversion
(accidental Ctrl-C or reboot) and you're panicing. Breathe. luksipc is designed
so that it is robust against these issues.

Basically, to be able to resume a luksipc process you need to have two things:

1. The data of the last overwritten block (there's always one "shadow" block
   that needs to be kept in memory, because usually the destination partition is
   smaller than the source partition because of the LUKS header)
2. The exact location of where the interruption occured.

luksipc stores exactly this (incredibly critical) information in a "resume
file" should the resume process be interrupted. It is usually called
"resume.bin". For example, say I interrupt the LUKS conversion of a disk, this
will be shown:
```
# luksipc -d /dev/sdf1
[...]
[I]:  0:00:  32.1%       110 MiB / 343 MiB     6.4 MiB/s   Left:     233 MiB  0:00 h:m
^C[C]: Shutdown requested by user interrupt, please be patient...
[I]: Gracefully shutting down.
[I]: Synchronizing disk...
[I]: Synchronizing of disk finished.
```


If you go into more detail (log level increase) here's what you'll see:
```
# luksipc -d /dev/sdf1 -l4
[...]
[I]:  0:00:  32.1%       110 MiB / 343 MiB     6.2 MiB/s   Left:     233 MiB  0:00 h:m
^C[C]: Shutdown requested by user interrupt, please be patient...
[I]: Gracefully shutting down.
[D]: Wrote resume file: read pointer offset 136314880 write pointer offset 115343360, 10485760 bytes of data in active buffer.
[D]: Closing read/write file descriptors 4 and 5.
[I]: Synchronizing disk...
[I]: Synchronizing of disk finished.
[D]: Subprocess [PID 17857]: Will execute 'cryptsetup luksClose luksipc_f569b0bb'
[D]: Subprocess [PID 17857]: cryptsetup returned 0
[D]: Subprocess [PID 17860]: Will execute 'dmsetup remove /dev/mapper/alias_luksipc_raw_277f5e96'
[D]: Subprocess [PID 17860]: dmsetup returned 0
```


You can see the exact location of the interruption: The read pointer was at
offset 136314880 (130 MiB), the write pointer was at offset 115343360 (110 MiB)
and there are currently 10 MiB of data in the shadow buffer. Everything was
saved to a resume file. Here's an illustration of what it looks like. Every
block is 10 MiB in size:
```
       100    110    120    130    140
        |      |      |      |      |
        v      v      v      v      v
    ----+------+------+------+------+----
     ...|      | BUF1 | BUF2 |      |...
    ----+------+------+------+------+----
               ^             ^
               |             |
               W             R
```


At this point in time, luksipc has exactly two blocks in memory, BUF1 and BUF2.
This is why the read pointer is ahead two block sizes of the write pointer. Now
in the next step (if no interruption had occured) the BUF1 buffer would be
written to the LUKS device offset 110 MiB. This would overwrite some of the
plain data in BUF2, too (because the LUKS header means that there's an offset
between read- and write disk!). Therefore both have to be kept in memory.

But since the system was interrupted, it is fully sufficient to only save BUF1
to disk together with the write pointer location.

With the help of this resume file, you can continue the conversion process:
```
# luksipc -d /dev/sdf1 --resume resume.bin
[...]
[I]: Starting copying of data, read offset 125829120, write offset 115343360
[I]:  0:00:  64.1%       220 MiB / 343 MiB     6.6 MiB/s   Left:     123 MiB  0:00 h:m
[I]:  0:00:  93.3%       320 MiB / 343 MiB     9.2 MiB/s   Left:      23 MiB  0:00 h:m
[...]
```


Now we see that the process was resumed with the write pointer at the 110 MiB
mark and the read pointer at the 120 MiB mark. The next step would now be for
luksipc to read in BUF2 and we're exatly in the situation in which the abort
occured. Then from there on everything works like usual.

One thing you have to be very careful about is making copies of the resume
file. You have to be **very** careful about this. Let's say you copied the
resume file to some other location and accidently applied it twice. For
example, you run luksipc a first time and abort it. The resume file is written,
you copy it to resume2.bin. You resume the process (luksipc run 2) and let it
finish. Then you resume the process again with resume2.bin. What will happen is
that all data that was written in the resume run is encrypted **twice** and
will be unintelligible.  This can obviously be recovered, but it will require
very careful twiddling and lots of work. Just don't do it.

To prevent this sort of thing, luksipc truncates the resume file when resuming
only after everything else has worked (and I/O operation starts). This prevents
you from accidently applying a resume file twice to an interrupted conversion
process.



## Problems during LUKS to LUKS conversion
When a reLUKSification process aborts unexpectedly (but gracefully), a resume
file is written just as it would have been during LUKSification. So resuming
just like above is easily possible.  But suppose the case is a tad bit more
complicated: Let's say that someone accidently issued a reboot command during
reLUKSification. The reboot command causes a SIGTERM to be issued to the
luksipc process.  luksipc catches the signal, writes the resume.bin file and
shuts down gracefully. Then the system reboots.

For reLUKSification to work you need to have access to the plain (unlocked)
source container. Here's the big "but": In order to unlock the original
container, you need to use cryptsetup luksOpen. But the LUKS header has been
overwritten by the destination (final) LUKS header already. Therefore you can't
unlock the source data anymore.

At least you couldn't if this situation wouldn't have been anticipated by
luksipc. Lucky for you, it has been. When first firing up luksipc, a backup of
the raw device header (typically 128 MiB in size) is done by luksipc in a file
usually called "header_backup.img". You can use this header together with the
raw parition to open the partition using the old key. When you have opened the
device with the old key, we can just resume the process as we normally would.

First, this is the reLUKSificiation process that aborts. We assume our
container is unlocked at /dev/mapper/oldluks. Let's check the MD5 of the
container first (to verify everything ran smoothly):
```
# md5sum /dev/mapper/oldluks
41dc86251cba7992719bbc85de5628ab  /dev/mapper/oldluks
```


Alright, let's start the luksipc process (which will be interrupted):
```
# luksipc -d /dev/loop0 --readdev /dev/mapper/oldluks
[...]
[I]:  0:00:  10.8%       110 MiB / 1022 MiB     0.0 MiB/s   Left:     912 MiB  0:00 h:m
^C[C]: Shutdown requested by user interrupt, please be patient...
[I]: Gracefully shutting down.
[...]
```


Now let's say we've closed /dev/mapper/oldluks (e.g. by a system reboot). We
need to find a way to reopen it with the old header and old key in order to
successfully resume the proces. For this, we do:
```
# cryptsetup luksOpen --header=header_backup.img /dev/loop0 oldluks
```


And then, finally, we're able to resume luksipc:
```
# luksipc -d /dev/loop0 --readdev /dev/mapper/oldluks --resume resume.bin
[...]
[I]: Starting copying of data, read offset 220200960, write offset 209715200
[I]:  0:00:  30.3%       310 MiB / 1022 MiB     0.0 MiB/s   Left:     712 MiB  0:00 h:m
[I]:  0:00:  40.1%       410 MiB / 1022 MiB   147.9 MiB/s   Left:     612 MiB  0:00 h:m
```


Now after the process is run, let's do some cleanups:
```
# dmsetup remove oldluks
# dmsetup remove hybrid
# losetup -d /dev/loop3
```


And open our successfully converted device:
```
# cryptsetup luksOpen /dev/loop0 newluks -d /root/initial_keyfile.bin
```


But did it really work? We can check:
```
# md5sum /dev/mapper/newluks
41dc86251cba7992719bbc85de5628ab  /dev/mapper/newluks
```


Yes, it sure did :-)

Be aware that this is an absolute emergency recovery proedure that you'd only
use if everything else fails (i.e. the original source LUKS device was
accidently closed).  Any mistake whatsoever (e.g. wrong offsets) will cause you
to completely pulp your disk. So be very very careful with this and double
check everything.

## Testing
It's always a good idea to perform some tests before you give some unknown tool
by some unknown author a shot to handle your precious data. Here's a hint how
you could do this. I've setup some garbage partition on a drive which is
exactly 1234 MiB in size (1293942784 bytes). That partition is filled
completely with zeros:
```
# dd if=/dev/zero of=/dev/sda1 bs=1M
dd: error writing ‘/dev/sda1’: No space left on device
1235+0 records in
1234+0 records out
1293942784 bytes (1,3 GB) copied, 30,6571 s, 42,2 MB/s
```


Let's check that the pattern matches:
```
# md5sum /dev/sda1
e83b40511b7b154b1816ef4c03d6be7d  /dev/sda1

# dd if=/dev/zero bs=1M count=1234 | md5sum
1234+0 records in
1234+0 records out
1293942784 bytes (1,3 GB) copied, 2,71539 s, 477 MB/s
e83b40511b7b154b1816ef4c03d6be7d  -
```


Alright, so the partition is **really** filled with 1234 MiB of zeros.

Let's LUKSify it:
```
# luksipc -d /dev/sda1
WARNING! luksipc will perform the following actions:
   => Normal LUKSification of plain device /dev/sda1
   -> luksFormat will be performed on /dev/sda1

Please confirm you have completed the checklist:
    [1] You have resized the contained filesystem(s) appropriately
    [2] You have unmounted any contained filesystem(s)
    [3] You will ensure secure storage of the keyfile that will be generated at /root/initial_keyfile.bin
    [4] Power conditions are satisfied (i.e. your laptop is not running off battery)
    [5] You have a backup of all important data on /dev/sda1

    /dev/sda1: 1234 MiB = 1.2 GiB
    Chunk size: 10485760 bytes = 10.0 MiB
    Keyfile: /root/initial_keyfile.bin
    LUKS format parameters: None given

Are all these conditions satisfied, then answer uppercase yes: YES
[I]: Generated raw device alias: /dev/sda1 -> /dev/mapper/alias_luksipc_raw_944e8f9034a6344f
[I]: Size of reading device /dev/sda1 is 1293942784 bytes (1234 MiB + 0 bytes)
[I]: Performing dm-crypt status lookup on mapper name 'luksipc_41c33f9940708688'
[I]: Performing luksFormat of raw device /dev/mapper/alias_luksipc_raw_944e8f9034a6344f using key file /root/initial_keyfile.bin
[I]: Performing luksOpen of raw device /dev/mapper/alias_luksipc_raw_944e8f9034a6344f using key file /root/initial_keyfile.bin and device mapper handle luksipc_41c33f9940708688
[I]: Size of writing device /dev/mapper/luksipc_41c33f9940708688 is 1291845632 bytes (1232 MiB + 0 bytes)
[I]: Write disk smaller than read disk, 2097152 bytes occupied by LUKS header (2048 kB + 0 bytes)
[I]: Starting copying of data, read offset 10485760, write offset 0
[I]:  0:00:   8.9%       110 MiB / 1232 MiB    44.9 MiB/s   Left:    1122 MiB  0:00 h:m
[I]:  0:00:  17.0%       210 MiB / 1232 MiB    43.2 MiB/s   Left:    1022 MiB  0:00 h:m
[I]:  0:00:  25.2%       310 MiB / 1232 MiB    18.8 MiB/s   Left:     922 MiB  0:00 h:m
[I]:  0:00:  33.3%       410 MiB / 1232 MiB    21.3 MiB/s   Left:     822 MiB  0:00 h:m
[I]:  0:00:  41.4%       510 MiB / 1232 MiB    23.4 MiB/s   Left:     722 MiB  0:00 h:m
[I]:  0:00:  49.5%       610 MiB / 1232 MiB    22.0 MiB/s   Left:     622 MiB  0:00 h:m
[I]:  0:00:  57.6%       710 MiB / 1232 MiB    18.7 MiB/s   Left:     522 MiB  0:00 h:m
[I]:  0:00:  65.7%       810 MiB / 1232 MiB    19.8 MiB/s   Left:     422 MiB  0:00 h:m
[I]:  0:00:  73.9%       910 MiB / 1232 MiB    20.3 MiB/s   Left:     322 MiB  0:00 h:m
[I]:  0:00:  82.0%      1010 MiB / 1232 MiB    17.8 MiB/s   Left:     222 MiB  0:00 h:m
[I]:  0:00:  90.1%      1110 MiB / 1232 MiB    18.6 MiB/s   Left:     122 MiB  0:00 h:m
[I]:  0:01:  98.2%      1210 MiB / 1232 MiB    19.4 MiB/s   Left:      22 MiB  0:00 h:m
[I]: Disk copy completed successfully.
[I]: Synchronizing disk...
[I]: Synchronizing of disk finished.
```


Then luksOpen it:
```
# cryptsetup luksOpen /dev/sda1 myluksdev -d /root/initial_keyfile.bin
```


And check the hash:
```
# md5sum /dev/mapper/myluksdev
e2226de7d184a3c9bd4c1e3d8a56b1b2  /dev/mapper/myluksdev
```


The hash value differs from what it said before - this is absolutely to be
expected! The reason for this is that the device is now shorter (because part
of the space is used for the 2 MiB LUKS header). Proof:
```
# dd if=/dev/zero bs=1M count=1232 | md5sum
1232+0 records in
1232+0 records out
1291845632 bytes (1,3 GB) copied, 2,6588 s, 486 MB/s
e2226de7d184a3c9bd4c1e3d8a56b1b2  -
```


Now let's check the current key and reLUKSify it with a different key and
algorithm! First, let's check out the "before" values:
```
# dmsetup table myluksdev --showkeys
0 2523136 crypt aes-xts-plain64 d164b3fd2b7d482fc6e0a2d0e58f51c5dafe4560507322cb29af4bd8f552ba4f 0 8:1 4096

# cryptsetup luksDump /dev/sda1
LUKS header information for /dev/sda1

Version:        1
Cipher name:    aes
Cipher mode:    xts-plain64
Hash spec:      sha1
Payload offset: 4096
MK bits:        256
MK digest:      dd 08 3d 43 ae 50 64 7c d9 c6 20 cb de dd 7a 62 69 10 63 fe
MK salt:        f1 95 eb 18 2d 90 61 e9 c8 df 4b 4d 44 ab 62 87
                5a f5 39 5a c4 f5 3b 7a 09 8c f1 75 33 a5 f3 25
MK iterations:  50375
UUID:           127277bf-b07b-4209-bf55-37cb1c10c83b

Key Slot 0: ENABLED
    Iterations:             201892
    Salt:                   fc d9 3a 73 b4 73 ee 98 6c 35 34 a0 c7 7d 8a 71
                            5b 75 b7 6c 75 af 65 20 eb 90 7c 69 34 10 1e a6
    Key material offset:    8
    AF stripes:             4000
Key Slot 1: DISABLED
Key Slot 2: DISABLED
Key Slot 3: DISABLED
Key Slot 4: DISABLED
Key Slot 5: DISABLED
Key Slot 6: DISABLED
Key Slot 7: DISABLED
```


Then reLUKSify:
```
# my /root/initial_keyfile.bin /root/initial_keyfile_old.bin

# luksipc -d /dev/sda1 --readdev /dev/mapper/myluksdev --luksparams='-c,twofish-lrw-benbi,-s,320,-h,sha256'
WARNING! luksipc will perform the following actions:
   => reLUKSification of LUKS device /dev/sda1
   -> Which has been unlocked at /dev/mapper/myluksdev
   -> luksFormat will be performed on /dev/sda1

Please confirm you have completed the checklist:
    [1] You have resized the contained filesystem(s) appropriately
    [2] You have unmounted any contained filesystem(s)
    [3] You will ensure secure storage of the keyfile that will be generated at /root/initial_keyfile.bin
    [4] Power conditions are satisfied (i.e. your laptop is not running off battery)
    [5] You have a backup of all important data on /dev/sda1

    /dev/sda1: 1234 MiB = 1.2 GiB
    Chunk size: 10485760 bytes = 10.0 MiB
    Keyfile: /root/initial_keyfile.bin
    LUKS format parameters: -c,twofish-lrw-benbi,-s,320,-h,sha256

Are all these conditions satisfied, then answer uppercase yes: YES
[I]: Generated raw device alias: /dev/sda1 -> /dev/mapper/alias_luksipc_raw_c84651981fc98f36
[I]: Size of reading device /dev/mapper/myluksdev is 1291845632 bytes (1232 MiB + 0 bytes)
[I]: Performing dm-crypt status lookup on mapper name 'luksipc_9afeee69aec4912c'
[I]: Performing luksFormat of raw device /dev/mapper/alias_luksipc_raw_c84651981fc98f36 using key file /root/initial_keyfile.bin
[I]: Performing luksOpen of raw device /dev/mapper/alias_luksipc_raw_c84651981fc98f36 using key file /root/initial_keyfile.bin and device mapper handle luksipc_9afeee69aec4912c
[I]: Size of writing device /dev/mapper/luksipc_9afeee69aec4912c is 1291845632 bytes (1232 MiB + 0 bytes)
[I]: Write disk size equal to read disk size.
[I]: Starting copying of data, read offset 10485760, write offset 0
[I]:  0:00:   8.9%       110 MiB / 1232 MiB    43.1 MiB/s   Left:    1122 MiB  0:00 h:m
[I]:  0:00:  17.0%       210 MiB / 1232 MiB    42.0 MiB/s   Left:    1022 MiB  0:00 h:m
[I]:  0:00:  25.2%       310 MiB / 1232 MiB    28.3 MiB/s   Left:     922 MiB  0:00 h:m
[I]:  0:00:  33.3%       410 MiB / 1232 MiB    19.1 MiB/s   Left:     822 MiB  0:00 h:m
[I]:  0:00:  41.4%       510 MiB / 1232 MiB    21.3 MiB/s   Left:     722 MiB  0:00 h:m
[I]:  0:00:  49.5%       610 MiB / 1232 MiB    21.6 MiB/s   Left:     622 MiB  0:00 h:m
[I]:  0:00:  57.6%       710 MiB / 1232 MiB    19.9 MiB/s   Left:     522 MiB  0:00 h:m
[I]:  0:00:  65.7%       810 MiB / 1232 MiB    18.6 MiB/s   Left:     422 MiB  0:00 h:m
[I]:  0:00:  73.9%       910 MiB / 1232 MiB    19.8 MiB/s   Left:     322 MiB  0:00 h:m
[I]:  0:00:  82.0%      1010 MiB / 1232 MiB    19.4 MiB/s   Left:     222 MiB  0:00 h:m
[I]:  0:01:  90.1%      1110 MiB / 1232 MiB    17.6 MiB/s   Left:     122 MiB  0:00 h:m
[I]:  0:01:  98.2%      1210 MiB / 1232 MiB    18.4 MiB/s   Left:      22 MiB  0:00 h:m
[I]: Disk copy completed successfully.
[I]: Synchronizing disk...
[I]: Synchronizing of disk finished.
```


Now, let's detach the mapping of the old LUKS container first (this container
now contains complete garbage):
```
# cryptsetup luksClose myluksdev
```


And reopen it with the correct key:
```
# cryptsetup luksOpen /dev/sda1 mynewluksdev -d /root/initial_keyfile.bin
```


Check that the content is still the same:
```
# cat /dev/mapper/mynewluksdev | md5sum
e2226de7d184a3c9bd4c1e3d8a56b1b2  -
```


It sure is. Now look at the luksDump output:
```
LUKS header information for /dev/sda1

Version:        1
Cipher name:    twofish
Cipher mode:    lrw-benbi
Hash spec:      sha256
Payload offset: 4096
MK bits:        320
MK digest:      10 b9 35 7b c8 23 d7 c3 2a b9 3e e6 95 74 cf 7f ef 75 1b 32
MK salt:        3f 58 e6 1e 29 e1 c7 a2 f1 14 9e 1f c7 09 fa 23
                93 7c 9c 59 20 67 d7 a7 7e 7d fe a0 12 9f 0f 25
MK iterations:  29000
UUID:           1dd5e426-9e37-4d1e-a6f9-17aa4179eb1e

Key Slot 0: ENABLED
    Iterations:             117215
    Salt:                   9d 58 5c 30 2b dc 35 33 19 bf 78 ab 3e aa 6e 8a
                            fa 6c 9b ee 45 f7 db 9e f1 ab 0c fb cb 3c eb 51
    Key material offset:    8
    AF stripes:             4000
Key Slot 1: DISABLED
Key Slot 2: DISABLED
Key Slot 3: DISABLED
Key Slot 4: DISABLED
Key Slot 5: DISABLED
Key Slot 6: DISABLED
Key Slot 7: DISABLED
```


And the used key internally:
```
# dmsetup table mynewluksdev --showkeys
0 2523136 crypt twofish-lrw-benbi d6b007ce62de58b62331f800edf5864da390eb274b908506b368035e7a0f8ea1c3583c2b939928c3 0 8:1 4096
```


As you can see, completely different keys, completely different algorithm --
but still identical data. It worked :-)

Of course you can do this test with arbitrary data (not just constant zeros). I
was just too lazy to write a PRNG that outputs easily reproducible results.
Feel free to play around with it and please report any and all bugs if you find
some.
