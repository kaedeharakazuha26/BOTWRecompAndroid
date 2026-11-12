# BOTWRecompAndroid: In-Game Shadow Toggle & Huawei Maleoon Performance Design

## Intent & The Why
The player is getting 19–25 FPS in the open world on a Huawei nova 16s (Kirin 8020 / Maleoon 920 GPU). While playable, the goal is reaching stable 30–40 FPS. The primary GPU bottlenecks are multi-cascade shadow mapping passes and software-decoded BC4/BC5 normal maps on the phone's 1.5K+ display. The player wants an in-game option to toggle shadows on or off so they can freely choose between visual fidelity and maximum framerate.

## Goals, Non-Goals & Anti-Goals
- **Goals:**
  - Add an in-game "Shadows" toggle (`On` / `Off`) under Video settings in `runtime/ui/menu.mm`.
  - Persist the setting in `botwrecompandroid.ini` so it survives restarts.
  - When disabled, switch Vulkan shadow sampler depth testing to `VK_COMPARE_OP_ALWAYS`, bypassing expensive shadow cascade pixel testing.
  - Switch normal map decoding on Android to fast bitshift decompression, eliminating CPU division overhead during terrain and foliage streaming.
  - Build an updated release APK for immediate testing.
- **Non-Goals:**
  - Creating complex graphics preset managers that override independent resolution and anti-aliasing choices.
  - Modifying the game's core Havok physics or delta-time clock (already handled by `framepace.c`).
- **Anti-Goals:**
  - Breaking existing graphics settings or introducing visual artifacts in menus and cutscenes.
  - Forcing shadows permanently off with no way to turn them back on.

## Constraints
- **Hardware/Driver:** HiSilicon Maleoon 920 GPU (`vendorID 0x19E5`) running Vulkan on Android.
- **Settings Persistence:** Must maintain backward compatibility with existing `botwrecompandroid.ini` files.
- **UI:** Must maintain clean ImGui touch-friendly layout in the existing `Video` tab.

## Approaches Considered
1. **Approach 1: Dedicated "Shadows" Toggle + Built-in Normal Map Fast Paths (Selected)**
   - *Why chosen:* Provides the exact control requested without cluttering the mobile touch interface. Resolution, upscaler, and anti-aliasing remain independently adjustable.
2. **Approach 2: Performance Preset Profiles (Quality / Balanced / High FPS)**
   - *Trade-off:* Bundles multiple settings into rigid profiles, forcing resolution down or disabling AA even if the user wanted them enabled.
3. **Approach 3: Granular Mobile Shader Knobs**
   - *Trade-off:* Adds too many developer-level switches to the mobile menu, creating confusion and maintenance bloat.

## Implementation Details
1. **Settings Definition (`runtime/settings.h`):**
   - Add `bool shadows;` to `Settings` struct.
2. **Settings Persistence & Defaults (`runtime/settings.c`):**
   - Set default `.shadows = true`.
   - In `settings_load()`, parse `shadows = true|false`.
   - In `settings_save()`, serialize `shadows`.
   - In `settings_reset_defaults()`, reset `g_settings.shadows`.
3. **In-Game Video Menu UI (`runtime/ui/menu.mm`):**
   - In `tab_video()`, insert `opt_toggle("Shadows", &g_settings.shadows, true, "Draws dynamic world and character shadows. Turn off to significantly boost framerate in the open world.");`.
4. **Vulkan Sampler Depth Comparison (`runtime/nvn/vulkan/textures.c`):**
   - Wire `g_settings.shadows` into `si.compareOp`: when `!g_settings.shadows` (or `LARECOMP_NO_SHADOW` is set), use `VK_COMPARE_OP_ALWAYS`.
   - Replace slow `BCDEC_BC4BC5_PRECISE` decoding with the fast bitshift variant for Android builds.
5. **Build and Verification:**
   - Compile via `./gradlew.bat :app:assembleRelease`.
   - Verify APK generation and confirm settings parse cleanly.
