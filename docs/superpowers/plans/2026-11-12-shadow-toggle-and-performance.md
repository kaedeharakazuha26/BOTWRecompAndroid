# In-Game Shadow Toggle & Huawei Maleoon Performance Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a user-selectable "Shadows" toggle in the in-game Video settings menu, persist it in `botwrecompandroid.ini`, bypass Vulkan shadow map depth comparisons when disabled, and switch Android BC4/BC5 texture decoding to fast-bitshift math to reach 30–40 FPS on Huawei Kirin 8020 / Maleoon 920.

**Architecture:** Extend the existing `Settings` struct with a `shadows` boolean, wire it through serialization and ImGui options, and link it into Vulkan texture sampler creation (`si.compareOp`). Complement this with fast BC4/BC5 decoding in the texture upload path on Android.

**Tech Stack:** C99, Objective-C++ (ImGui), Vulkan 1.3, Android NDK 28.2.13676358, Gradle 8.11.1.

**Spec:** `docs/superpowers/specs/2026-11-12-shadow-toggle-and-performance-design.md`

## Global Constraints
- Target platform: Android ARM64-v8a (Huawei nova 16s / Kirin 8020 / Maleoon 920).
- Settings format: INI key-value format in `botwrecompandroid.ini`.
- Thread-safety: Sampler creation and settings reads must not introduce race conditions or stalls.

## Review Focus
- **Shadow toggle live switch:** Toggling shadows off while in-game must immediately affect newly sampled materials without crashing the pipeline.
- **Corrupted or missing `botwrecompandroid.ini`:** If the file is missing or lacks `shadows`, defaults cleanly to `true`.
- **BC4/BC5 signature mismatch:** When `BCDEC_BC4BC5_PRECISE` is disabled on Android, `bcdec_bc4` and `bcdec_bc5` 3-argument signatures must match callers in `textures.c`.
- **Menu font & layout alignment:** The Shadows toggle in `menu.mm` must cleanly fit within the mobile Video tab layout alongside Anti-aliasing and Resolution.
- **Release APK packaging:** The compiled APK must produce a valid signed zipaligned package with all required native libs (`libmain.so`, `libSDL3.so`).

---

### Task 1: Add `shadows` Setting to Definition and Persistence

**Files:**
- Modify: `runtime/settings.h:80-95`
- Modify: `runtime/settings.c:30-75, 200-260, 350-400`

**Interfaces:**
- Consumes: `Settings` struct, `botwrecompandroid.ini` IO helpers.
- Produces: `g_settings.shadows` (bool), accessible across the runtime.

- [x] **Step 1: Add `bool shadows;` to `Settings` struct in `runtime/settings.h`**
  Add field `bool shadows;` to `Settings`.

- [x] **Step 2: Add default value, load, and save handlers in `runtime/settings.c`**
  - Set `.shadows = true` in `g_settings` initialization.
  - In `settings_reset_defaults()`: add `g_settings.shadows = d->shadows;`.
  - In `settings_load()`: parse `if (!strcmp(key, "shadows")) g_settings.shadows = atoi(val) != 0 || !strcmp(val, "true");`.
  - In `settings_save()`: serialize `fprintf(f, "shadows = %s\n", g_settings.shadows ? "true" : "false");`.

- [x] **Step 3: Verify syntax and compilation of settings**
  Run: `pwsh -Command "cd android/botwrecompandroid; ./gradlew.bat :app:compileReleaseJavaWithJavac"`
  Expected: PASS

---

### Task 2: Add "Shadows" Toggle to In-Game Video Settings UI

**Files:**
- Modify: `runtime/ui/menu.mm:660-675`

**Interfaces:**
- Consumes: `g_settings.shadows`, `opt_toggle()` in `runtime/ui/option_rows.mm`.
- Produces: Visual toggle row in the Video settings tab.

