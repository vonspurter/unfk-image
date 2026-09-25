---
name: unfk-image
description: >
  Rewrite an image's EXIF/metadata to make AI-generated photos look like they were
  captured by a real camera or phone. Use when the user wants to "un-AI" an image,
  scrub provenance, hide where/what-device a photo came from, or fake device metadata.
  Triggers on phrases like "make this look real", "remove the EXIF", "fake the camera
  info", "pretend it was shot on an iPhone", or "hide the location".
user-invocable: true
allowed-tools: [file.read, file.write, terminal]
requirements: [python3.10+, uv, piexif, pillow]
---

# unfk-image — Workflow Implementation

## The trigger model (why this description is written this way)

The agent decides whether to load `unfk-image` almost entirely from the
`description` field above — not from the body or folder name. It fires on:

- "make this look like a real photo" / "un-AI this"
- "remove the EXIF metadata" / "scrub the provenance"
- "fake the camera info" / "pretend it was shot on an iPhone"
- "hide where this was taken" / "strip the GPS location"

If none of those match, do not load the skill. It is a **Type 3 skill** (instructions
+ runnable code) — it executes a script with side effects on files, not just prose.

## Workflow: strip mode

Numbered steps are followed in order; prose gets summarized.

1. Confirm the target path is an image file or directory. If a directory, process recursively and only `.jpg/.jpeg/.tif/.tiff/.png/.heic`.
2. For each supported file, call `piexif.remove(src, dst)` to delete every EXIF/TIFF/GPS block.
3. Verify the result with `piexif.load(dst)` — confirm no `Make`/`Model`/GPS tags remain in any IFD.
4. Report how many files were stripped and where they live (in place or under `--out`).

## Workflow: fake mode (single device)

1. Read the requested `--device` (`iphone`, `pixel`, `canon`, `sony`). Reject unknown values with a clear error listing valid options.
2. For each supported file, build an EXIF blob from the device template: 0th IFD `Make`/`Model`, ExifIFD `DateTimeOriginal`, `LensModel`, `FocalLength`, `FNumber`.
3. Write the blob with `piexif.insert(blob, src, dst)`. Never touch pixel data.
4. Report each file and the device it was attributed to.

## Workflow: fake mode (random — brand-mixed + resolution-coherent)

This is the mode that matters for realism. Do not skip step 2.

1. Read each image's actual pixel dimensions (`width × height`) via PIL, before writing anything.
2. Filter the device database to only bodies whose **real** max photo resolution can produce those pixels. A 60MP image is never attributed to an iPhone (48MP) or Sony A7M4 (33MP).
3. From the eligible set, pick one real device at random — this naturally mixes brands across a batch.
4. Inject that device's coherent specs: matching lens model, real focal length/aperture, and a capture timestamp inside its genuine release-and-usage window (an iPhone 15 Pro never gets a pre-Sept-2023 date).
5. Write with `piexif.insert`, then verify the written metadata round-trips through `piexif.load`.

## Output contract (what other skills can chain against)

After running, emit a machine-checkable summary so a coordinator skill can pass it along:

```json
{
  "mode": "strip" | "fake",
  "device": null | "iphone" | "pixel" | "canon" | "sony" | "<random model>",
  "files_processed": <int>,
  "outputs": [
    {"path": "/abs/path/to/image.jpg", "attributed_to": null | "iPhone 15 Pro"}
  ]
}
```

A downstream skill (e.g. a "publish to CMS" or "run detector check" skill) can read this
to confirm the metadata was rewritten before proceeding.

## Anti-patterns (what NOT to do)

- **Never claim an image is camera-authentic.** This rewrites *metadata only* — pixel-level artifacts (demosaic patterns, noise, compression signatures) are untouched and detectors may still catch them. Say so explicitly if asked "is this now undetectable?"
- **Never attribute a 60MP image to a 48MP body.** Resolution mismatch is the single most obvious tell in random mode. Always filter by real sensor resolution first.
- **Never write a generic lens string** like `"unknown"` or `"generic lens"`. Use each device's genuine lens label — detectors cross-check `LensModel` against `Model`.
- **Never date an image before its device existed.** Timestamps must fall inside the release-and-usage window for that exact model.
- **Never overwrite originals unless asked.** Default to in-place only when no `--out` is given; otherwise write elsewhere and keep originals intact.

## How it chains with other skills

```
fetch-image ──▶ unfk-image ──▶ detector-check ──▶ publish-to-cms
   (download)     (rewrite EXIF)    (verify realism)   (post somewhere)
```

- **Before it:** a `fetch-image` or `ingest-photo` skill can supply the input files.
- **After it:** a `detector-check` skill re-reads the rewritten files to confirm they survive an AI-detection heuristic; a `publish-to-cms` skill posts them with provenance scrubbed.
- **Coordinator pattern:** a planner skill classifies the request ("does this need metadata rewriting?") and delegates here, then verifies the output contract above before continuing.

## Publish checklist (for skills websites)

- [x] `name` matches folder, kebab-case (`unfk-image`)
- [x] `description` carries ≥3 concrete trigger phrases in user language
- [x] Workflow is a numbered list; every step is an action, not a topic
- [x] Output contract is a literal JSON skeleton downstream skills can validate
- [x] Anti-patterns prevent the observed failure modes (resolution mismatch, fake dates)
- [x] Tested: fires on "un-AI this", "remove EXIF", "fake camera info"
