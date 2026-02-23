# FFmpeg Docker Images - Build System Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        BUILD SYSTEM OVERVIEW                              │
│                      (Source of Truth → Generated)                        │
└─────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│                         SOURCE FILES (Edit These)                          │
└──────────────────────────────────────────────────────────────────────────┘

   ┌─────────────────────────────────────────────────────────────────┐
   │  update.py (Python Script - MASTER GENERATOR)                    │
   │  ─────────────────────────────────────────────────────────────  │
   │  • Reads FFmpeg versions                                         │
   │  • Generates Dockerfiles for all variants                        │
   │  • Generates build configuration flags                           │
   │  • Creates CI/CD configs (Azure, GitLab)                         │
   │                                                                   │
   │  ⚠️  ISSUES:                                                      │
   │    Line 307-310: Hardcoded /opt/ffmpeg/lib/x86_64-linux-gnu     │
   │    Line 325-328: Hardcoded /usr/include/x86_64-linux-gnu        │
   │    Line 285-286: Hardcoded CUDA lib64/lib32 paths               │
   └─────────────────────────────────────────────────────────────────┘
                              │
                              │ Generates ▼
                              │
   ┌──────────────────────────┴──────────────────────────────────────┐
   │                                                                   │
   ▼                                                                   ▼
┌──────────────────────────────┐      ┌─────────────────────────────────┐
│  templates/                   │      │  docker-images/                  │
│  ──────────────────────────  │      │  ───────────────────────────────│
│  • Dockerfile-env-*          │      │  8.0/                            │
│  • Dockerfile-run-*          │      │    ├── ubuntu2404/               │
│  • Dockerfile-template.*     │      │    │   ├── Dockerfile            │
│                               │      │    │   ├── build_source.sh      │
│  ✅ Already ARM-aware        │      │    │   ├── install_ffmpeg.sh    │
│     (includes aarch64 paths) │      │    │   └── download_tarballs.sh │
└──────────────────────────────┘      │    │                             │
                                       │    ├── alpine320/               │
                                       │    ├── scratch320/              │
┌──────────────────────────────┐      │    ├── nvidia2404/              │
│  build_source.sh (Template)   │      │    ├── vaapi2404/               │
│  ────────────────────────────│      │    └── ubuntu2404-edge/         │
│  • Builds FFmpeg libraries    │      │                                 │
│  • Called during Docker build │      │  ⚠️  AUTO-GENERATED - DO NOT    │
│                               │      │      EDIT DIRECTLY!             │
│  ⚠️  ISSUES:                  │      └─────────────────────────────────┘
│    Line 77:  --as=yasm       │
│    Line 89:  --enable-nasm   │
│    Line 173: ENABLE_NASM=on  │
└──────────────────────────────┘

┌──────────────────────────────┐
│  install_ffmpeg.sh (Template) │
│  ────────────────────────────│
│  • Installs built FFmpeg      │
│  • Copies libraries to system │
│                               │
│  ⚠️  ISSUES:                  │
│    Line 43-45: grep x86_64   │
│    Line 47-49: cuda x86_64   │
│    Line 81: x86_64 pkgconfig │
└──────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                        DOCKERFILE BUILD FLOW                              │
└─────────────────────────────────────────────────────────────────────────┘

  FROM ubuntu:24.04 / alpine:3.20 / nvidia/cuda:12.6.2-devel-ubuntu24.04
       │
       ├─ Stage 1: Builder
       │  │
       │  ├─ Install build dependencies (cmake, gcc, etc.)
       │  │
       │  ├─ Run: generate-source-of-truth-ffmpeg-versions.py
       │  │     └─ Creates manifest of library versions
       │  │
       │  ├─ Run: download_tarballs.sh
       │  │     └─ Downloads all library source tarballs
       │  │
       │  ├─ Run: build_source.sh ⚠️  (x86-specific flags)
       │  │     └─ Builds: libx264, libx265, libvpx, aom, opus, etc.
       │  │     └─ Builds: FFmpeg with all codec support
       │  │
       │  └─ Run: install_ffmpeg.sh ⚠️  (x86-specific paths)
       │        └─ Copies libraries to /usr/local/
       │
       ├─ Stage 2: Runtime (minimal final image)
       │  │
       │  └─ COPY --from=builder /usr/local/bin/ffmpeg
       │     COPY --from=builder /usr/local/lib/
       │
       └─ ENTRYPOINT ["ffmpeg"]