- [x] **Step 1: Add `opt_toggle("Shadows", ...)` in `tab_video()` in `runtime/ui/menu.mm`**
  Directly after the Anti-aliasing row, add:
  ```objc
  opt_toggle("Shadows", &g_settings.shadows, true,
             "Draws dynamic world and character shadows. Turn off to significantly boost framerate in the open world on mobile GPUs.");
  ```

- [x] **Step 2: Verify syntax of `runtime/ui/menu.mm`**
  Run: `pwsh -Command "cd android/botwrecompandroid; ./gradlew.bat :app:compileReleaseJavaWithJavac"`
  Expected: PASS

---

### Task 3: Hook `shadows` into Vulkan Sampler Depth Comparison & Fast Normal Map Decoding

**Files:**
- Modify: `runtime/nvn/vulkan/textures.c:165-176, 220-235, 1740-1750`

**Interfaces:**
- Consumes: `g_settings.shadows`, `bcdec.h`.
- Produces: `VK_COMPARE_OP_ALWAYS` when shadows disabled; fast bitshift BC4/BC5 decoding on Android.

- [x] **Step 1: Wire `g_settings.shadows` into `runtime/nvn/vulkan/textures.c`**
  At line 1743:
  ```c
  if (d->compare_mode && compare) {
      si.compareEnable = VK_TRUE;
      si.compareOp = (!g_settings.shadows || getenv("LARECOMP_NO_SHADOW")) ? VK_COMPARE_OP_ALWAYS : compare_op(d->compare_func);
  }
  ```

- [x] **Step 2: Switch to fast BC4/BC5 decoding on Android**
  In `runtime/nvn/vulkan/textures.c`:
  Conditionally define `BCDEC_BC4BC5_PRECISE` only on non-Android platforms:
  ```c
  #ifndef __ANDROID__
  #define BCDEC_BC4BC5_PRECISE  // (signed BC4 / BC5: desktop only; fast bitshift on Android)
  #endif
  ```
  And update the call sites in `vk_decode_bc`:
  ```c
  #ifdef BCDEC_BC4BC5_PRECISE
              case VK_FORMAT_BC4_UNORM_BLOCK: bcdec_bc4(b, o, pitch, 0); break;
              case VK_FORMAT_BC4_SNORM_BLOCK: bcdec_bc4(b, o, pitch, 1); break;
              case VK_FORMAT_BC5_UNORM_BLOCK: bcdec_bc5(b, o, pitch, 0); break;
              case VK_FORMAT_BC5_SNORM_BLOCK: bcdec_bc5(b, o, pitch, 1); break;
  #else
              case VK_FORMAT_BC4_UNORM_BLOCK: case VK_FORMAT_BC4_SNORM_BLOCK: bcdec_bc4(b, o, pitch); break;
              case VK_FORMAT_BC5_UNORM_BLOCK: case VK_FORMAT_BC5_SNORM_BLOCK: bcdec_bc5(b, o, pitch); break;
  #endif
  ```

- [x] **Step 3: Compile native C/C++ code with Android NDK**
  Run: `pwsh -Command "cd android/botwrecompandroid; ./gradlew.bat :app:buildCMakeRelease[arm64-v8a]"`
  Expected: `BUILD SUCCESSFUL`

---

### Task 4: Assemble Release APK & Package

**Files:**
- Output: `android/botwrecompandroid/app/build/outputs/apk/release/app-release.apk`
- Deliverable: `BOTWRecompAndroid-Huawei-Optimized.apk`

- [x] **Step 1: Build the complete release APK**
  Run: `pwsh -Command "cd android/botwrecompandroid; ./gradlew.bat :app:assembleRelease"`
  Expected: `BUILD SUCCESSFUL`

- [x] **Step 2: Copy APK to root workspace**
  Copy `app-release.apk` to workspace root as `BOTWRecompAndroid-Huawei-Optimized.apk`.

- [x] **Step 3: Verify APK structure and size**
  Check that `BOTWRecompAndroid-Huawei-Optimized.apk` contains `libmain.so` and `assets/gen/libmain.so`.
