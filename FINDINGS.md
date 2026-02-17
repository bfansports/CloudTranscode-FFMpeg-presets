# CloudTranscode-FFMpeg-presets Audit Findings

**Date:** 2026-02-17
**Auditor:** AI DevOps Agent (opus)
**Repo:** bfansports/CloudTranscode-FFMpeg-presets
**Branch:** master
**Scope:** Encoding quality vs file size, preset security (injection via params), outdated codecs, HLS/DASH, mobile compatibility

---

## Critical

### C-1: `video_codec_options` Parser Mismatch — Options Silently Ignored

**Files affected:** All 23 preset JSON files + `CloudTranscode/src/activities/transcoders/VideoTranscoder.php`

**Issue:** Every preset uses `:` as the key-value separator in `video_codec_options`:
```json
"video_codec_options": "MaxReferenceFrames:3,Profile:baseline,Level:3"
```

But the consumer (`VideoTranscoder.php` line 432) parses with `=`:
```php
$keyVal = explode("=", $option);
```

`explode("=", "Profile:baseline")` produces `["Profile:baseline"]` — a single-element array. `$keyVal[1]` is undefined, so the `if` conditions (`$keyVal[0] === 'Profile'`) never match because `$keyVal[0]` contains `"Profile:baseline"`, not `"Profile"`.

**Impact:** `-profile:v`, `-level`, and `-refs` flags are **never set** in the FFmpeg command. FFmpeg falls back to its own defaults (`high` profile, level auto-detected, refs auto). This means:
- **Baseline profile presets are not enforced** — FFmpeg uses `high` profile by default, which older devices (iPhone 3GS, Kindle Fire, iPod Touch) cannot decode
- **Level constraints are not enforced** — output could exceed device decoder capabilities
- **MaxReferenceFrames is not enforced** — higher ref counts increase decode complexity

The output videos may play fine on modern devices but **will fail on the legacy devices these presets target**.

**Recommendation:** Fix the parser in `VideoTranscoder.php` to use `explode(":", $option)` instead of `explode("=", $option)`. Alternatively, update all preset files to use `=` format. The parser fix is preferred — it's one line vs 23 files, and the colon format is more readable.

### C-2: `custom_cmd` Allows Arbitrary Command Injection

**File affected:** `VideoTranscoder.php` `craft_ffmpeg_custom_cmd()` (lines 194-224)

**Issue:** The `custom_cmd` field in job input JSON allows arbitrary FFmpeg commands with template variable replacement. While `${input_file}` is escaped with `escapeshellarg()`, `${output_file}` is **not escaped** and is injected via `preg_replace` directly. Additionally, the `custom_cmd` string itself is never sanitized — it can contain shell metacharacters, pipes, and subcommands.

**Impact:** If an attacker can submit a CloudTranscode job (via SFN API), they can execute arbitrary commands on the worker container. This is partially mitigated by:
- Workers run in Docker containers (blast radius limited to container)
- SFN activity tasks require IAM authentication
- The presets repo itself does not contain `custom_cmd` — it comes from job input

**Recommendation:** This finding is cross-repo (lives in CloudTranscode, not this presets repo) but is relevant context. File as issue in `bfansports/CloudTranscode`. The fix: validate `custom_cmd` against an allowlist of FFmpeg flags, or at minimum escape all template variables.

---

## High

### H-1: `kindleFireHDX-generic.json` Missing `video_codec_options`

**File:** `kindleFireHDX-generic.json`

**Issue:** This is the only preset missing the `video_codec_options` field entirely:
```json
{
    "name": "System preset: Generic KindleFireHDX",
    "description": "Generic transcoding preset for Kindle Fire HDX 8.9",
    "size": "1920x1080",
    "frame_rate": 30,
    "video_bitrate": "5000k",
    "audio_bitrate": "160k",
    "video_codec": "libx264",
    "audio_codec": "libfdk_aac",
}
```

Note the trailing comma after the last field — this is **invalid JSON**. `json_decode()` in PHP will return `null`, triggering the `BAD_PRESET_FORMAT` exception. This preset is completely broken and cannot be used.

**Impact:** Any CloudTranscode job referencing the `kindleFireHDX-generic` preset will fail at runtime with `BAD_PRESET_FORMAT`.

**Recommendation:** Remove trailing comma. Add `video_codec_options` field (e.g., `"MaxReferenceFrames:3,Profile:high,Level:4.1"` to match Kindle Fire HDX capabilities).

