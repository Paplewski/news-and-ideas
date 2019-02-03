---
post_title: debos image for HummingBoard Edge
author: Maciej Pijanowski
layout: post
published: true
post_date: 2018-09-24 16:00:00

tags:
	-Debian
	-linux
	-build
	-docker
categories:
	-OS Dev
---

## Intro

In my previous posts I have shared
[my first experience with debos](https://3mdeb.com/os-dev/our-first-look-at-debos-new-debian-images-generator)
and
[how to run debos in a container](https://3mdeb.com/os-dev/debos-in-docker-the-second-attempt).
In today's post, I'd like to present how can we use all of that to
generate base Debian image for an ARM board. My board of choice for this
particular example will be the
[HummingBoard Edge](https://www.solid-run.com/product/hummingboard-edge-imx6d-0c-e/).

The post is inspired by the feedback from the new users
(such as [this one](https://github.com/go-debos/debos/issues/114)) that there
are no end-to-end examples how to quickly start using this tool.

## HummingBoard Edge upstream support

The `Hummingboard Edge` is described in the Linux by the
[hummingboard2 devicetree](https://github.com/torvalds/linux/blob/master/arch/arm/boot/dts/imx6dl-hummingboard2.dts).
It is supported
[since the 4.16 Linux release](https://www.phoronix.com/scan.php?page=news_item&px=Linux-4.16-New-ARM-Hardware).
I have decided to use the `sid` flavor of `Debian` in order to get quite recent
kernel version
([4.18+98](https://packages.debian.org/search?suite=sid&arch=any&searchon=names&keywords=linux-image-armmp)
at the moment of writing).

## debos recipe

> The full recipe and all other files are available in the
> [3mdeb/debos-recipes repository on github](https://github.com/3mdeb/debos-recipes/tree/hb2/debian/arm/image-hb2).

I wanted to create a really base system image, just to try out that it boots
properly. I took the
[existing RPI3 recipe](https://github.com/go-debos/debos-recipes/blob/master/debian/arm64/image-rpi3/debimage-rpi3.yaml)
as a starting point.

As stated in the previous paragraph, the `sid` flavor of Debian will be used,
hence following action in the recipe file appears:

```
  - action: debootstrap
    suite: "sid"
    components:
      - main
    mirror: https://deb.debian.org/debian
    variant: minbase
```

Some of the following actions are:

* setting the hostname with a `run` action:

```
  - action: run
    description: Set hostname
    chroot: true
    command: echo HummingBoard2 > /etc/hostname
```

> note that we can decide with the
  [chroot flag](https://godoc.org/github.com/go-debos/debos/actions#hdr-Run_Action)
  whether the command shall be executed on the host or in the chrooted environment

* copying over `extlinux` config files with an `overlay` action:

```
  - action: overlay
    description: Install U-boot extlinux menu file
    source: u-boot
```

> The
> [extlinux config I came up with](https://github.com/3mdeb/debos-recipes/blob/hb2/debian/arm/image-hb2/u-boot/boot/extlinux/extlinux.conf)
> could have been definitely improved (e.g. to be autogenerated, to use `UUID`
> for root partition etc.), but is perfectly enough for the purpose of this
> demonstration.

* installation of the `Linux` and `U-Boot` packages:

```
  - action: apt
    description: Install Linux and U-Boot packages
    recommends: false
    packages:
      - linux-image-armmp
      - u-boot-imx
```

Another important action is the `image-partition`. We can specify there the
final image size, partitions to be created as well as their mount points. Note
that the `/etc/fstab` entries will be automatically added for created
partitions.

```
  # leave 4MB offset at the image start for U-Boot installation
  - action: image-partition
    description: Create partitioned image
    imagename: {{ $image }}
    imagesize: 1GB
    partitiontype: msdos
    mountpoints:
      - mountpoint: /
        partition: root
    partitions:
      - name: root
        fs: ext4
        start: 4MB
        end: 100%
        flags: [ boot ]
```

The substantial difference from the `RPI3` recipe is the bootloader. Unlike many
other `ARM` boards, `RPI3` does not use the `U-Boot` installed directly (into
not partitioned area) at the start of the image file (or block device).

As shown in the
[mx6cuboxi README](https://github.com/u-boot/u-boot/blob/master/board/solidrun/mx6cuboxi/README),
`SPL U-Boot` shall be installed into the device as follows:

```
sudo dd if=SPL of=/dev/mmcblk0 bs=1k seek=1; sync
sudo dd if=u-boot.img of=/dev/mmcblk0 bs=1k seek=69; sync
```

In `debos`, we can use the `raw` action and use build-in `sector` keyword to
install binaries at required offsets:

```
  # 'filesystem' origin points to the target root filesystem
  # 2 sectors = 1K
  # bs=1K seek=1
  - action: raw
    description: Install SPL to the image
    origin: filesystem
    source: /usr/lib/u-boot/mx6cuboxi/SPL
    offset: {{ sector 2 }}

  # bs=1K seek=69
  - action: raw
    description: Install U-Boot to the image
    origin: filesystem
    source: /usr/lib/u-boot/mx6cuboxi/u-boot.img
    offset: {{ sector 138 }}
```

Note that we can use `filesystem` as an argument for the `origin` in order to
gain access to files located in the filesystem we have generated in the
previous steps. This feature does not seem to be documented somewhere, but I
found such map
[in the code](https://github.com/go-debos/debos/blob/master/cmd/debos/debos.go#L110).
Other useful arguments might be: `artifacts` or `recipe`.

The final actions include deploying filesystem to image and compressing
the final image. Optionally, the
[bmap file](https://source.tizen.org/documentation/reference/bmaptool/introduction)
can be generated.

```
  - action: filesystem-deploy
    description: Deploy filesystem to the image

  - action: run
    description: Create bmap file
    postprocess: true
    command: bmaptool create {{ $image }} > {{ $image }}.img.bmap

  - action: run
    description: Compress image
    postprocess: true
    command: gzip -f9 {{ $image }}
```

## Image building

I've used our [debos docker container](https://github.com/3mdeb/debos-docker) to
perform `debos` build. I chose to save the
[run script](https://github.com/3mdeb/debos-docker/blob/master/run.sh) in my
`PATH` for easy execution:

```
debos-docker debimage-hb2.yaml
```

The end result looks like:

```
< truncated >
2018/09/19 19:07:19 ==== Install SPL to the image ====
2018/09/19 19:07:19 ==== Install U-Boot to the image ====
2018/09/19 19:07:19 ==== Deploy filesystem to the image ====
2018/09/19 19:07:20 Setting up fstab
2018/09/19 19:07:20 Setting up /etc/kernel/cmdline
Powering off.
2018/09/19 19:07:21 ==== Create bmap file ====
2018/09/19 19:07:26 ==== Compress image ====
2018/09/19 19:08:35 ==== Recipe done ====
```

And we should have a final image file:

```
-rw-r--r-- 1 maciej root   143M wrz 21 11:51 debian-hb2.img.gz
```

## Image flashing

### The usual way

No explanation needed for the `dd` command:

```
gzip -cdk debian-hb2.img.gz | sudo dd of=/dev/mmcblk0 bs=16M status=progress
```

### The fancy way

Thanks to the `bmap` file we created, we can use `bmaptools` to flash the
image. Explanation of this tool could be itself a story for another post. More
details can be found in the
[bmaptool documentation](https://source.tizen.org/documentation/reference/bmaptool/introduction).

We might need to install the tool first:

```
sudo apt install bmap-tools
```

Image can be flashed with following command:

```
bmaptool copy --bmap debian-hb2.img.img.bmap debian-hb2.img.gz /dev/mmcblk0
```

### Comparison

It is a good opportunity to show a quick comparison between those two:

* `dd` - 1 minute 18 seconds

```
time sh -c 'gzip -cdk debian-hb2.img.gz | sudo dd of=/dev/mmcblk0 bs=16M status=progress && sync'
1000000000 bytes (1,0 GB, 954 MiB) copied, 32 s, 31,2 MB/s
0+27202 records in
0+27202 records out
1000000000 bytes (1,0 GB, 954 MiB) copied, 78,7137 s, 12,7 MB/s
sh -c   6,62s user 1,27s system 10% cpu 1:18,73 total
```

* `bmaptool` - 38 seconds

```
time sudo sh -c 'bmaptool copy --bmap debian-hb2.img.img.bmap debian-hb2.img.gz /dev/mmcblk0 && sync'
bmaptool: info: block map format version 2.0
bmaptool: info: 244141 blocks of size 4096 (953.7 MiB), mapped 110274 blocks (430.8 MiB or 45.2%)
bmaptool: info: copying image 'debian-hb2.img.gz' to block device '/dev/mmcblk0' using bmap file 'debian-hb2.img.img.bmap'
bmaptool: info: 100% copied
bmaptool: info: synchronizing '/dev/mmcblk0'
bmaptool: info: copying time: 38.0s, copying speed 11.3 MiB/sec
sudo sh -c   8,48s user 1,26s system 25% cpu 38,209 total
```

## Image running

The boot log:

```
U-Boot SPL 2018.05+dfsg-1 (May 10 2018 - 20:24:57 +0000)
Trying to boot from MMC1


U-Boot 2018.05+dfsg-1 (May 10 2018 - 20:24:57 +0000)

CPU:   Freescale i.MX6DL rev1.3 996 MHz (running at 792 MHz)
CPU:   Commercial temperature grade (0C to 95C) at 38C
Reset cause: POR
Board: MX6 Hummingboard2
DRAM:  1 GiB
MMC:   FSL_SDHC: 0
Loading Environment from MMC... *** Warning - bad CRC, using default environment

Failed (-5)
No panel detected: default to HDMI
Display: HDMI (1024x768)
In:    serial
Out:   serial
Err:   serial
Net:   FEC
Hit any key to stop autoboot:  2  1  0
switch to partitions #0, OK
mmc0 is current device
Scanning mmc 0:1...
Found /boot/extlinux/extlinux.conf
Retrieving file: /boot/extlinux/extlinux.conf
498 bytes read in 110 ms (3.9 KiB/s)
U-Boot menu
1:    Debian GNU/Linux kernel
2:    Debian GNU/Linux kernel (rescue target)
Enter choice: 1:    Debian GNU/Linux kernel
Retrieving file: /initrd.img
17499178 bytes read in 1028 ms (16.2 MiB/s)
Retrieving file: /vmlinuz
4174336 bytes read in 395 ms (10.1 MiB/s)
append: root=/dev/mmcblk1p1 ro rootwait console=ttymxc0,115200
Retrieving file: /boot/dtbs/imx6dl-hummingboard2.dtb
39570 bytes read in 120 ms (321.3 KiB/s)
## Flattened Device Tree blob at 18000000
   Booting using the fdt blob at 0x18000000
   Using Device Tree in place at 18000000, end 1800ca91

Starting kernel ...

[    0.000000] Booting Linux on physical CPU 0x0
[    0.000000] Linux version 4.18.0-1-armmp (debian-kernel@lists.debian.org) (gcc version 7.3.0 (Debian 7.3.0-29)) #1 SMP Debian 4.18.6-1 (2018-09-06)
[    0.000000] CPU: ARMv7 Processor [412fc09a] revision 10 (ARMv7), cr=10c5387d

< trucated >

Starting Update UTMP about System Runlevel Changes...
[  OK  ] Started Update UTMP about System Runlevel Changes.

Debian GNU/Linux buster/sid HummingBoard2 ttymxc0


HummingBoard2 login: user
Password: 
Linux HummingBoard2 4.18.0-1-armmp #1 SMP Debian 4.18.6-1 (2018-09-06) armv7l
```

## Conclusion

As I have shown in this post, `debos` can be quite easily used for building
`Debian` image for an `ARM` platform. This is especially true if such board
has good mainline support and the `Linux` and `U-Boot` packages for it are
already available in the `Debian` package feed. I hope that my post can be
helpful for new `debos` users and that my recipe gets merged into the
[debos-recipes](https://github.com/go-debos/debos-recipes), so it can be easily
accessible.