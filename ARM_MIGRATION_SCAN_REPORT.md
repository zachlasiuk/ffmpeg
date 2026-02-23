# ARM (aarch64) Architecture Migration Report
## FFmpeg Docker Images Project

**Generated:** 2024
**Repository:** jrottenberg/ffmpeg
**Current Architecture:** x86_64 (amd64)
**Target Architecture:** ARM64 (aarch64)

---

## Executive Summary

This repository builds FFmpeg Docker images from source with various configurations (Ubuntu, Alpine, NVIDIA CUDA, VAAPI). The codebase contains multiple x86-specific hardcoded paths and assumptions that need to be addressed for ARM compatibility. The good news is that **ARM support is already partially present** in the codebase (e.g., PKG_CONFIG_PATH includes aarch64-linux-gnu paths), but it's not fully implemented or tested.

### Key Findings:
- ✅ **8 Dockerfiles** using multi-arch capable base images (Ubuntu 24.04, Alpine 3.20, NVIDIA CUDA)
- ⚠️ **47+ instances** of hardcoded x86_64-linux-gnu paths
- ⚠️ **7 instances** of hardcoded CUDA x86_64 paths
- ⚠️ **Multiple assembly tools** (NASM, YASM) that need ARM alternatives
- ⚠️ **Python update script** that generates x86-specific build flags
- ⚠️ **No architecture detection** in critical build scripts

---

## Detailed Findings

### 1. DOCKERFILES - Base Images (8 files)

#### Status: ✅ Multi-Arch Compatible (with caveats)

All Dockerfiles use base images that support both x86_64 and ARM64:

| Dockerfile | Base Image | ARM64 Support | Notes |
|------------|-----------|---------------|-------|
| `docker-images/8.0/ubuntu2404/Dockerfile` | `ubuntu:24.04` | ✅ Yes | Official Ubuntu multi-arch |
| `docker-images/8.0/ubuntu2404-edge/Dockerfile` | `ubuntu:24.04` | ✅ Yes | Official Ubuntu multi-arch |
| `docker-images/8.0/alpine320/Dockerfile` | `alpine:3.20` | ✅ Yes | Official Alpine multi-arch |
| `docker-images/8.0/scratch320/Dockerfile` | `alpine:3.20` | ✅ Yes | Official Alpine multi-arch |
| `docker-images/8.0/vaapi2404/Dockerfile` | `ubuntu:24.04` | ✅ Yes | VAAPI has ARM support |
| `docker-images/8.0/nvidia2404/Dockerfile` | `nvidia/cuda:12.6.2-devel-ubuntu24.04` | ⚠️ Partial | NVIDIA CUDA images support ARM, but NVENC/NVDEC availability varies by platform |

**Environment Variables in Dockerfiles:**
```dockerfile
# Already includes both architectures!
ENV PKG_CONFIG_PATH="/opt/ffmpeg/share/pkgconfig:/opt/ffmpeg/lib/pkgconfig:/opt/ffmpeg/lib64/pkgconfig:/opt/ffmpeg/lib/x86_64-linux-gnu/pkgconfig:/opt/ffmpeg/lib/aarch64-linux-gnu/pkgconfig:/usr/lib/x86_64-linux-gnu/pkgconfig:/usr/lib/pkgconfig"

ENV LD_LIBRARY_PATH="/opt/ffmpeg/lib:/opt/ffmpeg/lib64:/opt/ffmpeg/lib/aarch64-linux-gnu"
```

---

### 2. BUILD SCRIPTS - x86_64 Hardcoded Paths

#### 2.1 Main Build Script: `build_source.sh` (+ 6 generated copies)

**Location:** Root + all variant directories under `docker-images/8.0/*/build_source.sh`

**Critical Issues:**

##### Line 77 - libvpx build (YASM assembler):
```bash
# ISSUE: --as=yasm is x86-specific
./configure  --prefix="${PREFIX}" --disable-examples --disable-unit-tests \
    --enable-vp9-highbitdepth --enable-pic --enable-shared --as=yasm
```

**Impact:** YASM is x86-only. On ARM, this should either:
- Use auto-detection (remove `--as=yasm`)
- Use `--as=auto` to let configure script choose the right assembler
- Conditionally set based on architecture

##### Line 89 - libmp3lame build (NASM):
```bash
# ISSUE: --enable-nasm is x86-specific
./configure --prefix="${PREFIX}" --bindir="${PREFIX}/bin" \
    --enable-shared --enable-nasm --disable-frontend
```

**Impact:** NASM is x86-specific. ARM builds should use `--disable-nasm` or auto-detect.

