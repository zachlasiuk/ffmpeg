# ARM Migration Quick Reference
## FFmpeg Docker Images - Critical Changes Needed

---

## 🎯 Quick Summary

**Files to Change:** 3 source files (everything else auto-generates)
**Effort:** ~1-2 days
**Risk Level:** LOW ✅

---

## 📋 Checklist of x86-specific Issues

### Critical Issues (Must Fix)

- [ ] **update.py** (Line 307-310): Hardcoded `/opt/ffmpeg/lib/x86_64-linux-gnu`
- [ ] **update.py** (Line 325-328): Hardcoded `/usr/include/x86_64-linux-gnu` and `/usr/lib/x86_64-linux-gnu`
- [ ] **update.py** (Line 285-286): Hardcoded CUDA `lib64` and `lib32` paths
- [ ] **build_source.sh** (Line 77): libvpx uses `--as=yasm` (x86-only)
- [ ] **build_source.sh** (Line 89): libmp3lame uses `--enable-nasm` (x86-only)
- [ ] **build_source.sh** (Line 173): AOM uses `-DENABLE_NASM=on` (x86-only)
- [ ] **install_ffmpeg.sh** (Line 43-45): Hardcoded grep for `x86_64-linux-gnu`
- [ ] **install_ffmpeg.sh** (Line 47-49): Hardcoded CUDA path `/usr/local/cuda/targets/x86_64-linux/lib/`
- [ ] **install_ffmpeg.sh** (Line 81): Hardcoded pkgconfig path `${PREFIX}/lib/x86_64-linux-gnu/pkgconfig/`

### Already OK (No Changes Needed)

- ✅ **All Dockerfiles**: Use multi-arch base images (Ubuntu 24.04, Alpine 3.20, NVIDIA CUDA 12.6)
- ✅ **Environment Variables**: Already include both `x86_64-linux-gnu` and `aarch64-linux-gnu` in PKG_CONFIG_PATH
- ✅ **FFmpeg**: Compiles natively on ARM with all codecs

---

## 🔧 Architecture Detection Pattern

Add this to the beginning of all shell scripts:

```bash
# Detect system architecture
ARCH=$(uname -m)
case "$ARCH" in
    x86_64|amd64)
        MULTIARCH_TRIPLET="x86_64-linux-gnu"
        CUDA_TARGETS="x86_64-linux"
        USE_NASM="--enable-nasm"
        USE_YASM="--as=yasm"
        AOM_NASM="-DENABLE_NASM=on"
        ;;
    aarch64|arm64)
        MULTIARCH_TRIPLET="aarch64-linux-gnu"
        CUDA_TARGETS="sbsa-linux"  # NVIDIA ARM servers
        USE_NASM="--disable-nasm"
        USE_YASM=""  # Auto-detect (removes --as=yasm)
        AOM_NASM="-DENABLE_NASM=off"
        ;;
    *)
        echo "Unsupported architecture: $ARCH"
        exit 1
        ;;
esac
```

---

## 📝 Specific File Changes

### 1. **update.py** (Python Generator Script)

**Line 307-310** - Replace:
```python
# OLD:
FFMPEG_CONFIG_FLAGS.append(
    "--extra-ldflags=-L/opt/ffmpeg/lib/x86_64-linux-gnu"
)

# NEW:
import platform
arch = platform.machine()
multiarch = "x86_64-linux-gnu" if arch in ["x86_64", "AMD64"] else "aarch64-linux-gnu"
FFMPEG_CONFIG_FLAGS.append(
    f"--extra-ldflags=-L/opt/ffmpeg/lib/{multiarch}"
)
```

**Line 325-328** - Replace:
```python
# OLD:
if float(version[0:3]) >= 5.1:
    CFLAGS.append("-I/usr/include/x86_64-linux-gnu")
    LDFLAGS.append("-L/usr/lib/x86_64-linux-gnu")

# NEW:
if float(version[0:3]) >= 5.1:
    CFLAGS.append(f"-I/usr/include/{multiarch}")
    LDFLAGS.append(f"-L/usr/lib/{multiarch}")
```

**Line 285-286** - Replace:
```python
# OLD:
if variant["parent"] == "nvidia":
    LDFLAGS.append("-L/usr/local/cuda/lib64")
    LDFLAGS.append("-L/usr/local/cuda/lib32/")

# NEW:
if variant["parent"] == "nvidia":
    if arch in ["x86_64", "AMD64"]:
        LDFLAGS.append("-L/usr/local/cuda/lib64")
        LDFLAGS.append("-L/usr/local/cuda/lib32/")
    else:  # ARM64
        LDFLAGS.append("-L/usr/local/cuda/lib64")  # ARM also uses lib64
```