┌─────────────────────────────────────────────────────────────────────────┐
│                    ARCHITECTURE-SPECIFIC PATHS                            │
└─────────────────────────────────────────────────────────────────────────┘

                    x86_64 (Current)          ARM64 (Target)
                    ────────────────          ──────────────
Multiarch Triplet:  x86_64-linux-gnu    →     aarch64-linux-gnu
Library Path:       /usr/lib/x86_64... →     /usr/lib/aarch64...
Include Path:       /usr/include/x86... →     /usr/include/aarch64...
CUDA Target:        x86_64-linux       →     sbsa-linux (ARM servers)
                                       →     aarch64-linux (Jetson)

SIMD Instructions:  SSE, AVX, AVX2     →     NEON
Assembler:          NASM, YASM         →     Auto-detect (or disable)

┌─────────────────────────────────────────────────────────────────────────┐
│                      LIBRARY BUILD DEPENDENCIES                           │
└─────────────────────────────────────────────────────────────────────────┘

Build Order (dependencies resolved):

1. libogg             →  No dependencies
2. libvorbis          →  Requires: libogg
3. libtheora          →  Requires: libogg
4. libopus            →  No dependencies
5. libvpx ⚠️          →  Uses YASM (x86-specific)
6. libwebp            →  No dependencies
7. libmp3lame ⚠️      →  Uses NASM (x86-specific)
8. libx264            →  No dependencies
9. libx265            →  No dependencies
10. libxvid           →  No dependencies
11. libfdk-aac        →  No dependencies
12. openjpeg          →  No dependencies
13. freetype          →  No dependencies
14. libvidstab        →  No dependencies
15. fribidi           →  No dependencies
16. fontconfig        →  Requires: freetype
17. libass            →  Requires: freetype, fontconfig, fribidi
18. kvazaar           →  No dependencies
19. aom ⚠️            →  Uses NASM (x86-specific)
20. libsvtav1         →  No dependencies
21. libdav1d          →  No dependencies
22. libvmaf           →  No dependencies
23. whisper           →  No dependencies
24. FFmpeg            →  Requires: ALL of the above

⚠️  = Requires architecture-specific changes

┌─────────────────────────────────────────────────────────────────────────┐
│                         MULTI-ARCH BUILD WORKFLOW                         │
└─────────────────────────────────────────────────────────────────────────┘

Option 1: Native Build (on ARM64 hardware)
  ┌──────────────┐
  │ ARM64 Server │  ← Build runs natively
  │  (Graviton)  │
  └──────────────┘
       │
       ├─ docker build -t ffmpeg:arm64 .
       └─ Fast, native compilation

Option 2: Cross-Compile (using Docker Buildx)
  ┌──────────────┐
  │ x86_64 Server│
  └──────────────┘
       │
       ├─ docker buildx create --platform linux/arm64
       ├─ Uses QEMU emulation
       └─ Slower, but works anywhere

Option 3: Multi-Arch Manifest (Recommended for Production)
  ┌──────────────┐     ┌──────────────┐
  │ x86_64 Build │     │ ARM64 Build  │
  └──────────────┘     └──────────────┘
       │                     │
       └─────────┬───────────┘
                 │
         ┌───────▼────────┐
         │ Multi-Arch Tag │
         │  ffmpeg:8.0    │
         └────────────────┘
                 │
     ┌───────────┴───────────┐
     │                       │
     ▼                       ▼
  x86_64 Image          ARM64 Image

  docker pull automatically selects correct architecture!

┌─────────────────────────────────────────────────────────────────────────┐
│                           VARIANT COMPARISON                              │
└─────────────────────────────────────────────────────────────────────────┘

Variant          Base OS    Libraries   FFmpeg  HW Accel  Size   ARM Status
────────────────────────────────────────────────────────────────────────────
ubuntu2404       Ubuntu     From Repos  Source  None      Small  ✅ Ready
ubuntu2404-edge  Ubuntu     Source      Source  None      Large  ✅ Ready
alpine320        Alpine     From Repos  Source  None      Tiny   ✅ Ready
scratch320       Alpine     Source      Source  None      Minimal✅ Ready
vaapi2404        Ubuntu     From Repos  Source  VAAPI     Small  ✅ Ready
nvidia2404       Ubuntu     Source      Source  NVENC     Large  ⚠️  Platform-dependent

✅ Ready   = Works on all ARM64 hardware
⚠️  Partial = Requires specific hardware (Jetson, Grace Hopper)

