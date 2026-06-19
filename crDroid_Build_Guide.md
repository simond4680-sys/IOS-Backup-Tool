# crDroid Build Guide (Android 16 / branch `16.0`)
## 1) Overview
This document provides an end-to-end reference for building crDroid from source:
- Environment setup
- Source initialization
- Local manifest template for generic devices
- Build execution
- Build verification
- Troubleshooting
- Clean build procedures

All command examples in this guide assume a Linux shell (bash/zsh) inside a Linux build environment.

## 2) Prerequisites
### Recommended hardware
- CPU: 8+ cores
- RAM: 16 GB minimum (32 GB recommended)
- Storage: 250+ GB free (SSD recommended)

### OS
- Linux environment required for Android ROM builds (native Linux, VM, or WSL2 Ubuntu).

## 3) Environment Setup
Install build dependencies:

```bash
sudo apt update
sudo apt install -y bc bison build-essential ccache curl flex g++-multilib gcc-multilib git git-lfs gnupg gperf imagemagick lib32ncurses-dev lib32readline-dev lib32z1-dev liblz4-tool libncurses6 libncurses-dev libsdl1.2-dev libssl-dev libwxgtk3.2-dev libxml2 libxml2-utils lzop pngcrush rsync schedtool squashfs-tools xsltproc zip zlib1g-dev
```

Install `repo`:

```bash
mkdir -p ~/bin
echo 'export PATH=~/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod a+x ~/bin/repo
```

## 4) Initialize crDroid Source
```bash
mkdir -p ~/crDroid
cd ~/crDroid
repo init -u https://github.com/crdroidandroid/android.git -b 16.0 --git-lfs --no-clone-bundle
```

## 5) Local Manifests (Generic Device Template)
If your target device repos are not fully covered in the main manifest:

```bash
mkdir -p ~/crDroid/.repo/local_manifests
```

Create `.repo/local_manifests/<device>.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<manifest>
  <!-- Device tree -->
  <project
    name="{{your_org}}/android_device_{{oem}}_{{device_codename}}"
    path="device/{{oem}}/{{device_codename}}"
    remote="github"
    revision="16.0" />

  <!-- Common device tree (optional) -->
  <project
    name="{{your_org}}/android_device_{{oem}}_{{common_codename}}"
    path="device/{{oem}}/{{common_codename}}"
    remote="github"
    revision="16.0" />

  <!-- Kernel source -->
  <project
    name="{{your_org}}/android_kernel_{{oem}}_{{soc_or_device}}"
    path="kernel/{{oem}}/{{soc_or_device}}"
    remote="github"
    revision="16.0" />

  <!-- Vendor blobs (optional if not already included upstream) -->
  <project
    name="{{your_org}}/proprietary_vendor_{{oem}}"
    path="vendor/{{oem}}"
    remote="github"
    revision="16.0" />
</manifest>
```

## 6) Sync Source
```bash
cd ~/crDroid
repo sync --force-sync --no-tags --no-clone-bundle -j$(nproc --all)
```

Note: `--force-sync` can discard divergent local checkout states in synced projects. Use it intentionally.

## 7) Build Execution
Run build with log capture:

```bash
cd ~/crDroid
export USE_CCACHE=1
ccache -M 100G
source build/envsetup.sh
brunch <device_codename> 2>&1 | tee build.log
```

Build log location:
- `~/crDroid/build.log`

Fallback build flow (if `brunch` is not available in your tree):

```bash
cd ~/crDroid
source build/envsetup.sh
lunch <product>-userdebug
m bacon 2>&1 | tee build.log
```

## 8) Verify Build Success
### Success criteria
- `brunch <device_codename>` exits successfully (exit code `0`)
- No fatal `FAILED:` errors at end of build
- Artifacts appear in `out/target/product/<device_codename>/`
- Final flashable zip exists (`crDroid*.zip`)

### Verification commands
```bash
ls -lh out/target/product/<device_codename>/
ls -lt out/target/product/<device_codename>/crDroid*.zip
sha256sum out/target/product/<device_codename>/crDroid*.zip
```

## 9) Troubleshooting Common Build Errors
### Standard triage flow
```bash
cd ~/crDroid
repo sync --force-sync --no-tags --no-clone-bundle -j$(nproc --all)
source build/envsetup.sh
brunch <device_codename> 2>&1 | tee build.log
grep -nE "FAILED:|error:|ninja: build stopped" build.log
```

Always fix the **first real error** (later errors are often cascades).

### Frequent failures and fixes
- **`repo sync` fetch/network failures**
  - Retry with lower parallelism:
  ```bash
  repo sync --force-sync --no-tags --no-clone-bundle -j1
  ```
- **Missing module / `No rule to make target`**
  - Usually missing or incorrect local manifest entries.
  - Verify device/kernel/vendor repos and branch revision consistency.
- **Generic `ninja: build stopped: subcommand failed`**
  - Wrapper error only; inspect first compiler/linker error above it.
- **OOM / process `Killed`**
  - Reduce build job count:
  ```bash
  export NINJA_ARGS="-j8 -l8"
  brunch <device_codename>
  ```
- **Disk/inode exhaustion**
  ```bash
  df -h
  df -i
  ```

### Last-resort source reset
```bash
cd ~/crDroid
repo forall -vc "git reset --hard && git clean -fdx"
repo sync --force-sync --no-tags --no-clone-bundle -j$(nproc --all)
```

## 10) Clean Build Procedures
Apply cleanup in this order (least destructive to most destructive).

### 10.1 Light clean (recommended first)
```bash
cd ~/crDroid
source build/envsetup.sh
m installclean
brunch <device_codename>
```

### 10.2 Device-output clean
```bash
cd ~/crDroid
rm -rf out/target/product/<device_codename>
source build/envsetup.sh
brunch <device_codename>
```

### 10.3 Full output clean
```bash
cd ~/crDroid
rm -rf out
source build/envsetup.sh
brunch <device_codename>
```

### 10.4 Deep cache clean (`ccache`)
```bash
ccache -C
ccache -M 100G
cd ~/crDroid
source build/envsetup.sh
brunch <device_codename>
```

### 10.5 Recommended recovery order
1. Re-sync + rebuild
2. `m installclean`
3. Delete `out/target/product/<device_codename>`
4. Delete full `out/`
5. Clear `ccache`
6. Hard reset repos + full sync

## 11) Quick Daily Build Loop
```bash
cd ~/crDroid
repo sync --force-sync --no-tags --no-clone-bundle -j$(nproc --all)
source build/envsetup.sh
brunch <device_codename>
ls -lt out/target/product/<device_codename>/crDroid*.zip
```
