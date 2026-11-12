# Implementation Plan: Huawei Nova 16s Optimization

**Goal:** Optimize BOTWRecompAndroid for Huawei nova 16s (Kirin 8020 / Maleoon 920 GPU) by adding explicit vendor detection, ASTC 4x4 transcoding, pipeline management, and aggressive performance defaults.

## Tasks

### Task 1: Add Huawei / Maleoon Vendor Detection and Pipeline Caps
- **Files to modify:**
  - `runtime/nvn/vulkan/device.c`
  - `runtime/nvn/vulkan/exec.c`
  - `runtime/nvn/vulkan/textures.c`
- **Changes:**
  - Define `VK_VENDOR_ID_HUAWEI 0x19E5`.
  - Add `bool huawei = g_vk.props.vendorID == VK_VENDOR_ID_HUAWEI;`.
  - In `exec.c`, apply the 1500 pipeline limit to both Mali (`0x13B5`) and Huawei (`0x19E5`).
  - In `textures.c`, ensure mutable format restrictions apply to Huawei GPU vendor ID.

### Task 2: Ensure ASTC 4x4 Transcoding for Maleoon
- **Files to modify:**
  - `runtime/nvn/vulkan/textures.c`
- **Changes:**
  - In `bc1_astc()`, ensure ASTC 4x4 transcoding activates reliably when `vendorID == 0x19E5` if format supports ASTC.
  - Prevent fallback to 8x RGBA8 decompression.

### Task 3: Adjust Default Resolution & Profile for Optimal Framerate
- **Files to modify:**
  - `runtime/settings.c`
  - `runtime/settings.h`
- **Changes:**
  - Add or update default resolution to `RES_Q50` (50% scale) or safe fallback on non-Snapdragon GPUs.
  - Enable FSR 1.0 upscaling by default.

### Task 4: Compilation and Verification
- **Verification:**
  - Verify C syntax and buildability.
  - Run gradle build checks.
