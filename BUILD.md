# Building Python 3.14 Free-Threading for Android

Complete guide to building Python 3.14 with `--disable-gil` for Android ARM64.

## Prerequisites

### System Requirements

- **OS:** Linux x86_64 (Ubuntu 20.04+ or Debian 11+)
- **RAM:** 8GB+ recommended
- **Disk space:** 20GB+ free
- **Build time:** ~15-20 minutes

### Required Packages

```bash
sudo apt-get update
sudo apt-get install -y \
    build-essential \
    git \
    wget \
    curl \
    python3 \
    python3-pip \
    openjdk-17-jdk
```

### Android SDK Command-line Tools

Download and setup Android SDK:

```bash
# Download command-line tools
mkdir -p ~/android-sdk
cd ~/android-sdk
wget https://dl.google.com/android/repository/commandlinetools-linux-11076708_latest.zip
unzip commandlinetools-linux-11076708_latest.zip
mkdir -p cmdline-tools/latest
mv cmdline-tools/* cmdline-tools/latest/ 2>/dev/null || true

# Set environment variable
export ANDROID_HOME=$HOME/android-sdk
export PATH=$ANDROID_HOME/cmdline-tools/latest/bin:$PATH

# Accept licenses
yes | sdkmanager --licenses
```

**Important:** Use absolute path for `ANDROID_HOME`, not `~` (tilde).

## Build Process

### Step 1: Download Python Source

```bash
cd ~
mkdir -p python-build && cd python-build
git clone --depth 1 --branch v3.14.0 https://github.com/python/cpython.git python-3.14.0
cd python-3.14.0
```

### Step 2: Build with android.py Script

Python 3.14 includes an official Android build script that handles everything:

```bash
cd Android
export ANDROID_HOME=/home/YOUR_USERNAME/android-sdk  # Use absolute path!
./android.py build aarch64-linux-android -- --disable-gil
```

**Key points:**
- The script will **automatically download Android NDK** (~1GB, takes 5-10 minutes)
- The `-- --disable-gil` flag enables free-threading
- Build takes approximately **15-20 minutes** on a modern machine
- Use `-j16` or similar for parallel build (automatic on most systems)

### Step 3: Monitor Build Progress

The build has 4 stages:

1. **Configure build Python** (2-3 min) - Python for the build machine
2. **Make build Python** (5-10 min) - Compile build Python
3. **Configure host Python** (1-2 min) - Cross-compile config for Android
4. **Make host Python** (1-2 min) - Build for Android ARM64

You'll see output like:
```
checking for --disable-gil... yes
checking ABIFLAGS... t
checking SOABI... cpython-314t-aarch64-linux-android
```

The `t` suffix confirms **free-threading is enabled**.

### Step 4: Verify Build Output

After successful build:

```bash
cd ~/python-build/python-3.14.0/cross-build/aarch64-linux-android/prefix

# Check files exist
ls -lh lib/libpython3.14t.so
ls -lh lib/python3.14t/
ls -lh include/python3.14t/

# Verify architecture
file lib/libpython3.14t.so
# Expected: ELF 64-bit LSB shared object, ARM aarch64
```

**Expected output:**
```
lib/libpython3.14t.so       31MB
lib/python3.14t/           233MB (before cleanup)
include/python3.14t/       2.6MB
```

## Cleaning the Build

The stdlib includes large test modules that aren't needed:

```bash
cd prefix/lib/python3.14t

# Remove test suite (saves 152MB!)
rm -rf test/

# Optional: Remove other development modules
rm -rf idlelib/      # 7.9MB - IDLE IDE
rm -rf tkinter/      # 1.5MB - GUI toolkit
rm -rf ensurepip/    # 1.8MB - pip installer
rm -rf pydoc_data/   # 2.2MB - documentation
rm -rf _pyrepl/      # 1.5MB - REPL
rm -rf turtledemo/   # 576KB - demos

# After cleanup: ~67MB
```

## Extract Binaries

Create a clean distribution:

```bash
cd ~/python-build/python-3.14.0/cross-build/aarch64-linux-android/prefix

mkdir -p ~/python-3.14-android-free-threading/binaries/{lib,include}

# Copy library
cp lib/libpython3.14t.so \
   ~/python-3.14-android-free-threading/binaries/lib/

# Copy stdlib (cleaned)
cp -r lib/python3.14t \
      ~/python-3.14-android-free-threading/binaries/lib/

# Copy headers
cp -r include/python3.14t \
      ~/python-3.14-android-free-threading/binaries/include/
```

