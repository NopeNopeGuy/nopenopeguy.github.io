---
layout: default
---

# Building Android with BuildBuddy Remote Build Execution (RBE)

## Introduction

Remote Build Execution (RBE) allows you to leverage powerful remote servers to significantly speed up your Android builds. While Google's RBE service is commonly used, this guide focuses on using [BuildBuddy](https://buildbuddy.io/), a flexible alternative.  We'll walk through setting up RBE with LineageOS as an example, but the principles apply to most AOSP-based ROMs.

**Requirements:**

*   Fast internet connection (at least 40 Mbps).
*   Sufficient storage space (at least 100GB). BTRFS with compression is highly recommended (see Tips and Tricks).
*   Arch Linux (or a similar distribution). WSL2 *may* work, but is not officially tested.

## Setup

### 1. Install Dependencies

Install the necessary build tools and an AUR helper (if you don't have one already):

```bash
git clone https://aur.archlinux.org/yay-bin.git
cd yay-bin
makepkg -si
yay -S lineageos-devel
```

### 2. Clone LineageOS

Initialize and sync the LineageOS source code (or your chosen ROM):

```bash
repo init -u https://github.com/LineageOS/android.git -b lineage-21.0 --git-lfs --depth=1
repo sync -j$(nproc --all) -c
```

**Tip:**  See the "Tips and Tricks" section for ways to reduce storage usage. Using `--depth=1` in the `repo init` significantly reduces the initial download size.

## Setting up RBE with BuildBuddy

### 1. Obtain Your BuildBuddy API Key

1.  **Sign In:** Go to [buildbuddy.io](https://buildbuddy.io/) and sign in with your Google or GitHub account.
2.  **Quickstart:** Navigate to the "Quickstart" section in your BuildBuddy dashboard.
3.  **Copy API Key:**  You'll see a command similar to this:

    ```bash
    build --bes_backend=grpcs://your-instance.buildbuddy.io 
    build --remote_header=x-buildbuddy-api-key=xxx ...
    ```

    *   Copy the entire string *after* `--remote_header=`. This is your API key.  It will look like `x-buildbuddy-api-key=xxx`.
    *   Note the address after `--bes_backend=`. You only need the part *after* `grpcs://`, e.g., `your-instance.buildbuddy.io`.

### 2. Download and Configure `reclient`

1.  **Download:** Download the `reclient` package from [this link](https://chrome-infra-packages.appspot.com/p/infra/rbe/client/linux-amd64/+/latest).  This is a pre-built version of Google's `reclient` tool.
2.  **Extract:** Extract the downloaded archive to a directory within your ROM's source tree.  For example, you could create a directory called `rbe` in the root of your AOSP directory and extract it there.  It is important to keep the extracted directory structure.
3.  **Note Path:**  Remember the *relative* path from your AOSP root to this directory (e.g., `rbe`) or the absolute path.

### 3. Set Environment Variables

These environment variables configure `reclient` to use BuildBuddy.  It's highly recommended to add these to your `build/envsetup.sh` file so they are automatically set each time you initialize your build environment.

**IMPORTANT SEE NOTES BEFORE YOU DO THIS**

```bash
# --- Enable RBE and General Settings ---
export USE_RBE=1                                      
export RBE_DIR="path/to/reclient"                      # Path to the extracted reclient directory (relative or absolute)
export NINJA_REMOTE_NUM_JOBS=72                        # Number of parallel remote jobs (adjust based on your RAM, buildbuddy has 80 CPU cores in the free tier)

# --- BuildBuddy Connection Settings ---
export RBE_service="your-instance.buildbuddy.io:443"        # BuildBuddy instance address (without grpcs://, add the port 443)
export RBE_remote_headers="x-buildbuddy-api-key=xxx"    # Your BuildBuddy API key
export RBE_use_rpc_credentials=false                   
export RBE_service_no_auth=true                       

# --- Unified Downloads/Uploads (Recommended) ---
export RBE_use_unified_downloads=true
export RBE_use_unified_uploads=true

# --- Execution Strategies (remote_local_fallback is generally best) ---
export RBE_R8_EXEC_STRATEGY=remote_local_fallback
export RBE_D8_EXEC_STRATEGY=remote_local_fallback
export RBE_JAVAC_EXEC_STRATEGY=remote_local_fallback
export RBE_JAR_EXEC_STRATEGY=remote_local_fallback
export RBE_ZIP_EXEC_STRATEGY=remote_local_fallback
export RBE_TURBINE_EXEC_STRATEGY=remote_local_fallback
export RBE_SIGNAPK_EXEC_STRATEGY=remote_local_fallback
export RBE_CXX_EXEC_STRATEGY=remote_local_fallback    # Important see below.
export RBE_CXX_LINKS_EXEC_STRATEGY=remote_local_fallback
export RBE_ABI_LINKER_EXEC_STRATEGY=remote_local_fallback
export RBE_ABI_DUMPER_EXEC_STRATEGY=    # Will make build slower, by a lot. Keeping this for documentation
export RBE_CLANG_TIDY_EXEC_STRATEGY=remote_local_fallback
export RBE_METALAVA_EXEC_STRATEGY=remote_local_fallback
export RBE_LINT_EXEC_STRATEGY=remote_local_fallback

# --- Enable RBE for Specific Tools ---
export RBE_R8=1
export RBE_D8=1
export RBE_JAVAC=1
export RBE_JAR=1
export RBE_ZIP=1
export RBE_TURBINE=1
export RBE_SIGNAPK=1
export RBE_CXX_LINKS=1
export RBE_CXX=1
export RBE_ABI_LINKER=1
export RBE_ABI_DUMPER=    # Will make build slower, by a lot. Keeping this for documentation
export RBE_CLANG_TIDY=1
export RBE_METALAVA=1
export RBE_LINT=1

# --- Resource Pools ---
export RBE_JAVA_POOL=default
export RBE_METALAVA_POOL=default
export RBE_LINT_POOL=default
```

*   **`USE_RBE=1`**: Enables RBE.
*   **`RBE_DIR`**: The path to your extracted `reclient` directory.
*   **`RBE_service`**: The BuildBuddy instance address (without `grpcs://`, add the port 443).
*   **`RBE_remote_headers`**: Your BuildBuddy API key.
*   **`*_EXEC_STRATEGY`**:  Controls how different build steps are handled. `remote_local_fallback` means try remotely first, then fall back to local execution if the remote execution fails.
*   **`RBE_*=1`**: Enables RBE for specific build tools.
*   **`NINJA_REMOTE_NUM_JOBS`**:  The number of parallel jobs to run remotely. Start with 72 and increase if you have more RAM.  128 should be safe for 16GB RAM systems.  You can go higher (e.g., 500) if you have significantly more RAM.
*   **`RBE_*_POOL`**: Specifies the resource pool to use. The `default` pool is usually sufficient.

**Important Notes:**

*   Make sure to switch **`RBE_CXX_LINKS_EXEC_STRATEGY`** to **`local`** after your first build is done to reduce build times.
*   Many of these options are not officially documented by Google and were discovered through AOSP source code analysis.

## Building

Once you've set up RBE, you can build your ROM as usual:

```bash
source build/envsetup.sh
breakfast your_device  # Replace 'your_device' with your device codename
mka bacon -j$(nproc --all)
```

The `mka` command is the recommended way to build.  `-j$(nproc --all)` uses all available CPU cores for the local build steps. RBE will handle the remote execution according to your configuration.

## Tips and Tricks

### 1. Reducing Storage Requirements with BTRFS Compression

BTRFS compression can dramatically reduce the storage space needed for AOSP builds.

1.  **Create a BTRFS Filesystem:**

    ```bash
    sudo mkfs.btrfs /dev/your_storage_partition  # Replace with your actual partition
    ```

2.  **Mount with Compression:**

    ```bash
    sudo mount /dev/your_storage_partition /mnt/aosp -o defaults,noatime,compress-force=zstd,space_cache=v2,commit=120
    ```
    Replace `/mnt/aosp` with your desired mount point.

3.  **fstab Entry (Optional):**  For persistent mounting, add a line to your `/etc/fstab`:

    ```
    /dev/your_storage_partition /mnt/aosp btrfs defaults,noatime,compress-force=zstd,space_cache=v2,commit=120 0 0
    ```

**Benefits:** This can reduce storage usage by around 50% (e.g., a 195GB build might only take up 101GB).

**Example Compression Statistics:**
```
Processed 1682984 files, 1779448 regular extents (1841256 refs), 975364 inline.
Type       Perc     Disk Usage   Uncompressed Referenced
TOTAL       54%      101G         186G         195G
none       100%       48G          48G          51G
zstd        37%       52G         137G         143G
```
**Note:** You *might* be able to achieve even greater compression with `dwarfs`, but this is more complex to set up.

### 2. Working on Low-RAM Machines with zram

AOSP builds typically require 32GB+ of RAM.  You can use `zram` (compressed RAM) to build on systems with less physical RAM (e.g., 8GB).

1.  **Load zram Module:**

    ```bash
    sudo modprobe zram
    ```

2.  **Configure zram:**

    ```bash
    sudo swapoff /dev/zram0  # Only if you have existing zram
    sudo zramctl /dev/zram0 -s 48G  # 48GB for 8GB RAM systems, 32GB for 16GB RAM
    sudo mkswap /dev/zram0
    sudo swapon /dev/zram0
    ```

**Important:**  Using disk swap (a swap file or partition) is *not* recommended. It will be extremely slow.  zram is significantly faster because it uses compressed RAM.

## Results

These results demonstrate the potential build time improvements with RBE.

### System 1: 16GB RAM, 8 Cores

*   **Specs:** AMD EPYC 9634 8-Core, 16GB RAM, KVM Server

    ![specs](https://github.com/user-attachments/assets/4cb45f91-d131-4c9f-ac59-b43dda3b2629)\
    *System 1 Specifications (fastfetch output): AMD EPYC 9634 8-Core CPU, 16GB RAM, KVM Server, Debian GNU/Linux 12 (bookworm).*
*   **Build Time (R8, D8, CXX, JAVAC remote build and caching):**

    ![results1](https://github.com/user-attachments/assets/b42aec9f-89c8-4f87-b7bc-3e372713eab8)\
    *Image showing build time of approximately 2 hour 30 minutes with R8, D8, CXX, and JAVAC using remote build and caching.*
*   **Build Time (Everything cached except metalava):**

    ![result2](https://github.com/user-attachments/assets/4a65667b-876d-4d28-b3ba-f3611414bda8)\
    *System 1 Build Time (everything): 1 hour 27 minutes 43.496 seconds. RBE Stats: down 33.48 GB, up 14.86 GB, 90889 cache hits, 4289 remote executions, 121 local fallbacks.*

### System 2: 8GB RAM, 4 Cores

*   **Specs:** Intel Core i5-6500T, 8GB RAM, 48GB ZRAM, BTRFS, CachyOS

    ![specs2](https://github.com/user-attachments/assets/413a8def-2fe5-4d49-991b-0422fdbce007)
    *System 2 Specifications (fastfetch output): Intel Core i5-6500T CPU, 8GB RAM, 48GB ZRAM, CachyOS x86_64, BTRFS filesystem.*

*   **Build Time (R8, D8, CXX, JAVAC remote build and caching):**  > 5 hours
*   **Build Time (Everything cached):** ~ 4 hours 8 minutes

## Contact

If you encounter any issues, please contact @NopeNopeGuy on Telegram.
