Title: Installing a Lost App from a TWRP Full Backup
Date: 2020-08-09
Category: Projects
Tags: android, twrp, backup, recovery
Status: published
Summary: One of my most important apps, one of the first apps I paid for, was gone. Taken from me. Torn from the app store. Unavailable. But, armed only with a full system backup and a foolhardy love of the command line, I was going to get it back

One of my most important apps, one of the first apps I paid for, was gone. Taken from me. Torn from the app store. Unavailable. But, armed only with a full system backup and a foolhardy love of the command line, I was going to get it back.

## Background

I'll admit, I've fallen out of step with the smartphone world. My Nexus 6P is over three years old now, and I'm seriously considering swapping its battery a second time. This all came to a head several months ago when I found out abruptly that my phone was no longer officially supported by [LineageOS](https://lineageos.org/) after I installed an update and it went into a boot loop. That quickly turned into a stressful Saturday afternoon, but after much panicked head-bashing, I managed to get my 6P basically working again.

But it wasn't fully working. The Android installation was somehow sick, and symptoms included camera photos not saving consistently, random app crashes, and my alarm occasionally re-enabling itself after being turned off. So, after months of putting up with these problems, I finally sat down, made some important backups, and wiped my phone for a clean re-install with an unofficial LineageOS image. The process went about a smooth as I expected, and at long last, my phone seemed stable. I rejoiced.

Alas, all was still not well. I allow Google to make cloud backups of my phone, and the vast majority of my apps were restored automatically. Those that were not automatic (like Signal) I had prepared for and manually restored their data. Several apps, however, just did not install again and were not in the Play Store. I had seen this happen before, so I can't say I was surprised, but this time I had lost my alarm clock app, *Gentle Alarm*, which has many features I don't use and a few that I've become dependent on. After some research, it appears the developer decided not to support Android 10, and Google, in its infinite wisdom, decided to remove *Gentle Alarm* from the store entirely and make it completely unavailable, even for naive rubes like myself who are still back on the very-much-supported Android 8.

