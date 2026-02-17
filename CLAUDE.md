# CloudTranscode-FFMpeg-presets

## What This Is

A collection of proven FFmpeg transcoding presets used by CloudTranscode workers. Each preset is a JSON file defining codec, bitrate, resolution, and encoding parameters for specific target devices or quality levels (e.g., 360p-4.3, 1080p, iPhone4, HLS streams). These presets are referenced by name in CloudTranscode job input payloads.

## Tech Stack

- **Format**: JSON configuration files
- **Consumer**: CloudTranscode PHP workers (refs by preset name in `output_assets`)
- **FFmpeg version**: 4.2 (as of CloudTranscode)

## Quick Start

```bash
# No build or setup required — this is a data repo.

# Use a preset in CloudTranscode job input:
{
  "output_assets": [
    {
      "type": "VIDEO",
      "preset": "360p-4.3-generic",  # References 360p-4.3-generic.json
      "bucket": "...",
      "path": "...",
      "file": "..."
    }
  ]
}

# Add a new preset:
# 1. Copy an existing JSON file
# 2. Modify parameters (size, bitrate, codecs)
# 3. Test with CloudTranscode
# 4. Submit PR to this repo
```

## Project Structure

- `*.json` — FFmpeg preset definitions (one file per preset)
- `README.md` — Usage instructions
- `.github/workflows/github-backup.yml` — S3 mirror (currently broken — triggers on `develop` but default branch is `master`)

**Naming convention**: `<resolution>-<aspect-ratio>-<suffix>.json` or `<device>-<suffix>.json`
- Resolution presets: `360p-4.3-generic.json`, `1080p-generic.json`, etc.
- Device presets: `iphone4-generic.json`, `kindleFireHD-generic.json`, etc.
- HLS presets: `hls1M-generic.json`, `hls2M-generic.json`, etc.

## Preset Inventory

### Resolution Presets (7 files)

| File | Resolution | Video Bitrate | Audio Bitrate | Profile | Level | FPS |
|------|-----------|--------------|--------------|---------|-------|-----|
| `1080p-generic.json` | 1920x1080 | 5400k | 160k | baseline | 4 | 29.97 |
| `720p-generic.json` | 1280x720 | 2400k | 160k | baseline | 3.1 | 29.97 |
| `480p-generic.json` | -2x480 (auto-width) | 1200k | 128k | baseline | 3.1 | 29.97 |
| `480p-16.9-generic.json` | 854x480 | 1200k | 128k | baseline | 3.1 | 29.97 |
| `480p-4.3-generic.json` | 640x480 | 900k | 128k | baseline | 3 | 29.97 |
| `360p-16.9-generic.json` | 640x360 | 720k | 128k | baseline | 3 | 29.97 |
| `360p-4.3-generic.json` | 480x360 | 600k | 128k | baseline | 3 | 29.97 |
| `320x240-generic.json` | 320x240 | 300k | 64k | baseline | 1.3 | 21 |

### HLS Adaptive Bitrate Presets (5 files)

| File | Resolution | Video Bitrate | Audio Bitrate | Profile | Level | FPS |
|------|-----------|--------------|--------------|---------|-------|-----|
| `hls400k-generic.json` | 400x288 | 272k | 128k | baseline | 3 | 29.97 |
| `hls600k-generic.json` | 480x320 | 472k | 128k | baseline | 3 | 29.97 |
| `hls1M-generic.json` | 640x432 | 872k | 128k | main | 3.1 | auto |
| `hls1.5M-generic.json` | 960x640 | 1372k | 128k | main | 3.1 | 29.97 |
| `hls2M-generic.json` | 1024x768 | 1872k | 128k | main | 3.1 | 29.97 |

### Device Presets (10 files)