┌─────────────────────────────────────────────────────────────────────────┐
│                         FILE MODIFICATION IMPACT                          │
└─────────────────────────────────────────────────────────────────────────┘

              ┌──────────────┐
              │  update.py   │  ← FIX THIS
              └──────┬───────┘
                     │
      ┌──────────────┼──────────────┐
      │              │              │
      ▼              ▼              ▼
┌──────────┐  ┌─────────────┐  ┌──────────────┐
│ Variants │  │ build_*.sh  │  │ CI/CD configs│
│ (6 × 3)  │  │ install_*.sh│  │ (Azure/GitLab│
└──────────┘  └─────────────┘  └──────────────┘
  18 files       2 templates      2 files
  AUTO-REGEN     FIX TEMPLATES    AUTO-REGEN

Total Files Modified: 3 source files (update.py + 2 templates)
Total Files Regenerated: 20+ files (automatic)

┌─────────────────────────────────────────────────────────────────────────┐
│                          TESTING MATRIX                                   │
└─────────────────────────────────────────────────────────────────────────┘

Variant            Build Test    Encode Test   Decode Test   HW Accel Test
────────────────────────────────────────────────────────────────────────────
ubuntu2404         Required      Required      Required      N/A
ubuntu2404-edge    Required      Required      Required      N/A
alpine320          Required      Required      Required      N/A
scratch320         Required      Required      Required      N/A
vaapi2404          Required      Required      Required      ARM GPU needed
nvidia2404         Optional*     Optional*     Optional*     Jetson needed

*Optional because NVIDIA variant requires specific ARM hardware

Test Commands:
  Build:   docker build -t test-variant .
  Encode:  ffmpeg -i input.mp4 -c:v libx264 output.mp4
  Decode:  ffmpeg -i input.mp4 -f null -
  HW:      ffmpeg -hwaccel vaapi -i input.mp4 output.mp4

┌─────────────────────────────────────────────────────────────────────────┐
│                     PERFORMANCE OPTIMIZATION                              │
└─────────────────────────────────────────────────────────────────────────┘

x86_64 SIMD                    ARM64 NEON Equivalent
──────────────                 ─────────────────────
SSE2 (128-bit)           →     NEON (128-bit)
AVX (256-bit)            →     SVE (scalable, 128-2048 bit)
AVX2                     →     SVE2

FFmpeg ARM Optimizations:
  ✅ x264: Full NEON support
  ✅ x265: NEON optimizations
  ✅ libvpx: ARM assembly code
  ✅ AOM: NEON intrinsics
  ✅ Opus: ARM optimizations
  ✅ FFmpeg: Extensive NEON support

Expected Performance vs x86_64:
  Graviton3:     95-110% (depending on codec)
  Ampere Altra:  90-105%
  Apple M-series: 110-130% (with optimizations)

┌─────────────────────────────────────────────────────────────────────────┐
│                           SUCCESS CRITERIA                                │
└─────────────────────────────────────────────────────────────────────────┘

Phase 1: Build Success
  ☐ All variants build without errors on ARM64
  ☐ All libraries compile successfully
  ☐ FFmpeg binary runs and shows version

Phase 2: Functional Success
  ☐ Can encode with x264, x265, VP9, AV1
  ☐ Can decode all major formats
  ☐ Hardware acceleration works (VAAPI on supported HW)
  ☐ Multi-arch Docker manifest created

Phase 3: Performance Success
  ☐ Encoding speed within 90-110% of x86_64
  ☐ No crashes during long encodes
  ☐ All codecs functionally identical to x86_64

Phase 4: Production Success
  ☐ CI/CD builds both architectures automatically
  ☐ Multi-arch images published to Docker Hub
  ☐ Documentation updated
  ☐ No reported issues for 1 week

┌─────────────────────────────────────────────────────────────────────────┐
│                              TIMELINE                                     │
└─────────────────────────────────────────────────────────────────────────┘

Day 1 (8 hours):
  Hour 1-2:   Review documentation, understand build system
  Hour 3-4:   Modify update.py with architecture detection
  Hour 5-6:   Modify build_source.sh and install_ffmpeg.sh templates
  Hour 7-8:   Regenerate files, first build attempt (ubuntu2404)

Day 2 (8 hours):
  Hour 1-2:   Debug build issues
  Hour 3-4:   Build all variants
  Hour 5-6:   Functional testing
  Hour 7-8:   Documentation updates

Day 3 (4 hours):
  Hour 1-2:   CI/CD integration
  Hour 3-4:   Final testing and verification

Total: 20 hours (2.5 days)
```