Two days of trying to use the built-in alarm clock and many fruitless searches for a feature-matched alternative taught me that I need to have *Gentle Alarm* back. Fortunately, when I first  boot loop issues, I had used my recovery image, [Team Win Recovery Project](https://twrp.me/) (TWRP) v3.2.1, to manually run a complete backup of my phone. I couldn't just restore this backup to my phone, though, for fear that it might re-introduce some of the instability I had only just fixed. So I decided to find a way to extract *Gentle Alarm* alone from the backup and install it on my phone.

## Reading the TWRP Backup File

TWRP stores its backups in tar files named  like `data.ext4.win000`, which indicates the partition (`data`), filesystem (`ext4`), and the file part number (`000-009`). I had selected the option for compressed backups, so my files were gzipped.

As noted in the discussion of [this StackExchange question](https://android.stackexchange.com/questions/171202/extracting-data-from-a-twrp-backup), however, these are not your father's tar files. Attempts to open them with my system's archive manager or the `tar` command result in

```
::text
Archive type not supported.
```

or

```
::text
tar: Malformed extended header: missing equal sign
```

Helpfully, there is a link to [a blog post by Tom Hacohen](https://stosb.com/blog/recovering-data-from-a-corrupt-tar-archive/) which walks through the process of recovering a file from one of these 'extended' tarballs. I knew that my process would probably be more difficult, however, because I couldn't be laser-focused on a single file. I wanted to extract... well, at this point I didn't even know what I was looking for, but I was sure it would involve extracting multiple files and digging through directories. The post does a good job of explaining his process and helped to jump-start my analysis.

More googling led to [a GitHub issue](https://github.com/TeamWin/Team-Win-Recovery-Project/issues/898), but the solutions didn't net me a valid-looking tar archive. A [linked Python script](https://gist.github.com/AlD/cb443fe4ba152e1de8a42fa72721552b) looked interesting, though. It basically skips through the input file 512 bytes at a time and checks to see if the first 8 bytes of that block are "TWRP" followed by 4 null bytes (`TWRP\x00\x00\x00\x00`). If so, that block of 512 bytes isn't written to the output file.

Tar archives are basically a bunch of files that are all strung into one big file along with metadata headers to let you sort the individual files back out again. Both the headers and the file data are padded into 512-byte blocks, so that Python script is searching for any headers that start with the "TWRP" bytes and stripping them out (it also strips out any file contents unlucky enough to start with that exact string, but that's unlikely). It probably didn't work for me because my files have a different kind of header that I need to strip out. With that goal in mind and the [tar file format](https://en.wikipedia.org/wiki/Tar_(computing)#File_format) in hand, I jumped into a hexdump of the archive.

It turns out, my backup files (made with TWRP 3.2.1) actual have two entries for every file in the tar archive. The first entry seems to only store selinux information, which TWRP probably uses during its full system restore process. 

* The tar header starts with the filename at line `00001200`.
* The header is 512 bytes long (`200` in hex), and the "file contents" begin at line `00001400`.
* Tar archives support multiple entries for one filename as "links". The link type is at byte offset 156 in the header. For this entry, that link type is `x` (last non-null byte on `00001290`), which isn't in the tar standard and is likely the problem for untarring programs.

```
::hexdump
00001200  2f 64 61 74 61 2f 64 61  74 61 2f 63 6f 6d 2e 6d  |/data/data/com.m|
00001210  6f 62 69 74 6f 62 69 2e  61 6e 64 72 6f 69 64 2e  |obitobi.android.|
00001220  67 65 6e 74 6c 65 61 6c  61 72 6d 2f 66 69 6c 65  |gentlealarm/file|
00001230  73 2f 67 61 43 6c 69 65  6e 74 49 64 00 00 00 00  |s/gaClientId....|
00001240  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00001260  00 00 00 00 30 31 30 30  36 30 30 00 30 30 32 33  |....0100600.0023|
00001270  36 30 36 00 30 30 32 33  36 30 36 00 30 30 30 30  |606.0023606.0000|
00001280  30 30 30 30 30 37 36 00  30 30 30 30 35 34 35 36  |0000076.00005456|
00001290  36 34 35 00 30 30 32 33  33 33 37 00 78 00 00 00  |645.0023337.x...|
000012a0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00001300  00 75 73 74 61 72 20 20  00 75 30 5f 61 31 31 38  |.ustar  .u0_a118|
00001310  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00001320  00 00 00 00 00 00 00 00  00 75 30 5f 61 31 31 38  |.........u0_a118|
00001330  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00001400  36 32 20 52 48 54 2e 73  65 63 75 72 69 74 79 2e  |62 RHT.security.|
00001410  73 65 6c 69 6e 75 78 3d  75 3a 6f 62 6a 65 63 74  |selinux=u:object|
00001420  5f 72 3a 61 70 70 5f 64  61 74 61 5f 66 69 6c 65  |_r:app_data_file|
00001430  3a 73 30 3a 63 35 31 32  2c 63 37 36 38 0a 00 00  |:s0:c512,c768...|
00001440  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
```

The actual file comes directly after the selinux file entry. Its link type (the last non-null byte on line `00001690`) is `0`, indicating that this is a normal file. The file contents start at line `00001800` and look more like the file should. This one appears to be some kind of GUID.

```
::hexdump
00001600  2f 64 61 74 61 2f 64 61  74 61 2f 63 6f 6d 2e 6d  |/data/data/com.m|
00001610  6f 62 69 74 6f 62 69 2e  61 6e 64 72 6f 69 64 2e  |obitobi.android.|
00001620  67 65 6e 74 6c 65 61 6c  61 72 6d 2f 66 69 6c 65  |gentlealarm/file|
00001630  73 2f 67 61 43 6c 69 65  6e 74 49 64 00 00 00 00  |s/gaClientId....|
00001640  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00001660  00 00 00 00 30 31 30 30  36 30 30 00 30 30 32 33  |....0100600.0023|
00001670  36 30 36 00 30 30 32 33  36 30 36 00 30 30 30 30  |606.0023606.0000|
00001680  30 30 30 30 30 34 34 00  30 30 30 30 35 34 35 36  |0000044.00005456|
00001690  36 34 35 00 30 30 32 33  32 32 32 00 30 00 00 00  |645.0023222.0...|
000016a0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00001700  00 75 73 74 61 72 20 20  00 75 30 5f 61 31 31 38  |.ustar  .u0_a118|
00001710  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00001720  00 00 00 00 00 00 00 00  00 75 30 5f 61 31 31 38  |.........u0_a118|
00001730  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00001800  38 62 33 62 34 39 61 38  2d 61 36 37 66 2d 34 38  |8b3b49a8-a67f-48|
00001810  32 32 2d 38 39 34 33 2d  38 64 31 30 32 36 31 31  |22-8943-8d102611|
00001820  39 39 65 64 00 00 00 00  00 00 00 00 00 00 00 00  |99ed............|
00001830  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
```

Thus, to get a plain tar archive, I just need to remove the selinux entry before every file. To do this, I modified the `twrp2tar.py` script I found earlier to seek out headers with the `x` link type. I also added a check for the "ustar" entry (which indicates a *Unix Standard TAR* header format) to avoid removing any file contents that happen to have an "x" at the 156-byte offset. Additionally, I needed to remove the selinux file contents along with any matching header, so I skip the next 512 bytes after a match by reading 1024 bytes and throwing away the first half.

```
::python
import sys

if len(sys.argv) < 3:
   print("Usage: %s <twrp_file> <output_tar>" % sys.argv[0])
   exit(1)


inf = open(sys.argv[1], 'rb')

out = open(sys.argv[2], 'wb')

buf = inf.read(512)
while buf != '':
   if buf[156] == 'x' and buf[256:262] == '\x00ustar':
      print("Skipping %r" % buf[:16])
      buf = inf.read(1024)[512:]
   else:
      out.write(buf)
      buf = inf.read(512)

print("Done!")

inf.close()
out.close()
```

It's a little quick-and-dirty, but it does work. Running this script on the TWRP backup files gives me a valid tar archive I can open on my computer with all the files intact.

## Recovering Android App Installation Files

I can now open and browse the backup files but I still don't know for certain what I'm looking for. I found a [lovely guide for manually restoring app data from a backup](https://www.semipol.de/2016/07/30/android-restoring-apps-from-twrp-backup.html), but it assumes the app can already be installed. I needed to restore my app installation from a backup, and searching didn't reveal many options for that.

In my backup folder, there is a file called `recovery.log` which appears to be a log of the backup process. It includes a list of all the files TWRP put in the archive. Combined with `grep`, I found this file very handy for skimming for *Gentle Alarm* files in the backup and figuring out what might help me with my problem.

```
::text
$ grep gentlealarm recovery.log
I:addFile '/data/data/com.mobitobi.android.gentlealarm' including root: 1
I:addFile '/data/data/com.mobitobi.android.gentlealarm/cache' including root: 1
I:addFile '/data/data/com.mobitobi.android.gentlealarm/files' including root: 1
I:addFile '/data/data/com.mobitobi.android.gentlealarm/files/gaClientId' including root: 1
I:addFile '/data/data/com.mobitobi.android.gentlealarm/files/.Fabric' including root: 1
[...]
```

Many entries were from the `/data/data/` directory, which are useful for restoring app data as in the blog linked above, but my eyes were on the `/data/app/` directory.

```
::text
I:addFile '/data/app/com.mobitobi.android.gentlealarm-1' including root: 1
I:addFile '/data/app/com.mobitobi.android.gentlealarm-1/lib' including root: 1
I:addFile '/data/app/com.mobitobi.android.gentlealarm-1/oat' including root: 1
I:addFile '/data/app/com.mobitobi.android.gentlealarm-1/oat/arm64' including root: 1
I:addFile '/data/app/com.mobitobi.android.gentlealarm-1/oat/arm64/base.odex' including root: 1
I:addFile '/data/app/com.mobitobi.android.gentlealarm-1/oat/arm64/base.vdex' including root: 1
I:addFile '/data/app/com.mobitobi.android.gentlealarm-1/base.apk' including root: 1
```

Now these look like installation files. The `oat/arm64` directory holds ahead-of-time compiled code, so I knew I was close, and-- be still my beating heart, is that an APK? Could I be so lucky?

Using `grep` again, I determined which archive files contained the directory I cared about (after gunzipping and running my `twrp2tar.py` script on the files, I gave them the `.tar` extension).

```
::text
$ for tarfile in `ls data.ext4.win00*.tar`;
> do echo "$tarfile";
> tar -t -f "$tarfile" | grep /data/app/com.mobitobi.android.gentlealarm-1;
> done
data.ext4.win000.tar
tar: Removing leading `/' from member names
data.ext4.win001.tar
tar: Removing leading `/' from member names
data.ext4.win002.tar
tar: Removing leading `/' from member names
data.ext4.win003.tar
tar: Removing leading `/' from member names
data.ext4.win004.tar
tar: Removing leading `/' from member names
/data/app/com.mobitobi.android.gentlealarm-1/
/data/app/com.mobitobi.android.gentlealarm-1/lib/
/data/app/com.mobitobi.android.gentlealarm-1/oat/
/data/app/com.mobitobi.android.gentlealarm-1/oat/arm64/
/data/app/com.mobitobi.android.gentlealarm-1/oat/arm64/base.odex
/data/app/com.mobitobi.android.gentlealarm-1/oat/arm64/base.vdex
/data/app/com.mobitobi.android.gentlealarm-1/base.apk
[...]
```

I extracted the APK and opened it to take a look (APKs are basically JAR files which are basically ZIP files). It appeared to be a valid APK, and several asset files told me this was the actual *Gentle Alarm* package. Optimistically, I renamed the file and broke out ADB to perform the installation over USB.

```
::text
$ adb install gentlealarm.apk 
Performing Streamed Install
Success
```

That looks like good news, but is the app truly installed?

![A screenshot of my phone's app menu, including the icon for Gentle Alarm.]({attach}app_menu_screenshot.jpg "Gentle Alarm is back, baby! Wait, why do I still have Fitbit installed? I haven't worn one of those in years.")

**Success!** Google's backup had even restored all my saved alarms, despite not restoring the app itself. That's okay, though, I'm just happy to have it back.