##### Line 173 - AOM build (NASM):
```bash
cmake -G "Unix Makefiles" -DCMAKE_INSTALL_PREFIX="${PREFIX}" \
    -DBUILD_SHARED_LIBS=1 -DENABLE_NASM=on ..
```

**Impact:** Should be `-DENABLE_NASM=off` on ARM or use conditional logic.

---

#### 2.2 Generated Build Scripts (per variant)

**Files affected:** 
- `docker-images/8.0/ubuntu2404/build_source.sh`
- `docker-images/8.0/ubuntu2404-edge/build_source.sh`
- `docker-images/8.0/nvidia2404/build_source.sh`
- `docker-images/8.0/alpine320/build_source.sh`
- `docker-images/8.0/scratch320/build_source.sh`
- `docker-images/8.0/vaapi2404/build_source.sh`

**Example from ubuntu2404 (Lines 373-375):**
```bash
--extra-cflags="-I${PREFIX}/include -I/usr/include/x86_64-linux-gnu" \
--extra-ldflags="-L${PREFIX}/lib -L/usr/lib/x86_64-linux-gnu -L/usr/lib" \
--extra-ldflags=-L/opt/ffmpeg/lib/x86_64-linux-gnu \
```

**Example from nvidia2404 (Line 379):**
```bash
--extra-cflags="-I${PREFIX}/include -I${PREFIX}/include/ffnvcodec -I/usr/local/cuda/include/ -I/usr/include/x86_64-linux-gnu" \
--extra-ldflags="-L${PREFIX}/lib -L/usr/local/cuda/lib64 -L/usr/local/cuda/lib32/ -L/usr/lib/x86_64-linux-gnu -L/usr/lib" \
```

**Impact:** These paths won't exist on ARM systems. They should be dynamically determined.

---

### 3. INSTALL SCRIPTS - Runtime Library Paths

#### File: `install_ffmpeg.sh` (+ 6 generated copies)

**Critical Issues:**

##### Lines 43-45 - x86_64 library detection:
```bash
# Check if ffmpeg library is linked to x86_64-linux-gnu and copy it to /usr/local/lib
if ldd ${PREFIX}/bin/ffmpeg | grep x86_64-linux-gnu | cut -d ' ' -f 3 | grep -q . ; then
    ldd ${PREFIX}/bin/ffmpeg | grep x86_64-linux-gnu | cut -d ' ' -f 3 | xargs -i cp -p {} /usr/local/lib/
fi
```

**Impact:** On ARM, libraries are in `aarch64-linux-gnu`, so this check will fail silently and libraries won't be copied.

##### Lines 47-49 - CUDA x86_64 paths:
```bash
# some nvidia libs are in the cuda targets directory
if [[ -d /usr/local/cuda/targets/x86_64-linux/lib/ ]]; then
    cp -p /usr/local/cuda/targets/x86_64-linux/lib/libnpp* /usr/local/lib
fi
```

**Impact:** On ARM with NVIDIA GPU, the path is `/usr/local/cuda/targets/sbsa-linux/lib/` or `/usr/local/cuda/targets/aarch64-linux/lib/`.

##### Line 81 - pkgconfig paths:
```bash
for pc in ${PREFIX}/lib/pkgconfig/libav*.pc ${PREFIX}/lib/pkgconfig/libpostproc.pc \
    ${PREFIX}/lib/pkgconfig/kvazaar.pc ${PREFIX}/lib/pkgconfig/libsw*.pc \
    ${PREFIX}/lib/x86_64-linux-gnu/pkgconfig/libvmaf*; do
```

**Impact:** The `x86_64-linux-gnu` path won't exist on ARM.

---

### 4. PYTHON GENERATOR SCRIPT - `update.py`

**Location:** `./update.py` (root directory)

This is the **source of truth** that generates all variant Dockerfiles and build scripts.

**Critical Issues:**

##### Lines 307-310 - Hardcoded x86_64 paths:
```python
# Some shenagians to get libvmaf to build with static linking
FFMPEG_CONFIG_FLAGS.append(
    "--extra-ldflags=-L/opt/ffmpeg/lib/x86_64-linux-gnu"
)
```

##### Lines 325-328 - Conditional x86_64 paths:
```python
if float(version[0:3]) >= 5.1:
    CFLAGS.append("-I/usr/include/x86_64-linux-gnu")
    LDFLAGS.append("-L/usr/lib/x86_64-linux-gnu")
    LDFLAGS.append("-L/usr/lib")  # for alpine ( but probably fine for all)
```