## Testing on Device

### Push to Android Device

```bash
adb push binaries/lib/libpython3.14t.so /data/local/tmp/
adb push binaries/lib/python3.14t /data/local/tmp/

adb shell chmod +x /data/local/tmp/libpython3.14t.so
```

### Verify Free-Threading

```bash
# Check Python version
adb shell "cd /data/local/tmp && \
           LD_LIBRARY_PATH=. ./libpython3.14t.so -VV"
# Expected: "Python 3.14.0 free-threading build"

# Check GIL status
adb shell "cd /data/local/tmp && \
           LD_LIBRARY_PATH=. ./libpython3.14t.so -c \
           'import sys; print(\"GIL enabled:\", sys._is_gil_enabled())'"
# Expected: "GIL enabled: False"
```

## Build Configuration

The `android.py` script uses these settings:

```python
--host=aarch64-linux-android
--build=x86_64-pc-linux-gnu
--disable-gil              # FREE-THREADING!
--enable-shared
--with-build-python=...
--prefix=.../prefix
```

**Dependencies** (automatically handled):
- OpenSSL (SSL/TLS support)
- SQLite3 (database)
- libffi (foreign function interface)
- zlib, bz2, lzma (compression)

All downloaded from [BeeWare's mobile-forge](https://github.com/beeware/mobile-forge).

## Troubleshooting

### Issue: sdkmanager not found

**Problem:** `ANDROID_HOME` not set correctly or uses `~` tilde.

**Solution:**
```bash
# Use ABSOLUTE path
export ANDROID_HOME=/home/YOUR_USERNAME/android-sdk  # Not ~/android-sdk
```

### Issue: NDK download hanging

**Problem:** Large NDK download (~1GB) with no progress output.

**Solution:** Be patient. The script downloads NDK silently. Check with:
```bash
ls -lh $ANDROID_HOME/ndk/
```

### Issue: Configure fails

**Problem:** Missing dependencies or wrong architecture.

**Solution:**
```bash
# Check build Python exists
ls cross-build/build/python

# Verify environment
echo $ANDROID_HOME
which sdkmanager
```

### Issue: Build fails with "permission denied"

**Problem:** Missing execute permissions.

**Solution:**
```bash
chmod +x Android/android.py
```

## Build Environment Details

After successful build, you'll have:

- **NDK version:** 27.3.13750724 (auto-installed)
- **API level:** 24 (Android 7.0+)
- **Compiler:** Clang 18.0.4 (from NDK)
- **SOABI:** cpython-314t-aarch64-linux-android
- **ABI flags:** `t` (free-threading)

## Alternative: Manual Cross-Compilation

If you prefer manual control instead of `android.py`:

```bash
export ANDROID_NDK=$HOME/android-sdk/ndk/27.3.13750724
export TOOLCHAIN=$ANDROID_NDK/toolchains/llvm/prebuilt/linux-x86_64

export CC=$TOOLCHAIN/bin/aarch64-linux-android24-clang
export CXX=$TOOLCHAIN/bin/aarch64-linux-android24-clang++
export AR=$TOOLCHAIN/bin/llvm-ar
export RANLIB=$TOOLCHAIN/bin/llvm-ranlib

./configure \
    --host=aarch64-linux-android \
    --build=x86_64-linux-gnu \
    --disable-gil \
    --enable-shared \
    --with-build-python=$(which python3) \
    --prefix=$PWD/prefix

make -j$(nproc)
make install
```

**Note:** This requires manual dependency setup. The `android.py` script is recommended.

## Resources

- [CPython Android README](https://github.com/python/cpython/blob/main/Android/README.md)
- [PEP 779 - Free-Threading](https://peps.python.org/pep-0779/)
- [Python 3.14 Release Notes](https://docs.python.org/3.14/whatsnew/3.14.html)

## Questions?

Open an issue on GitHub if you encounter problems!

---

**Author:** [Fibogacci](https://fibogacci.com)
**Blog:** [AndroidPython.com](https://androidpython.com)