| File | Target Device(s) | Resolution | Video Bitrate | Profile | Level |
|------|-----------------|-----------|--------------|---------|-------|
| `iphone3GS-generic.json` | iPhone 3GS | 640x480 | 600k | baseline | 3 |
| `iphone4-generic.json` | iPhone 4, iPod Touch 5G/4G, iPad 1G/2G | 1280x720 | 2200k | main | 3.1 |
| `iphone4S-generic.json` | iPhone 4S+, iPad 3G+, Galaxy S2/S3 | 1920x1080 | 5000k | high | 4.1 |
| `ipodTouch-generic.json` | iPhone 1/3, iPod classic | 640x480 | 1500k | baseline | 3 |
| `appleTV2G-generic.json` | Apple TV 2nd gen | 1280x720 | 5000k | main | 3.1 |
| `appleTV3G-generic.json` | Apple TV 3rd gen, Roku HD/2 XD | 1920x1080 | 5000k | high | 4 |
| `kindleFire-generic.json` | Kindle Fire 1st gen | 1024x576 | 1600k | main | 3.1 |
| `kindleFireHD-generic.json` | Kindle Fire HD | 1280x720 | 2200k | main | 4 |
| `kindleFireHD8.9-generic.json` | Kindle Fire HD 8.9" | 1920x1080 | 5400k | main | 4 |
| `kindleFireHDX-generic.json` | Kindle Fire HDX 8.9" | 1920x1080 | 5000k | **MISSING** | **N/A** |

### Web Preset (1 file)

| File | Target | Resolution | Video Bitrate | Profile | Level |
|------|--------|-----------|--------------|---------|-------|
| `web-generic.json` | Facebook, SmugMug, Vimeo, YouTube | 1280x720 | 2200k | main | 3.1 |

## Codec Information

**Video codec:** All presets use `libx264` (H.264/AVC)
- Profiles used: `baseline` (broad compatibility), `main` (B-frames, better compression), `high` (CABAC + 8x8 DCT, best compression)
- Levels: 1.3 (QCIF) through 4.1 (1080p60)

**Audio codec:** All presets use `libfdk_aac` (Fraunhofer FDK AAC)
- Requires FFmpeg compiled with `--enable-libfdk-aac --enable-nonfree`
- Bitrates: 64k (320x240) through 160k (720p+)

**No modern codecs present:** H.265/HEVC, VP9, AV1, and Opus are not represented.

## Dependencies

**Consumed by:**
- CloudTranscode workers (`VideoTranscoder.php`) — loaded as a git submodule at `presets/` path
- The `get_preset_values()` method reads JSON files from `__DIR__ . '/../../../presets/'`
- The `validate_preset()` method checks preset existence by scanning the same directory

**No external dependencies**: Pure JSON configuration. FFmpeg codecs and options must be supported by the FFmpeg version bundled in CloudTranscode Docker images.

## API / Interface

**Preset JSON schema:**
```json
{
  "name": "Human-readable preset name",
  "description": "Description of use case",
  "size": "WIDTHxHEIGHT",
  "frame_rate": 29.97,
  "video_bitrate": "5400k",
  "audio_bitrate": "128k",
  "video_codec": "libx264",
  "audio_codec": "libfdk_aac",
  "video_codec_options": "MaxReferenceFrames:3,Profile:baseline,Level:3"
}
```

**Required fields:** `name`, `description`, `size`, `frame_rate`, `video_bitrate`, `audio_bitrate`, `video_codec`, `audio_codec`

**Optional fields:** `video_codec_options` (but strongly recommended)

**`video_codec_options` format:** Comma-separated `Key:Value` pairs. Supported keys:
- `MaxReferenceFrames` — maps to FFmpeg `-refs`
- `Profile` — maps to FFmpeg `-profile:v` (baseline, main, high)
- `Level` — maps to FFmpeg `-level` (3, 3.1, 4, 4.1, etc.)

**CRITICAL BUG:** The consumer (`VideoTranscoder.php`) parses `video_codec_options` using `explode("=", ...)` but the presets use `:` as separator. This means profile, level, and refs are **silently ignored** at runtime. See FINDINGS.md C-1.

**Consumed by**: CloudTranscode workers parse these presets and translate them into FFmpeg command-line arguments. Job input references presets by filename (without `.json` extension).

## Key Patterns

- **Device-specific presets**: Optimized for specific hardware (iPhone 4, Kindle Fire, Apple TV)
- **Resolution presets**: Generic quality levels (360p, 480p, 720p, 1080p)
- **HLS presets**: Adaptive bitrate streaming profiles (400k, 1M, 1.5M, 2M)
- **Baseline profile**: Most presets use `Profile:baseline` for broad compatibility
- **libfdk_aac audio**: All presets use `libfdk_aac` (requires FFmpeg with --enable-libfdk-aac)
- **Fixed aspect ratios**: Some presets enforce 4:3 or 16:9 (e.g., `480x360`, `854x480`)
- **CBR encoding**: All presets use constant bitrate (`-b:v`), no CRF/VBR presets exist