---

### 2. **build_source.sh** (Build Script Template)

**Line 77** - libvpx (Replace):
```bash
# OLD:
./configure  --prefix="${PREFIX}" --disable-examples --disable-unit-tests \
    --enable-vp9-highbitdepth --enable-pic --enable-shared --as=yasm

# NEW:
./configure  --prefix="${PREFIX}" --disable-examples --disable-unit-tests \
    --enable-vp9-highbitdepth --enable-pic --enable-shared ${USE_YASM}
```

**Line 89** - libmp3lame (Replace):
```bash
# OLD:
./configure --prefix="${PREFIX}" --bindir="${PREFIX}/bin" \
    --enable-shared --enable-nasm --disable-frontend

# NEW:
./configure --prefix="${PREFIX}" --bindir="${PREFIX}/bin" \
    --enable-shared ${USE_NASM} --disable-frontend
```

**Line 173** - AOM (Replace):
```bash
# OLD:
cmake -G "Unix Makefiles" -DCMAKE_INSTALL_PREFIX="${PREFIX}" \
    -DBUILD_SHARED_LIBS=1 -DENABLE_NASM=on ..

# NEW:
cmake -G "Unix Makefiles" -DCMAKE_INSTALL_PREFIX="${PREFIX}" \
    -DBUILD_SHARED_LIBS=1 ${AOM_NASM} ..
```

---

### 3. **install_ffmpeg.sh** (Install Script Template)

**Line 43-45** - Replace:
```bash
# OLD:
if ldd ${PREFIX}/bin/ffmpeg | grep x86_64-linux-gnu | cut -d ' ' -f 3 | grep -q . ; then
    ldd ${PREFIX}/bin/ffmpeg | grep x86_64-linux-gnu | cut -d ' ' -f 3 | xargs -i cp -p {} /usr/local/lib/
fi

# NEW:
if ldd ${PREFIX}/bin/ffmpeg | grep ${MULTIARCH_TRIPLET} | cut -d ' ' -f 3 | grep -q . ; then
    ldd ${PREFIX}/bin/ffmpeg | grep ${MULTIARCH_TRIPLET} | cut -d ' ' -f 3 | xargs -i cp -p {} /usr/local/lib/
fi
```

**Line 47-49** - Replace:
```bash
# OLD:
if [[ -d /usr/local/cuda/targets/x86_64-linux/lib/ ]]; then
    cp -p /usr/local/cuda/targets/x86_64-linux/lib/libnpp* /usr/local/lib
fi

# NEW:
if [[ -d /usr/local/cuda/targets/${CUDA_TARGETS}/lib/ ]]; then
    cp -p /usr/local/cuda/targets/${CUDA_TARGETS}/lib/libnpp* /usr/local/lib
fi
```

**Line 81** - Replace:
```bash
# OLD:
for pc in ${PREFIX}/lib/pkgconfig/libav*.pc ${PREFIX}/lib/pkgconfig/libpostproc.pc \
    ${PREFIX}/lib/pkgconfig/kvazaar.pc ${PREFIX}/lib/pkgconfig/libsw*.pc \
    ${PREFIX}/lib/x86_64-linux-gnu/pkgconfig/libvmaf*; do

# NEW:
for pc in ${PREFIX}/lib/pkgconfig/libav*.pc ${PREFIX}/lib/pkgconfig/libpostproc.pc \
    ${PREFIX}/lib/pkgconfig/kvazaar.pc ${PREFIX}/lib/pkgconfig/libsw*.pc \
    ${PREFIX}/lib/${MULTIARCH_TRIPLET}/pkgconfig/libvmaf*; do
```

---

## 🧪 Testing Commands

### Build Single Variant (Ubuntu)
```bash
# Native ARM build
docker build -t ffmpeg:8.0-ubuntu2404-arm64 \
    -f docker-images/8.0/ubuntu2404/Dockerfile .

# Cross-compile (from x86 to ARM)
docker buildx build --platform linux/arm64 \
    -t ffmpeg:8.0-ubuntu2404-arm64 \
    -f docker-images/8.0/ubuntu2404/Dockerfile .
```

### Build Multi-Arch Image
```bash
docker buildx build \
    --platform linux/amd64,linux/arm64 \
    -t ffmpeg:8.0-ubuntu2404 \
    -f docker-images/8.0/ubuntu2404/Dockerfile .
```

### Test FFmpeg Functionality
```bash
# Run container
docker run --rm -it ffmpeg:8.0-ubuntu2404-arm64 bash

# Inside container - verify architecture
uname -m  # Should show: aarch64

# Test ffmpeg
ffmpeg -version
ffmpeg -codecs | grep h264
ffmpeg -encoders | grep x264
```

