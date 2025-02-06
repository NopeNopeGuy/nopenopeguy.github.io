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
export RBE_use_unified_downloads=true
export RBE_use_unified_uploads=true
export RBE_R8_EXEC_STRATEGY=local
export RBE_CXX_EXEC_STRATEGY=remote_local_fallback
export RBE_D8_EXEC_STRATEGY=remote_local_fallback
export RBE_JAVAC_EXEC_STRATEGY=remote_local_fallback
export RBE_JAR_EXEC_STRATEGY=remote_local_fallback
export RBE_ZIP_EXEC_STRATEGY=remote_local_fallback
export RBE_TURBINE_EXEC_STRATEGY=remote_local_fallback
export RBE_SIGNAPK_EXEC_STRATEGY=remote_local_fallback
export RBE_CXX_LINKS_EXEC_STRATEGY=remote_local_fallback
export RBE_ABI_LINKER_EXEC_STRATEGY=remote_local_fallback
export RBE_ABI_DUMPER_EXEC_STRATEGY=remote_local_fallback
export RBE_CLANG_TIDY_EXEC_STRATEGY=remote_local_fallback
export RBE_METALAVA_EXEC_STRATEGY=remote_local_fallback
export RBE_LINT_EXEC_STRATEGY=remote_local_fallback
export RBE_CLANG_TIDY=1
export RBE_ABI_LINKER=1
export RBE_ABI_DUMPER=1
export RBE_LINT=1
export RBE_JAVAC=1
export RBE_R8=1
export RBE_D8=1
export RBE_METALAVA=1
export RBE_ZIP=1
export RBE_TURBINE=1
export RBE_JAR=1
export RBE_CXX_LINKS=1
export RBE_SIGNAPK=1
export NINJA_REMOTE_NUM_JOBS=72
export RBE_JAVA_POOL=default
export RBE_METALAVA_POOL=default
export RBE_LINT_POOL=default
```

Most of these options (except R8, D8, JAVAC, CXX) are undocumented and were found by digging through AOSP code. Thanks Google.

You can increase NINJA_REMOTE_NUM_JOBS to 500 and even more if you have the ram for it, setting it to 128 should work fine completely for 16GB ram devices.

R8 currently doesn't work on BuildBuddy (just the remote build part, remote caching is working fine!), I have submitted an issue to reclient but haven't gotten a response yet.\
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
`sudo zramctl /dev/zram0 -s 48G` (Reduce this to 32G if you have a 16GB memory system)\
`sudo mkswap /dev/zram0`\
`sudo swapon /dev/zram0`

Note: Disk swap is not recommended as it will be **REALLY** Slow

# Results

## 16GB Ram, 8 cores (Credits to @armdebug)

### Specs:
![specs](https://github.com/user-attachments/assets/4cb45f91-d131-4c9f-ac59-b43dda3b2629)

### Build Time with just R8,D8,CXX,JAVAC remote build and caching:
![results1](https://github.com/user-attachments/assets/b42aec9f-89c8-4f87-b7bc-3e372713eab8)

### Build Time with everything cached except metalava:
![result2](https://github.com/user-attachments/assets/4a65667b-876d-4d28-b3ba-f3611414bda8)

## 8GB Ram, 4 cores

### Specs:
![specs2](https://github.com/user-attachments/assets/413a8def-2fe5-4d49-991b-0422fdbce007)


### Build Time with just R8,D8,CXX,JAVAC remote build and caching:
Greater than 5 hours (source: trust me)

### Build Time with everything cached (no exceptions):
4 hours 8 mins (source: trust me)