## Usage Patterns

**Standard video transcoding:**
```json
{
  "output_assets": [{
    "type": "VIDEO",
    "preset": "720p-generic",
    "bucket": "output-bucket",
    "path": "/videos/",
    "file": "output.mp4"
  }]
}
```

**Override preset values in job input:**
```json
{
  "output_assets": [{
    "type": "VIDEO",
    "preset": "720p-generic",
    "video_bitrate": "3000k",
    "size": "1280x720"
  }]
}
```
Job-level `video_codec`, `audio_codec`, `video_bitrate`, `audio_bitrate`, `frame_rate`, and `size` override preset values.

**HLS multi-bitrate:** Submit multiple outputs referencing different HLS presets. CloudTranscode processes them sequentially. The HLS manifest must be generated separately.

## Environment

- **Runtime**: None (data-only repo)
- **FFmpeg codec requirements**: CloudTranscode's FFmpeg build must support:
  - `libx264` (video)
  - `libfdk_aac` (audio — requires `--enable-libfdk-aac --enable-nonfree` build flag)

## Deployment

No independent deployment. This repo is consumed as a **git submodule** by CloudTranscode at the `presets/` path. The submodule is pinned to a specific commit in CloudTranscode's `.gitmodules`:
```
[submodule "presets"]
  path = presets
  url = https://github.com/sportarchive/CloudTranscode-FFMpeg-presets.git
```

To update presets in production:
1. Merge changes to `master` in this repo
2. In CloudTranscode repo: `cd presets && git pull origin master && cd .. && git add presets && git commit`
3. Rebuild and deploy CloudTranscode Docker image

## Testing

**No automated tests exist.** Validation is manual:
1. Reference a preset in a CloudTranscode job input JSON
2. Submit the job to AWS Step Functions
3. Verify the output video matches the expected resolution, bitrate, codec
4. Check video playback on target device (if device-specific preset)

**Recommended validation (not yet implemented):**
- JSON syntax check: `jq . *.json`
- Required fields check: verify all 8 required fields present
- FFmpeg codec availability check: validate codec names against FFmpeg build
- Bitrate sanity check: video_bitrate should be reasonable for resolution

## Gotchas

- **CRITICAL — Parser mismatch**: `video_codec_options` uses `:` separator but `VideoTranscoder.php` parses with `=`. Profile, level, and refs are silently ignored. See FINDINGS.md C-1.
- **BROKEN — `kindleFireHDX-generic.json`**: Has trailing comma (invalid JSON) and missing `video_codec_options`. Will cause `BAD_PRESET_FORMAT` error at runtime.
- **libfdk_aac dependency**: All presets require FFmpeg compiled with `--enable-libfdk-aac --enable-nonfree`. Standard FFmpeg packages don't include this.
- **Outdated device presets**: 8 of 23 presets target devices from 2009-2013. They work but produce unnecessarily low quality for modern devices.
- **HLS non-standard resolutions**: HLS presets use non-standard resolutions that don't match Apple's HLS Authoring Spec. `hls2M` outputs 4:3 (1024x768).
- **HLS presets**: These are for individual HLS renditions only. Multi-bitrate HLS requires multiple presets and an HLS manifest generated separately.
- **No DASH/CMAF presets**: Only HLS streaming presets exist. Android-native adaptive streaming (DASH) is not supported.
- **Frame rate inconsistency**: Mix of 29.97, 30, 21, and "auto" across presets. HLS ladder presets should use consistent frame rates.
- **1080p baseline profile**: The generic 1080p preset uses baseline profile, wasting ~15-25% bandwidth vs high profile.
- **CBR only**: All presets use constant bitrate. No CRF/VBR presets exist. Sports content would benefit from variable bitrate.
- **No validation**: No CI, no schema, no linting. Invalid presets only fail at runtime.
- **Backup workflow broken**: `.github/workflows/github-backup.yml` triggers on `develop` branch but default branch is `master`.
- **`480p-generic.json` auto-width**: Uses `-2x480` syntax which bypasses the consumer's enlargement check.
- **Generic suffix**: All files end in `-generic.json`. No specialized high-quality or device-optimized variants.