### H-2: All Presets Use Only H.264/AAC — No Modern Codecs

**Issue:** Every preset uses `libx264` (H.264) for video and `libfdk_aac` for audio. No presets exist for:
- **H.265/HEVC** (`libx265`) — 30-50% better compression at same quality; supported on all devices since ~2015
- **VP9** (`libvpx-vp9`) — royalty-free, excellent for web delivery
- **AV1** (`libaom-av1` or `libsvtav1`) — best compression, growing device support
- **Opus** audio — superior to AAC at all bitrates

**Impact:** bFAN is paying ~30-50% more for S3 storage and CloudFront bandwidth than necessary. For a sports platform serving video to millions of fans, this is significant.

**Recommendation:** Add modern codec presets as new files (don't modify existing). Priority:
1. H.265 equivalents of 720p and 1080p presets (immediate bandwidth savings)
2. VP9 for web delivery (royalty-free)
3. AV1 for future-proofing (encode is slow but decode is widespread)

### H-3: HLS Presets Have Non-Standard Resolutions

**Issue:** The HLS presets use unusual resolutions that don't match Apple's HLS Authoring Specification or common ABR ladder standards:

| Preset | Resolution | Expected (Apple HLS Spec) |
|--------|-----------|---------------------------|
| hls400k | 400x288 | 416x234 or 480x270 |
| hls600k | 480x320 | 640x360 |
| hls1M | 640x432 | 640x360 or 768x432 |
| hls1.5M | 960x640 | 960x540 |
| hls2M | 1024x768 (4:3!) | 1280x720 |

**Impact:**
- `hls2M` outputs 4:3 aspect ratio (1024x768) — sports content is almost always 16:9, causing letterboxing or distortion
- Non-standard resolutions may cause scaling artifacts on displays that expect standard dimensions
- Apple devices may not select the optimal rendition from a non-standard ladder

**Recommendation:** Create a new set of HLS presets following Apple's recommended ABR ladder (https://developer.apple.com/documentation/http-live-streaming/hls-authoring-specification-for-apple-devices). Standard 16:9 ladder: 416x234, 640x360, 768x432, 960x540, 1280x720, 1920x1080.

### H-4: No DASH/CMAF Presets

**Issue:** Only HLS presets exist. No DASH (Dynamic Adaptive Streaming over HTTP) or CMAF (Common Media Application Format) presets are provided.

**Impact:** DASH is required for Android-native adaptive streaming. Since bFAN has both iOS and Android apps, relying only on HLS means:
- Android devices use HLS (supported but not native — ExoPlayer handles it, but DASH is more efficient)
- No fMP4 segment format (HLS+TS only), missing out on CMAF's single-encoding benefit

**Recommendation:** Add DASH preset variants, or better yet, add CMAF presets that can serve both HLS and DASH from a single set of encoded segments.

---

## Medium

### M-1: `1080p-generic.json` Uses Baseline Profile — Wastes Bandwidth

**File:** `1080p-generic.json`

**Issue:** The 1080p preset uses `Profile:baseline` with `Level:4`. Baseline profile at 1080p lacks B-frames and CABAC entropy coding, resulting in ~15-25% larger file sizes compared to `main` or `high` profile at the same visual quality.

**Impact:** 1080p is likely the highest-traffic preset. Using baseline profile at this resolution wastes bandwidth and storage with no compatibility benefit — any device capable of decoding 1080p supports at least `main` profile.

**Recommendation:** Change to `Profile:high,Level:4.1` (matches what `iphone4S-generic.json` already uses for 1080p). Baseline profile should only be used for the lowest-resolution presets targeting very old hardware.

### M-2: `720p-generic.json` Uses Baseline Profile

**File:** `720p-generic.json`

**Issue:** Same as M-1 but for 720p. Uses `Profile:baseline,Level:3.1`. All 720p-capable devices support `main` profile.

**Recommendation:** Change to `Profile:main,Level:3.1` (matches `web-generic.json` which is also 720p).

### M-3: Inconsistent Frame Rate Handling

**Issue:** Frame rates are inconsistent across presets:
- Most presets: `29.97` (NTSC standard)
- `320x240-generic.json`: `21` (unusual, non-standard)
- Device presets: `30` (integer, not NTSC)
- `hls1M-generic.json`: `"auto"` (string, preserves source)
- `web-generic.json`: `30` (integer)

**Impact:**
- `frame_rate: 21` on 320x240 is non-standard and may cause judder or sync issues
- Mixing `29.97` and `30` across presets in the same ABR ladder causes frame rate switches during adaptive streaming, which is visible to users
- Only one preset uses `"auto"` — the rest force a frame rate, which can cause judder if the source is 24fps or 25fps (PAL)

**Recommendation:** Standardize: use `"auto"` for non-HLS presets (preserve source frame rate). For HLS presets, use a consistent frame rate across the entire ladder (either all `29.97` or all `30`).

### M-4: `480p-generic.json` Uses FFmpeg Auto-Scale Syntax `-2x480`

**File:** `480p-generic.json`

**Issue:** The `size` field is `"-2x480"` which uses FFmpeg's automatic width calculation (`-2` means "calculate width to maintain aspect ratio, rounded to nearest even number"). This is valid FFmpeg syntax but the consumer code's enlargement check (`set_output_video_size`) does `explode("x", $size)` and then `intval($width)` — `intval("-2")` returns `-2`, which will always be less than the input width, bypassing the upscale check incorrectly.

**Impact:** The aspect-ratio-preserving behavior is good, but the enlargement protection is bypassed. A 240p source could be upscaled to 480p height without the expected safety check.

**Recommendation:** If auto-scale is desired, handle the `-2` case in the consumer's enlargement check. Consider using this syntax more broadly for better aspect ratio handling.

### M-5: `libfdk_aac` Dependency — Non-Default FFmpeg Codec

**Issue:** All 23 presets use `libfdk_aac` which requires FFmpeg to be compiled with `--enable-libfdk-aac --enable-nonfree`. This is **not included in default FFmpeg builds** due to licensing (Fraunhofer FDK AAC is not GPL-compatible).

**Impact:** If the Docker base image is rebuilt without this flag (e.g., using a standard FFmpeg package), all presets fail. The built-in `aac` encoder in modern FFmpeg (5.0+) is comparable in quality to `libfdk_aac` and requires no special build flags.

**Recommendation:** For new presets, use `aac` (built-in) instead of `libfdk_aac`. For existing presets, keep `libfdk_aac` until the FFmpeg version is upgraded (at which point `aac` is the better choice).

### M-6: No JSON Schema Validation

**Issue:** There is no schema file, CI check, or validation script to ensure preset JSON files are well-formed and contain all required fields. The `kindleFireHDX-generic.json` trailing comma bug (H-1) would be caught by any JSON linter.

**Recommendation:** Add a GitHub Actions workflow that runs `jq . *.json` on every PR to catch syntax errors. Optionally add a JSON Schema definition for the preset format.

### M-7: GitHub Actions Backup Workflow Targets Wrong Branch

**File:** `.github/workflows/github-backup.yml`

**Issue:** The S3 backup workflow triggers on pushes to `develop` branch, but this repo's default branch is `master`. The backup never runs.

**Impact:** No automated backups of this repo to S3.

**Recommendation:** Change trigger branch from `develop` to `master`. Also update `actions/checkout@v2` to `v4` and `peter-evans/s3-backup@v1` to latest.

---

## Low

### L-1: Legacy Device Presets (2010-2013 Era)

**Issue:** 8 of 23 presets (35%) target devices that are long discontinued:
- `iphone3GS-generic.json` — iPhone 3GS (2009, iOS support ended 2012)
- `iphone4-generic.json` — iPhone 4 (2010, iOS support ended 2013)
- `iphone4S-generic.json` — iPhone 4S (2011, iOS support ended 2016)
- `ipodTouch-generic.json` — targets "iPhone 1, 3, iPod classic" (2007-2009)
- `appleTV2G-generic.json` — Apple TV 2nd gen (2010)
- `appleTV3G-generic.json` — Apple TV 3rd gen (2012)
- `kindleFire-generic.json` — Kindle Fire 1st gen (2011)
- `kindleFireHDX-generic.json` — Kindle Fire HDX (2013)

**Impact:** These presets are likely unused but add confusion. They use overly conservative settings (baseline profile, low bitrates) that produce worse quality than necessary for any modern device.

**Recommendation:** Don't delete — they may be referenced in existing CloudTranscode job configurations. Add a `deprecated/` directory and move them there, or add a `"deprecated": true` field to the JSON. Add modern device presets: generic iOS (H.265-capable), generic Android (H.264 high profile + H.265), smart TV.

### L-2: No Preset for Audio-Only Transcoding

**Issue:** All presets include both video and audio parameters. There are no audio-only presets (e.g., for podcast content, audio highlights, or accessibility audio descriptions).

**Recommendation:** Add audio-only presets if bFAN produces audio content (e.g., `audio-128k-aac.json`, `audio-192k-aac.json`).

### L-3: Bitrate Values Are Not Optimized for Modern CRF/CQ Encoding

**Issue:** All presets use CBR (constant bitrate) encoding via `-b:v`. Modern FFmpeg best practice is CRF (Constant Rate Factor) encoding, which produces better quality per byte by varying bitrate with content complexity. Sports content (fast motion, crowd scenes) especially benefits from variable bitrate.

**Recommendation:** For new presets, consider CRF mode. This requires adding a `video_quality` field (e.g., `"crf": 23`) and updating the consumer to use `-crf` instead of `-b:v`. Keep CBR presets for HLS (where predictable bitrate is important for ABR ladder).

### L-4: `320x240-generic.json` Has Unusually Low Settings

**File:** `320x240-generic.json`

**Issue:** Frame rate of 21 fps, 64k audio bitrate, and Level 1.3. These settings are extremely conservative, producing very low quality output. Level 1.3 is the minimum H.264 level and limits decoding to QCIF resolution.

**Recommendation:** If this preset is still needed, bump to at least 24 fps, 96k audio, and Level 2.0.

---

## Agent Skill Improvements

### S-1: Add Preset Validation to CI

Create `.github/workflows/validate-presets.yml` that runs on PR:
```yaml
name: Validate Presets
on: [pull_request]
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Validate JSON syntax
        run: |
          errors=0
          for f in *.json; do
            if ! jq empty "$f" 2>/dev/null; then
              echo "INVALID JSON: $f"
              errors=$((errors + 1))
            fi
          done
          exit $errors
      - name: Check required fields
        run: |
          required='["name","description","size","frame_rate","video_bitrate","audio_bitrate","video_codec","audio_codec"]'
          for f in *.json; do
            for field in $(echo $required | jq -r '.[]'); do
              if ! jq -e ".\"$field\"" "$f" > /dev/null 2>&1; then
                echo "MISSING $field in $f"
              fi
            done
          done
```

### S-2: Create Modern ABR Ladder Presets

Recommended new presets following Apple HLS spec + DASH best practices:
- `hls-234p-generic.json` (416x234, 145k video, 64k audio, baseline L3.0)
- `hls-360p-generic.json` (640x360, 365k video, 64k audio, main L3.0)
- `hls-432p-generic.json` (768x432, 730k video, 96k audio, main L3.1)
- `hls-540p-generic.json` (960x540, 2000k video, 128k audio, main L3.1)
- `hls-720p-generic.json` (1280x720, 3000k video, 128k audio, high L3.1)
- `hls-1080p-generic.json` (1920x1080, 6000k video, 160k audio, high L4.0)

### S-3: Add Deprecation Metadata

Extend the JSON schema with optional fields:
```json
{
  "deprecated": true,
  "deprecated_reason": "Target device discontinued. Use 480p-generic instead.",
  "successor": "480p-generic"
}
```

---

## Positive Observations

1. **Consistent JSON structure** — All presets follow the same schema (except `kindleFireHDX-generic.json`). The format is clean and easy to parse.

2. **Good separation of concerns** — Presets are a data-only repo, consumed as a git submodule by CloudTranscode. This allows preset changes without redeploying the transcoding engine.

3. **HLS bitrate ladder exists** — Even though the resolutions need updating, having 5 HLS tiers (400k, 600k, 1M, 1.5M, 2M) shows adaptive streaming was designed in.

4. **Web preset uses `main` profile** — The `web-generic.json` preset correctly uses `Profile:main` for broad web compatibility with good compression. This should be the model for updating other presets.

5. **Preset name matches filename** — The `"name"` field in each JSON matches the filename pattern, making it easy to trace which preset was used in logs.

6. **`iphone4S-generic.json` is well-configured** — Uses `Profile:high,Level:4.1` for 1080p, which is correct for its target class of devices and provides the best compression.

7. **Git submodule architecture** — Being loaded as a submodule at `presets/` in CloudTranscode means preset updates are version-controlled and pinned to specific commits. Workers always use a known preset version.