---

## 📦 Package/Library ARM Compatibility

All dependencies already support ARM64:

| Library | ARM64 Support | Notes |
|---------|---------------|-------|
| x264 | ✅ Yes | Full ARM support with NEON optimizations |
| x265 | ✅ Yes | ARM NEON optimizations available |
| libvpx | ✅ Yes | Auto-detects ARM, has NEON code |
| aom (AV1) | ✅ Yes | ARM NEON optimizations built-in |
| libmp3lame | ✅ Yes | C fallback, no assembly needed |
| opus | ✅ Yes | ARM optimizations available |
| fdk-aac | ✅ Yes | Portable C code |
| libass | ✅ Yes | Pure C library |
| FFmpeg | ✅ Yes | Excellent ARM support with NEON |

**Note:** On ARM, NEON replaces SSE/AVX (x86 SIMD). Libraries auto-detect and use appropriate optimizations.

---

## 🚀 Build Workflow

### Step 1: Fix Source Files
1. Edit `update.py` - Add architecture detection
2. Edit `build_source.sh` - Add architecture detection and use variables
3. Edit `install_ffmpeg.sh` - Add architecture detection and use variables

### Step 2: Regenerate All Variants
```bash
python3 update.py
```
This will regenerate all files under `docker-images/8.0/*/`

### Step 3: Test One Variant
```bash
# Build ubuntu2404 variant
cd docker-images/8.0/ubuntu2404
docker buildx build --platform linux/arm64 -t test-arm64 .
```

### Step 4: Test All Variants
Build each variant: ubuntu2404, ubuntu2404-edge, alpine320, scratch320, vaapi2404, nvidia2404

### Step 5: Push Multi-Arch Images
```bash
docker buildx build \
    --platform linux/amd64,linux/arm64 \
    --push \
    -t jrottenberg/ffmpeg:8.0-ubuntu2404 \
    -f docker-images/8.0/ubuntu2404/Dockerfile .
```

---

## ⚠️ Special Considerations

### NVIDIA Variant (`nvidia2404`)
- **x86_64**: Full NVENC/NVDEC support
- **ARM64**: Works on NVIDIA Grace, Jetson platforms
- **CUDA Path**: Uses `sbsa-linux` instead of `x86_64-linux`
- **Recommendation**: Test on actual ARM64 NVIDIA hardware

### VAAPI Variant (`vaapi2404`)
- ✅ Works on ARM with supported GPUs
- ARM SoCs (Rockchip, Amlogic, Mali) support VAAPI
- No special changes needed beyond general ARM fixes

### Alpine/Scratch Variants
- ✅ Alpine fully supports ARM64
- musl libc works identically on ARM
- All apk packages available for aarch64

---

## 📊 Migration Impact

### What Changes:
- Build scripts detect architecture
- Library paths use variables instead of hardcoded x86_64
- Assembly tools disabled on ARM (libraries use NEON instead)

### What Stays the Same:
- All Dockerfiles (already use multi-arch base images)
- All libraries (already support ARM)
- Build process (same steps, just with different flags)
- Functionality (FFmpeg works identically)

### Performance:
- ARM64 can be **faster** than x86_64 for video encoding
- Modern ARM chips (Graviton, Ampere) excel at FFmpeg workloads
- NEON optimizations are highly efficient

---

## 🎓 Key Learnings

1. **Already Partially ARM-Aware**: Environment variables include aarch64 paths
2. **Main Issue**: Hardcoded paths in build logic, not in Dockerfiles
3. **Simple Fix**: Add architecture detection variable, replace hardcoded strings
4. **Low Risk**: All dependencies support ARM, Docker base images support ARM
5. **High Reward**: Unlocks AWS Graviton, ARM servers, Apple Silicon builds

---

## 🔗 Useful Resources

- [FFmpeg on ARM](https://trac.ffmpeg.org/wiki/CompilationGuide) - Official compilation guide
- [Docker Buildx](https://docs.docker.com/buildx/working-with-buildx/) - Multi-arch builds
- [libvpx ARM](https://chromium.googlesource.com/webm/libvpx) - ARM optimization notes
- [x264 ARM](https://code.videolan.org/videolan/x264) - NEON support documentation

---

## 💡 Pro Tips

1. **Use buildx for testing**: Faster than native ARM build for initial testing
2. **Check generated files**: After running `update.py`, verify generated scripts
3. **Test incrementally**: Build ubuntu2404 first (simplest), then others
4. **Docker cache**: Use BuildKit cache to speed up rebuilds
5. **Multi-stage builds**: Already implemented, works great for size optimization

---

**Last Updated:** 2024
**Next Review:** After first successful ARM build
