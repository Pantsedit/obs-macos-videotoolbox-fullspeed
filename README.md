# VideoToolbox Full Speed for OBS Studio (macOS)

I wanted to record in 4K60 and stream at the same time, from one OBS on one Mac (a Mac
Studio M2 Max). OBS kept showing *Encoding overloaded*, even though the same Mac could
record lots of 4K ProRes streams at once. So I dug into why.

It turns out the hardware was never the problem. OBS sets up Apple's HEVC and H.264
encoders in two ways that hold them back. Change both, and my M2 Max records 4K60 HEVC
and streams 4K60 HEVC at the same time, with room left for a 1080p60 output on top.
This plugin makes those two changes for you.

## What it adds

Two encoders, next to the built-in Apple ones:

- **Apple VT HEVC Hardware (Full Speed)**
- **Apple VT H.264 Hardware (Full Speed)**

They are the same Apple hardware encoders OBS already uses, with two settings changed.

## Results (Mac Studio M2 Max, real gameplay, 4K60 canvas)

| Workload | Stock OBS | With this plugin |
|---|---|---|
| 4K60 HEVC recording + 1440p60 H.264 stream | ❌ | ✅ |
| 4K60 HEVC recording + 4K60 HEVC stream | ❌ ~42 fps each | ✅ 60 fps each |
| 2x 4K60 HEVC + 1x 1080p60 HEVC | ❌ 75 to 81% frames skipped | ✅ 0.0% skipped |
| 2x 1440p60 H.264 (synthetic test) | ❌ ~50 fps each | ✅ ~113 fps each (capacity) |

Tested on OBS 32.1.2 and OBS 33.0 (master).

## Why stock OBS struggles

Apple Silicon Max chips have **two** H.264/HEVC encode engines. Each can encode roughly
90+ frames per second of 4K HEVC, which is enough for one 4K60 output each with room to
spare. OBS sets up its encoder sessions in a way that wastes that:

1. **Real-time is switched off.** The encoder treats the job like exporting a file and
   takes its time, reaching only ~50 fps at 4K. OBS turned it off in 2022 to work around
   frame drops on early Apple Silicon (obs-studio issue #5840). Those drops did not come
   back in any of our tests on current macOS.
2. **OBS tells the encoder the frame rate in advance.** With that hint set, two 4K
   sessions slow each other down to ~36 fps each. Without it, each runs at full speed.
   Your recording and stream frame rate does not change, because OBS timestamps every
   frame.

**Both** changes are needed. In OBS on an M2 Max, 2x 4K60 HEVC ran at about 42 fps each
with only one of them, and at a clean 60 fps with both.

## Install

1. Quit OBS.
2. Run the installer package, **or** copy `obs-macos-videotoolbox-fullspeed.plugin` into
   `~/Library/Application Support/obs-studio/plugins/`.
3. Start OBS. The new encoders appear in **Settings → Output** (Output Mode: Advanced).

Requires OBS Studio 31.1 or newer on macOS 12+ (Apple Silicon recommended; CBR needs macOS 13+).

## Use

- In **Settings → Output → Recording** and **Streaming**, pick
  **Apple VT HEVC Hardware (Full Speed)** (or the H.264 one).
- **Use the Full Speed encoders for every high resolution output.** A single stock
  Apple encoder running at 4K alongside them still slows every 4K session down.
- Also works for extra outputs from plugins that let you choose an encoder
  (e.g. obs-multi-rtmp).

### Settings

All the usual encoder settings are there, plus:

| Setting | Default | What it does |
|---|---|---|
| **Optimize for multiple outputs (recommended)** | On | Lets the encoders keep up with live video and run several outputs at full speed together. Your frame rate is unchanged. Untick for exactly the standard Apple VT behavior. |
| **Log encoder timing (troubleshooting)** | Off | Writes encoder timing to the OBS log every 10 seconds. |

Under the hood the first option turns real-time encoding on and stops sending the
expected frame rate. For testing, the two can be overridden individually by adding
`"realtime"` and `"frame_rate_hint"` to the encoder settings JSON in your profile.

### Quality

Measured on the same 30 seconds of 4K gameplay:

- **Constant quality (CRF 70):** same file size, and a difference of 0.15 dB PSNR, which
  is too small to see.
- **Fixed bitrate (CBR 40 Mbps):** about 0.7 dB lower, roughly what 8 to 10% less bitrate
  would give. If that matters to you, raise the bitrate slightly.

## Limits (M2 Max)

- **Two 4K60 HEVC outputs is the ceiling.** A third 4K60 output has to share an engine
  and drops to ~40 fps. A 1080p60 third output fits; 1440p60 does not.
- **H.264 cannot do 4K60 on this chip** (~53 fps even alone). Use HEVC for 4K,
  H.264 at 1440p and below.
- Chips with more engines (Ultra) should scale further; untested.

## Uninstalling

If you remove the plugin, any profile that uses a Full Speed encoder will show a
missing encoder error. Pick the stock Apple encoder again in Settings → Output.

## Upstream

`src/vt-encoder.c` is a copy of OBS Studio's `mac-videotoolbox` encoder with the changes
above; see [UPSTREAM.md](UPSTREAM.md). A weekly workflow opens an issue when OBS changes
that file so fixes can be ported. If OBS adopts these changes itself, this plugin
becomes unnecessary.

## Author

[n8made.com](https://n8made.com/)

## License and credits

GPL-2.0-or-later. Based on the `mac-videotoolbox` plugin from
[OBS Studio](https://github.com/obsproject/obs-studio).