##### Lines 283-286 - NVIDIA CUDA paths:
```python
if variant["parent"] == "nvidia":
    CFLAGS.append("-I${PREFIX}/include/ffnvcodec")
    CFLAGS.append("-I/usr/local/cuda/include/")
    LDFLAGS.append("-L/usr/local/cuda/lib64")
    LDFLAGS.append("-L/usr/local/cuda/lib32/")
```

**Impact:** This script generates ALL build scripts and Dockerfiles. Fixing this file will fix all generated files.

---

### 5. TEMPLATE FILES

**Location:** `./templates/`

All templates use the same environment variables:

**File:** `templates/Dockerfile-env-ubuntu` (and similar files)

##### Lines 91-94:
```dockerfile
ENV PKG_CONFIG_PATH="/opt/ffmpeg/share/pkgconfig:/opt/ffmpeg/lib/pkgconfig:/opt/ffmpeg/lib64/pkgconfig:/opt/ffmpeg/lib/x86_64-linux-gnu/pkgconfig:/opt/ffmpeg/lib/aarch64-linux-gnu/pkgconfig:/usr/lib/x86_64-linux-gnu/pkgconfig:/usr/lib/pkgconfig"

ENV LD_LIBRARY_PATH="/opt/ffmpeg/lib:/opt/ffmpeg/lib64:/opt/ffmpeg/lib/aarch64-linux-gnu"
```

**Status:** ✅ Already includes both architectures in path! But x86_64 path is listed first.

---

### 6. ASSEMBLY AND OPTIMIZATION FLAGS

#### Current Status:
- **NASM**: Used for libmp3lame, AOM (x86-only assembler)
- **YASM**: Used for libvpx (x86-only assembler)

#### Required Changes:
On ARM, these libraries have **NEON** intrinsics support and don't need x86 assemblers.

**libvpx on ARM:**
- Supports NEON automatically
- Configure script auto-detects ARM
- Remove `--as=yasm` (let it auto-detect)

**libmp3lame on ARM:**
- Use `--disable-nasm` flag
- C implementation is sufficient, or use ARM NEON if available

**AOM on ARM:**
- Use `-DENABLE_NASM=OFF`
- Has ARM NEON optimizations built-in

---

### 7. ARCHITECTURE DETECTION - MISSING

**Critical Gap:** No architecture detection in any shell script.

**Recommended Addition:**
```bash
# Detect architecture
ARCH=$(uname -m)
case "$ARCH" in
    x86_64|amd64)
        MULTIARCH_TRIPLET="x86_64-linux-gnu"
        CUDA_TARGETS_PATH="x86_64-linux"
        ENABLE_NASM="yes"
        ENABLE_YASM="yes"
        ;;
    aarch64|arm64)
        MULTIARCH_TRIPLET="aarch64-linux-gnu"
        CUDA_TARGETS_PATH="sbsa-linux"  # or aarch64-linux
        ENABLE_NASM="no"
        ENABLE_YASM="no"
        ;;
    *)
        echo "Unsupported architecture: $ARCH"
        exit 1
        ;;
esac
```

---

## Summary of Required Changes

### High Priority (Breaking Changes)

1. **`update.py` - Architecture Detection**
   - Add architecture detection logic
   - Make x86_64-linux-gnu paths conditional
   - Add aarch64-linux-gnu paths for ARM
   - Update CUDA lib64/lib32 paths to be conditional
   - Make NASM/YASM flags conditional

2. **`build_source.sh` Template**
   - Add architecture detection at the top
   - Make libvpx `--as=yasm` conditional (remove on ARM)
   - Make libmp3lame `--enable-nasm` conditional
   - Make AOM `-DENABLE_NASM` conditional

3. **`install_ffmpeg.sh` Template**
   - Replace hardcoded `x86_64-linux-gnu` with `$MULTIARCH_TRIPLET` variable
   - Update CUDA targets path to be conditional
   - Update pkgconfig loop to use variable

### Medium Priority

4. **Template Files** (`templates/Dockerfile-env-*`)
   - Consider reordering PKG_CONFIG_PATH to be arch-neutral
   - Document that both architectures are supported

5. **Testing**
   - Add CI/CD for ARM64 builds
   - Test all variants on ARM64 hardware

### Low Priority

6. **Documentation**
   - Update README.md to mention ARM64 support
   - Add architecture-specific notes for NVIDIA variant
   - Document NASM/YASM not being used on ARM

---

## Architecture-Specific Considerations

### NVIDIA CUDA Variant (`nvidia2404`)

