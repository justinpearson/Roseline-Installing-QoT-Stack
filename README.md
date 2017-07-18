---
title: "Roseline Project: Installing the Roseline Quality-of-Time Software Stack"
header-includes:
    <script type="text/javascript" src="highslide/highslide.js" ></script>
---



# Roseline Project: Installing the Roseline Quality-of-Time Software Stack

How to build a linux kernel equipped with Roseline's Quality-of-Time service (Aug 2016)

- UCLA trip with Sean Kim 
- Aug 12, 2016
- Justin Pearson

I visited Sean Kim (2nd-year UCLA CS student, hired to help w/ QoT development & install procedure) last Friday. We successfully installed the QoT stack on one of my BBB's SD cards (we could run `qot-daemon`, `phc2phc`, and `helloworld`). 

The secret: "Having the BBBs pull their rootfs from the NUC's NFS causes trouble, you should install the QoT stack onto a micro SD card and run it over ssh."

We were basically following instructions from our project wiki:

https://bitbucket.org/rose-line/qot-stack

However, the "install to SD card" instructions are incomplete and several steps are out of order. Sean helped me figure out the right order.

I ran a screen-recorder to capture everything we did:

- `installing-qot-to-SD-card-BASIC-WORKS.ogv`
- `installing-qot.mov` (same movie, different format)
- on the UCLA NUC (I'm borrowing it)
- 4.5 hours long
- We took some wrong turns, so I don't recommend watching & executing from start to finish. 

What follows is a summary of the video -- how to install the QoT stack to an SD card, based on our experience. 

If you want more details, watch the video -- skipping to the right time of course (look at the screenshot's video timestamp in the lower-right).

## Get 8GB micro-SD card.

Hint: Can't tell which micro SD card to use? In the "old" days (with Andrew S, Oct 2015) we had partitions

- data
- rootfs
    - log
    - ssh
    - ssh_client
- BOOT

![](Images/2016-08-15-09-38-51.png "Optional title")



That's the kind you can wipe out.

The following format is more recent (July 2016). We used it when the "host" computer (the Intel NUC) is running an NFS that the BBB mounts its rootfs from. It has structure

- user
- rootfs
    - opensplice
    - ssh
- boot    
    - MLO
    - u-boot.img
    - uEnv.txt

![](Images/2016-08-15-09-41-41.png "Optional title")

Note: these are all 8GB cards. I think 4 GB cards won't be big enough.



## Download SD card image `roseline_bbb_v3.img`

- https://drive.google.com/file/d/0B5sYz4zKsYSaQk1MWTlicGdFcU0/view?usp=sharing
- 4GB.
- my NUC:~/qot-stack-2016-08-12/roseline_bbb_v3.img

## Prepare new SD card.

- Unmount SD card but do not eject. Eg, using `gparted`:

![](Images/2016-08-15-09-44-02.png "Optional title")

- Right-click disk image, "Open With Disk Image Writer"

![](Images/2016-08-15-09-45-21.png "Optional title")

![](Images/2016-08-15-09-47-50.png "Optional title")

- Takes 5 minutes.

![](Images/2016-08-15-09-49-59.png "Optional title")


## Boot BBB:

- Plug in SD card to BBB
- plug in FTDI-to-usb cable (black: pin 1, green: pin 4, white: pin 5, red: no connect!)
- Start minicom:

![](Images/2016-08-15-09-51-18.png "Optional title")

- Power up BBB: 

![](Images/2016-08-15-09-51-45.png "Optional title")

![](Images/2016-08-15-09-55-34.png "Optional title")

At this point you can interact w/ the BBB thru Minicom. But we should ssh into it instead: faster, doesn't require FTDI cable. And we'll need ethernet access to copy over files later.

## Set up networking

Previously I had followed these instructions 

https://bitbucket.org/rose-line/qot-stack#markdown-header-step-2-networking

to configure my NUC to run a DHCP server and assign IP addrs to the BBBs. 

Installing a DHCP server is a pain. You could probably statically assign an IP to the BBB. 

When you get it working, you can run `ifconfig` on the BBB thru Minicom and see that the NUC (host, DHCP server) assigned an IP to the BBB:

![](Images/2016-08-15-09-55-56.png "Optional title")

I skipped Step 4 from the instructions -- I could already `ssh root@10.42.0.126` without a password. Fine.

![](Images/2016-08-15-09-59-01.png "Optional title")


## Install a ton of QoT stuff and put it on the BBB

We're basically on Step 5 here:

![](Images/2016-08-15-10-01-20.png "Optional title")

This is where the instructions got "nonlinear" -- it turns out you have to download a ton of files, the Linux kernel, and so on, even though we hardly use them. Their paths (like `/export/rootfs/...`) are hard-coded into various make files, so the simplest thing is just to follow the instructions to download these huge files so that your file directories are what the makefiles expect :-/




### Get the QoT bundle (wget)

(What is it? Why is it like 500 MB? How different from the QoT src code, which we also download below?)

The following wget takes forever, so if you can, use a USB drive to get `qot-bundle.tar.bz2` from a friend.

    $> su -
    $> sudo mkdir /export
    $> cd /export
    $> wget https://bitbucket.org/rose-line/qot-stack/downloads/qot-bundle.tar.bz2
    $> tar -xjpf qot-bundle.tar.bz2
    $> rm -rf qot-bundle.tar.bz2


![](Images/2016-08-15-10-02-38.png "Optional title")

I forget why you need the QoT bundle. You shouldn't really need it, since the kernel & rootfs is baked into the SD card image. But some makefile fails if you don't have it. 


### Get the Linux kernel source (git clone)



A later step assumes that you have a linux source tree to copy into (`/export/bb-kernel/KERNEL`), and will fail if you don't have it, like this:


![](Images/2016-08-15-11-00-12.png "Optional title")

I guess it's needed to build the kernel modules? Dunno. 

Solution: Get the linux src, the `bb-kernel` directory, from git (or a friend, or a previous install):

![](Images/2016-08-15-11-00-12.png "Optional title")

If you can't cp it from somewhere, you'll have to just download it with git clone:

    $> su -
    $> cd /export
    $> git clone https://github.com/RobertCNelson/bb-kernel.git -b 4.1.12-bone-rt-r16


At the end of the previous 2 steps (QoT bundle and Linux kernel), , your `/export/` should look like:


![](Images/2016-08-15-10-30-46.png "Optional title")

Armed with `/export/bb-kernel/` and `/export/rootfs`, you're ready to build the actual QoT code.


### Get & build the QoT source (git clone)

We're basically following these instructions, with some modifications:

![](Images/2016-08-15-11-04-32.png "Optional title")

Specifically:

    $> git clone https://bitbucket.org/rose-line/qot-stack.git
    $> cd qot-stack
    $> git submodule init
    $> git submodule update

![](Images/2016-08-15-11-30-30.png "Optional title")


    $> pushd thirdparty/bb.org-overlays
    $> ./dtc-overlay.sh
    $> popd
    $> pushd thirdparty/opensplice
    $> git checkout -b v64 OSPL_V6_4_OSS_RELEASE
    $> git submodule init
    $> git submodule update
    $> ./configure

![](Images/2016-08-15-11-32-35.png "Optional title")

![](Images/2016-08-15-11-33-06.png "Optional title")


I see that my dtc is 

    dtc: Version: DT 1.4.1-g1e...

(is that good?)    




    $> . envs-armv7l.linux-dev.sh
    $> make
    $> make install

![](Images/2016-08-15-11-35-06.png "Optional title")

![](Images/2016-08-15-11-35-58.png "Optional title")



Now insert `-std=c++11` to the CPPFLAGS variable in ./install/HDE/armv7l.linux-dev/custom_lib/Makefile.Build_DCPS_ISO_Cpp_Lib file.

![](Images/2016-08-15-11-36-55.png "Optional title")


    $> pushd install/HDE/%build%/custom_lib
    $> make -f Makefile.Build_DCPS_ISO_Cpp_Lib

![](Images/2016-08-15-11-38-20.png "Optional title")

    $> popd
    $> popd
    $> mkdir -p build % Do this in the top most project directory /qot-stack %
    $> pushd build
    $> ccmake ..


![](Images/2016-08-15-11-44-24.png "Optional title")

Cmake is screwy: press 'c' for configure, then edit the menu items, then 'c' to configure again? Then 'g' to generate makefiles.

Had some weird Cmake error because we hadn't installed the QoT bundle above:

![](Images/2016-08-15-11-45-40.png "Optional title")

![](Images/2016-08-15-11-46-14.png "Optional title")

We needed the `/export/rootfs` dir that comes with teh QoT bundle. Also we needed the `/export/bb-kernel` that comes from the Linux kernel step. You wouldn't guess that from the error msg. I copied them over and essentially did all the steps over again, and it worked that time. Hopefully you started by intalling the QoT bundle so it should just work for you.







    $> make



![](Images/2016-08-15-11-49-46.png "Optional title")



Arg, "WARNING: "qot_register" undefined:

![](Images/2016-08-15-10-32-55.png "Optional title")

Fix it like this:

![](Images/2016-08-15-11-08-47.png "Optional title")

(From the wiki:)

You may get this error:

    WARNING: "qot_register" [/home/shk/bb/qot-stack/src/modules/qot_am335x/qot_am335x.ko] undefined!
    WARNING: "qot_unregister" [/home/shk/bb/qot-stack/src/modules/qot_am335x/qot_am335x.ko] undefined!

while compiling. This is due to module files being in different directories. Copy over Module.symvers file to fix this error.

    $> cp src/modules/qot/Module.symvers src/modules/qot_am335x/Module.symvers

Run make again and the warning should be gone.


![](Images/2016-08-15-12-00-15.png "Optional title")




    $> sudo make install
    $> popd



![](Images/2016-08-15-11-10-00.png "Optional title")


You can see it's installing to `/export/rootfs/usr/local/include/qot.h` and so on. It's silly because we're not using that dir. But whatever. That's why we needed to download teh QoT bundle, which creates that dir.


Now we edit the main Makefile to have the correct kernel ver & ip addr:

![](Images/2016-08-15-12-00-57.png "Optional title")

![](Images/2016-08-15-12-01-38.png "Optional title")


And finally we do 

    make install_sd

to copy some scripts onto the card:

![](Images/2016-08-15-12-02-06.png "Optional title")

![](Images/2016-08-15-12-02-42.png "Optional title")

![](Images/2016-08-15-12-02-54.png "Optional title")

Lastly, run `ldconfig` on the BBB. And maybe `depmod -a` too:

![](Images/2016-08-15-12-28-15.png "Optional title")

The important thing is that `lsmod` shows that the `qot` and `qot_am335x` kernel modules have been loaded.




## Running QoT tests

### `testptp`

![](Images/2016-08-15-10-43-38.png "Optional title")

It worked if you have "4 external time stamp channels" on ptp0 and "3 external time stamp channels" on ptp1. 

There's probably a simpler way to see if it worked, like check `/dev/ptp*` or something.

### `phc2phc`

This program sync's the ethernet PHY's clock (which timestamps incoming packets) with the CPU's clock (which is what everything else like CLOCK_REALTIME is timestamped with).

![](Images/2016-08-15-10-43-23.png "Optional title")

### `qotdaemon -v`

This program watches for new QoT timelines to be broadcast across the network.

![](Images/2016-08-15-10-44-03.png "Optional title")

### `helloworld`

The "hello world" of the QoT stack. This program makes a timeline then sleeps for 1 sec on that timeline.

![](Images/2016-08-15-10-44-56.png "Optional title")

![](Images/2016-08-15-10-46-34.png "Optional title")

I imagine that if you have other BBBs on the network, this test is a litle more interesting.

### Get other scripts on the BBB

Because this was the SD card installation, you have to `scp` them over. There is a makefile to do that for you:

![](Images/2016-08-15-10-48-45.png "Optional title")

![](Images/2016-08-15-10-49-44.png "Optional title")

![](Images/2016-08-15-10-49-50.png "Optional title")

