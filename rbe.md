---
layout: default
---
# Building Android using Buildbuddy RBE

## Introduction

RBE stands for Remote Build Execution, android is buildable using Google's RBE service but we are not just limited to that, in this guide you'll learn how to use other RBE services for AOSP Building.

This guide will assume that you are using Archlinux, it may or may not work with WSL2.

Note: You will need fast (40Mbps minimum) internet and atleast 20GB per build. 

## Setup

Installing all the dependencies:

`git clone https://aur.archlinux.org/yay-bin.git`\
`cd yay-bin && makepkg -si`\
`yay -S lineageos-devel`

This will download all the dependencies needed and also install an AUR helper. Skip installing the aur helper if you've already installed one.

## Cloning LineageOS

I'll be using LineageOS as an example but it should work with most roms out there.

`repo init -u https://github.com/LineageOS/android.git -b lineage-21.0 --git-lfs --depth=1`\
`repo sync -j$(nproc --all) -c`

See `Tips and Tricks` to reduce storage utilization.

## Setting up RBE

```
export USE_RBE=1
export RBE_use_rpc_credentials=false
export RBE_service=remote.buildbuddy.io:443
export RBE_service_no_auth=true
export RBE_R8_EXEC_STRATEGY=local
export RBE_CXX_EXEC_STRATEGY=remote_local_fallback
export RBE_D8_EXEC_STRATEGY=remote
export RBE_JAVAC_EXEC_STRATEGY=remote
export RBE_JAVAC=1
export RBE_R8=1
export RBE_D8=1
export NINJA_REMOTE_NUM_JOBS=48
export RBE_JAVA_POOL=default
```

R8 currently doesn't work on BuildBuddy, I am working on a fix but that'll take time, so until then R8 will be run locally.\
I recommend putting this in your `build/envsetup.sh` so you don't have copy this everytime

## Building

You should just be able to build normally after this:\
`breakfast (your device)`\
`mka bacon -j$(nproc --all)`

That should be all you need to do in order to get a successful build, contact me on @NopeNopeGuy on telegram if you have any issues


# Tips and Tricks

## Reducing Storage Requirements

You should be able to reduce the space used by using BTRFS compression. Create a BTRFS filesystem using:\
`mkfs.btrfs /dev/(your storage partition here)`

Mount your btrfs partition using these options for the best compression: \
`mount /dev/(your storage partition) /(where you want it mounted) -o defaults,noatime,compress-force=zstd,space_cache=v2,commit=120`

Here is a reference fstab entry:\
`/dev/sda1 /home/user/LineageOS btrfs defaults,noatime,compress-force=zstd,space_cache=v2,commit=120 0 0`

Using this you should be able to reduce the space required by around 50% using this (Around 100GB with a full build).

Stats:
```
Processed 1682984 files, 1779448 regular extents (1841256 refs), 975364 inline.
Type       Perc     Disk Usage   Uncompressed Referenced  
TOTAL       54%      101G         186G         195G       
none       100%       48G          48G          51G       
zstd        37%       52G         137G         143G       
```

You may also be able to reduce the space used even more if you use dwarfs, but that is left as an exercise for the reader.

## Working on Low Ram Machines

AOSP needs atleast 32GiB of RAM to build, but we can reduce that requirement to 8GB using a lot of zram, here's how to do it:

`sudo modprobe zram` (If it isn't loaded already)\
`sudo swapoff /dev/zram0` (If you have zram already, else skip this)\
`sudo zramctl /dev/zram0 -s 32G` (Reduce this to 24G if you have a 16GB memory system)\
`sudo mkswap /dev/zram0`\
`sudo swapon /dev/zram0`

Note: Disk swap is not recommended as it will be **REALLY** Slow