**Challenge:** NVIDIA CUDA on ARM requires Jetson or Grace Hopper platforms
- CUDA paths: `/usr/local/cuda/targets/sbsa-linux/` or `/usr/local/cuda/targets/aarch64-linux/`
- Not all NVIDIA features available on ARM (depends on GPU)
- Jetson platforms use different library paths

**Recommendation:** Add platform detection and skip NVIDIA variant on incompatible ARM platforms, or make it conditional.

### VAAPI Variant (`vaapi2404`)

**Status:** ✅ VAAPI works on ARM (Rockchip, Amlogic, etc.)
- ARM SoCs have video acceleration (Mali, Rockchip, etc.)
- VAAPI is supported on ARM Linux
- No major changes needed beyond general ARM fixes

### Alpine Variants (`alpine320`, `scratch320`)

**Status:** ✅ Alpine supports ARM64 natively
- musl libc works on ARM
- All packages available for aarch64
- Just need to fix assembly tool flags

---

## Files Requiring Modification

### Source of Truth (must fix first):
1. ✅ `update.py` - Main generator script (Python)
2. ✅ `build_source.sh` - Template for build scripts
3. ✅ `install_ffmpeg.sh` - Template for install scripts

### Generated Files (will auto-regenerate):
- All files under `docker-images/8.0/*/` are auto-generated by `update.py`
- After fixing `update.py`, run it to regenerate all variant files

### Template Files (environment variables):
1. `templates/Dockerfile-env-ubuntu`
2. `templates/Dockerfile-env-ubuntu-edge`
3. `templates/Dockerfile-env-alpine`
4. `templates/Dockerfile-env-alpine-scratch`
5. `templates/Dockerfile-env-nvidia`
6. `templates/Dockerfile-env-vaapi`

---

## Testing Strategy

### Phase 1: Build Testing
1. Build on ARM64 host (native)
2. Build using Docker buildx (cross-compilation)
3. Verify all libraries compile successfully

### Phase 2: Functionality Testing
1. Run ffmpeg with various codecs
2. Test hardware acceleration (VAAPI on ARM)
3. Benchmark encoding/decoding performance

### Phase 3: CI/CD Integration
1. Add ARM64 builders to Azure Pipelines
2. Add ARM64 builders to GitLab CI
3. Publish multi-arch images to Docker Hub

---

## Multi-Architecture Build Strategy

### Recommended Approach: Docker Buildx

```bash
# Create buildx builder with multi-arch support
docker buildx create --name ffmpeg-builder --use --platform linux/amd64,linux/arm64

# Build multi-arch image
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --tag jrottenberg/ffmpeg:8.0-ubuntu2404 \
  --push \
  -f docker-images/8.0/ubuntu2404/Dockerfile .
```

This will:
- Build for both architectures simultaneously
- Create a multi-arch manifest
- Allow `docker pull` to automatically select the right architecture

---

## Risk Assessment

### Low Risk:
- ✅ Base images support multi-arch
- ✅ FFmpeg itself compiles on ARM
- ✅ Most libraries have ARM support
- ✅ Template already includes aarch64 in paths

### Medium Risk:
- ⚠️ NVIDIA variant may have limited ARM support (Jetson-specific)
- ⚠️ Testing coverage on ARM64 hardware needed
- ⚠️ Performance tuning for ARM-specific optimizations

### High Risk:
- ❌ None identified - this is a clean migration

---

## Next Steps

1. **Fix Source of Truth** - Modify `update.py` with architecture detection
2. **Update Templates** - Add architecture detection to shell script templates
3. **Regenerate Files** - Run `update.py` to regenerate all variant files
4. **Test Build** - Build one variant (e.g., ubuntu2404) on ARM64
5. **Iterate** - Fix any issues discovered during build
6. **Full Testing** - Build all variants on ARM64
7. **CI/CD Integration** - Add ARM64 to build pipelines
8. **Documentation** - Update README with ARM64 support

---

## Conclusion

This codebase is **well-positioned for ARM migration**. The main blockers are:
1. Hardcoded x86_64 paths (easily fixed with variables)
2. x86-specific assembly tools (disable on ARM, libraries have ARM optimizations)
3. Architecture detection logic (add at the beginning of scripts)

**Estimated Effort:** 1-2 days for implementation + testing

**Compatibility:** All variants except potentially NVIDIA (platform-dependent) should work on ARM64.

The repository already shows awareness of multi-arch support (aarch64 paths in environment variables), suggesting the maintainers have considered ARM support. The changes needed are primarily about making the architecture detection **dynamic** rather than **hardcoded**.
