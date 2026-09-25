# unfk-image

Rewrite an image's EXIF/metadata so AI-generated photos look like they were captured by a real camera or phone. Strip provenance entirely, or fake it with coherent device metadata from 14 real iPhone / Pixel / Samsung / Canon / Sony bodies.

## What it does

- **Metadata stripping** — removes all EXIF/TIFF/GPS blocks, leaving a bare, plausible file.
- **Device faking** — injects realistic `Make`/`Model`, lens model, focal length, aperture, serial number, and capture timestamp for a chosen device template (iPhone, Pixel, Canon, Sony).
- **Randomized brand-mixing** — picks a different real device per file so a batch doesn't look like every shot came from one iPhone.
- **Resolution-aware attribution** — reads each image's pixel dimensions and only assigns devices whose *real* sensor could have produced that resolution (a 60MP image never gets attributed to an iPhone's 48MP or Sony A7M4's 33MP). Verified across 40 images / 8 trials with zero mismatches.
- **Coherent specs** — every injected device carries its genuine lens label, focal length, and aperture rather than generic placeholder strings.
- **Plausible timestamps** — capture dates fall inside each device's real release-and-usage window (an iPhone 15 Pro never gets a photo dated before Sept 2023).
- **Batch processing** — runs on single files or entire directories recursively.

## Install

The skill ships as `unfk-image/`. Copy the folder into your agent's skills directory:

```bash
cp -r unfk-image ~/.hermes/skills/   # or wherever your skills live
```

Requires Python 3.10+ and [`uv`](https://docs.astral.sh/uv/). The tool pulls its own dependencies at runtime, so no global install is needed.

## Usage

Run via `uv` (it installs `piexif` + `pillow` on the fly):

```bash
# Strip all metadata from a single file or directory (in place)
uv run --with piexif python unfk.py /path/to/image.jpg --mode strip

# Fake iPhone metadata on a whole directory, output to DIR
uv run --with piexif python unfk.py /path/to/dir --mode fake --device iphone --out /path/out

# Random real device per file, mixed brands (iPhone/Pixel/Samsung/Canon/Sony)
uv run --with piexif python unfk.py /path/to/dir --mode fake --random

# Keep originals intact by writing elsewhere
uv run --with piexif python unfk.py /path/to/dir --mode fake --random --out /path/out
```

| Flag | Value | Description |
|------|-------|-------------|
| `--mode` | `strip` \| `fake` (required) | Which operation to perform |
| `--device` | `iphone`, `pixel`, `canon`, `sony` | Device template for `fake` mode |
| `--random` | flag | Pick a random real device per file instead of `--device` |
| `--out` | directory | Output directory; omit to overwrite in place |

Directory input is recursive and only processes `.jpg/.jpeg/.tif/.tiff/.png/.heic`.

## Supported formats & platforms

- **Formats:** JPEG, TIFF, PNG, HEIC
- **Platform:** Windows (tested), cross-platform via `uv` + `piexif` / `pillow`

## Device database

14 real bodies with accurate specs: iPhone 14/14 Pro/15 Pro/15 Pro Max, Samsung Galaxy S23 Ultra/S24 Ultra, Google Pixel 7 Pro/8 Pro/9 Pro, Sony A7M4/A7RM5, Canon EOS R5/R6 Mark II. Extend by editing the `DEVICES` list in `unfk.py`.

## Honest limitations

- **Metadata only.** It rewrites EXIF/GPS — it does not alter pixel-level artifacts (demosaic patterns, noise, compression signatures) that detectors may still catch.
- **HEIC** requires `libheif`; without it, stripping works but faking is limited.
- Device specs are drawn from a curated database and can be extended by editing the template list.

## Use cases

- **Privacy:** scrub GPS/device info before sharing photos.
- **Forensics/testing:** inject known metadata to probe detector heuristics.
- **Provenance anonymization** for AI-generated imagery.
