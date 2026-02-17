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

**Naming convention**: `<resolution>-<aspect-ratio>-<suffix>.json` or `<device>-<suffix>.json`
- Resolution presets: `360p-4.3-generic.json`, `1080p-generic.json`, etc.
- Device presets: `iphone4-generic.json`, `kindleFireHD-generic.json`, etc.
- HLS presets: `hls1M-generic.json`, `hls2M-generic.json`, etc.

## Dependencies

**Consumed by:**
- CloudTranscode workers (`TranscodeAssetActivity.php`)

**No external dependencies**: Pure JSON configuration. FFmpeg codecs and options must be supported by the FFmpeg version bundled in CloudTranscode Docker images.

## API / Interface

**Preset JSON schema:**
```json
{
  "name": "Human-readable preset name",
  "description": "Description of use case",
  "size": "WIDTHxHEIGHT",           # e.g., "1920x1080", "640x480"
  "frame_rate": 29.97 | "auto",     # Numeric FPS or "auto"
  "video_bitrate": "5400k",         # e.g., "600k", "5400k"
  "audio_bitrate": "128k",          # e.g., "128k", "160k"
  "video_codec": "libx264",         # e.g., "libx264", "libvpx"
  "audio_codec": "libfdk_aac",      # e.g., "libfdk_aac", "aac"
  "video_codec_options": "MaxReferenceFrames:3,Profile:baseline,Level:3"  # Codec-specific
}
```

**Consumed by**: CloudTranscode workers parse these presets and translate them into FFmpeg command-line arguments.

<!-- Ask: How are these presets loaded by CloudTranscode workers? Are they bundled in the Docker image, fetched from S3, or pulled from this repo at runtime? -->

## Key Patterns

- **Device-specific presets**: Optimized for specific hardware (iPhone 4, Kindle Fire, Apple TV)
- **Resolution presets**: Generic quality levels (360p, 480p, 720p, 1080p)
- **HLS presets**: Adaptive bitrate streaming profiles (400k, 1M, 1.5M, 2M)
- **Baseline profile**: Most presets use `Profile:baseline` for broad compatibility
- **libfdk_aac audio**: Most presets use `libfdk_aac` (requires FFmpeg with --enable-libfdk-aac)
- **Fixed aspect ratios**: Some presets enforce 4:3 or 16:9 (e.g., `480x360`, `854x480`)

## Environment

- **Runtime**: None (data-only repo)
- **FFmpeg codec requirements**: CloudTranscode's FFmpeg build must support:
  - `libx264` (video)
  - `libfdk_aac` (audio)
  - Any codecs referenced in presets

## Deployment

No deployment. This repo is consumed as a reference or bundled into CloudTranscode workers.

<!-- Ask: Are these presets copied into the CloudTranscode Docker image at build time, or loaded dynamically from S3/GitHub? -->

## Testing

**Manual testing:**
1. Reference a preset in a CloudTranscode job input JSON
2. Submit the job to AWS Step Functions
3. Verify the output video matches the expected resolution, bitrate, codec
4. Check video playback on target device (if device-specific preset)

**Validation:**
- Ensure `video_codec` and `audio_codec` are supported by FFmpeg 4.2
- Ensure `size` dimensions are valid (width x height)
- Ensure `video_bitrate` and `audio_bitrate` are formatted correctly (e.g., "600k", not "600")

<!-- Ask: Is there a script to validate all presets against FFmpeg's supported codecs/options? -->

## Gotchas

- **libfdk_aac dependency**: Most presets use `libfdk_aac`, which requires FFmpeg compiled with `--enable-libfdk-aac`. If CloudTranscode's FFmpeg lacks this, jobs will fail. Alternative: use `aac` (built-in) instead.
- **Outdated device presets**: iPhone 3GS, iPhone 4, iPod Touch presets are for legacy devices (2010-2012 era). Modern devices support higher resolutions/bitrates.
- **HLS presets**: These are for individual HLS renditions. Multi-bitrate HLS requires multiple presets and an HLS manifest.
- **video_codec_options format**: Uses colon-separated key:value pairs (e.g., `Profile:baseline,Level:3`). CloudTranscode workers must parse this format correctly.
- **No validation**: Preset JSON files are not validated on commit. Invalid presets will only fail at runtime in CloudTranscode.
- **Frame rate "auto"**: Preserves source video frame rate. Some presets hardcode 29.97 (NTSC).
- **Generic suffix**: Most files end in `-generic.json`. No specialized variants (e.g., no high-profile or high-bitrate versions of device presets).

<!-- Ask: Should we deprecate legacy device presets (iPhone 3GS, iPhone 4, etc.) and add modern device profiles? -->
<!-- Ask: Is there documentation mapping which presets are used for which bFAN workflows (e.g., social media vs in-app playback)? -->