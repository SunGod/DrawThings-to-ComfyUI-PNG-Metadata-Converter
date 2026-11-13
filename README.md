# Draw Things-to-ComfyUI PNG Metadata Converter
**A handy little Python script that will take a directory full of PNGs exported from DrawThings, and convert their metadata so that the same PNGs can also be used in ComfyUI.**



**Note:**
If you want to export all your images and videos from a DrawThings project, use [ComfyThings](https://github.com/SunGod/ComfyThings) instead. 


**Version 1.01** · Python 3.9+ · Windows 10/11 and macOS

Convert existing PNGs exported from [Draw Things](https://drawthings.ai/) into images that also contain native [ComfyUI](https://github.com/comfyanonymous/ComfyUI) `workflow` and `prompt` metadata. Drag a converted PNG into ComfyUI to inspect a reconstructed, best-effort workflow **without losing the original Draw Things generation metadata or image data**.

The converter is a **standalone script**: it reads individual PNG files or folders of PNGs. It **does not** read SQLite databases, regenerate images, or convert model weights. It accepts **any model filename**; identification is used to improve the workflow, not to determine whether an image may be processed.

## Requirements

- **Python 3.9 or later** on Windows 10/11 or macOS (including macOS 26/27). No `pip install` or additional Python packages are required.
- PNGs exported from Draw Things that contain readable Draw Things generation JSON in XMP `exif:UserComment`.
- ComfyUI is **not needed to run the converter**. A vanilla ComfyUI installation is sufficient to inspect supported generated nodes; actually executing a workflow can require additional models or compatible node types.
- Free disk space for staging, rollback, and optional backups. Images are processed one at a time.

Download [`convert_metadata_drawthings_to_comfyui.py`](convert_metadata_drawthings_to_comfyui.py). Run the commands below from a directory containing the script, or supply its full path.

## Quick start

Try a **copy** of a few PNGs before processing an important image collection.

**Windows — Command Prompt or PowerShell:**

```cmd
py convert_metadata_drawthings_to_comfyui.py . --dry-run --verbose
py convert_metadata_drawthings_to_comfyui.py . --backup --verify --report conversion.csv
```

**macOS — Terminal:**

```bash
python3 convert_metadata_drawthings_to_comfyui.py . --dry-run --verbose
python3 convert_metadata_drawthings_to_comfyui.py . --backup --verify --report conversion.csv
```

`.` processes PNGs in the current directory. Supply a specific file or directory path instead; names with spaces must be quoted. Subdirectories are scanned **only** with `--recursive` (`-r`).

```bash
# One image
python3 convert_metadata_drawthings_to_comfyui.py photo.png --backup --verify

# Folder with a space in its name
python3 convert_metadata_drawthings_to_comfyui.py "DrawThings iPhone created 2026.04.07" --backup --verify

# Include subfolders
python3 convert_metadata_drawthings_to_comfyui.py ./images --recursive --backup --verify

# Replace existing ComfyUI metadata using the original Draw Things metadata
python3 convert_metadata_drawthings_to_comfyui.py ./images --overwrite --backup --verify

# Do not contact Civitai
python3 convert_metadata_drawthings_to_comfyui.py ./images --no-civitai --backup --verify
```

## Optional image renaming: `--rename-images`

Draw Things can export PNGs with long, prompt-derived filenames. Use **`--rename-images`** to rename them using the same timestamp-based style as ComfyThings:

```text
DrawThings_YYYYMMDD_HHMMSS_ffffff.png
```

Example: `DrawThings_20260407_143012_456789.png`.

The date and time come from the PNG file's **filesystem Date Created**, including its six-digit **microsecond** component—not from the prompt, seed, a hash, or Date Modified. The file creation timestamp may reflect the time of manual export or copying rather than the actual generation time. A suffix of `_000000` is **valid** when the available creation timestamp has zero microseconds; the converter never invents fractional seconds.

With `--rename-images`:

- **First-time conversion:** Add ComfyUI metadata, then rename the PNG.
- **Already-converted PNG:** Skip metadata conversion, but rename it if necessary. The PNG bytes and existing workflow are not rewritten.
- **Existing timestamp-style filename:** Recompute the expected name from the actual Date Created, **including microseconds**, and rename if any component differs—even if the filename already looks correct.
- **Identical timestamps:** Add `_1`, `_2`, and so on to prevent overwrites (for example, `DrawThings_20260407_143012_000000_1.png`).
- **Already correct:** Leave the filename unchanged. The option is opt-in; without it, original names are kept.

**Preview filename changes and write a CSV report without changing PNGs:**

```bash
python3 convert_metadata_drawthings_to_comfyui.py "DrawThings iPhone created 2026.04.07" --rename-images --dry-run --verbose --report rename_preview.csv
```

The CSV has `file`, `status`, `message`, and `warnings` columns. The `message` indicates proposed names (for example, `would rename only; rename to DrawThings_...png`) or first-time conversions (`would convert; rename to ...`). **The report is written even in dry-run mode; the images are not changed.**

**Apply conversion and/or renaming:**

```bash
python3 convert_metadata_drawthings_to_comfyui.py "DrawThings iPhone created 2026.04.07" --rename-images --backup --verify --report conversion.csv
```

Do **not** specify `--overwrite` when you only want to rename previously converted PNGs. On Windows and macOS, the renamer uses the filesystem creation timestamp; on Linux, rename support requires an available filesystem birth timestamp and does not substitute Date Modified.

## Command-line options

| Option | Behavior |
| --- | --- |
| `path` | One PNG or a directory; defaults to `.`. |
| `--recursive`, `-r` | Process PNGs in subdirectories. |
| `--backup` | Keep numbered backup files (`.bak1.png`, `.bak2.png`, etc.). |
| `--dry-run` | Preview operations without changing PNG files. |
| `--overwrite` | Recreate `workflow` and `prompt` metadata in previously converted PNGs. |
| **`--rename-images`** | Rename PNGs using filesystem Date Created, including microseconds; works for new and existing conversions. |
| `--verify` | Validate output PNG, original image data and XMP, and generated ComfyUI metadata. |
| `--report FILE.csv` | Save per-file statuses, messages, and warnings, including proposed renames during dry-run. |
| `--verbose` | Display per-file outcomes and warnings. |
| `--quiet` | Suppress routine console output. |
| `--no-progress` | Disable progress-bar display. |
| `--no-civitai` | Disable online model identification; use the embedded catalog and offline fallbacks. |
| `--version` | Display the script version. |
| `--help` | Display all command-line options. |

Options can be combined. Renaming is never enabled by default. The script reports per-file failures and exits with a nonzero status when errors occur. Non-PNG files and numbered backup PNGs are ignored during directory scans; PNGs without Draw Things generation metadata are skipped.

## What the conversion preserves and reconstructs

The converter reads Draw Things generation JSON from the PNG's existing XMP data and adds **ComfyUI `workflow` and `prompt` JSON**. Where there is enough information, it reconstructs a connected graph using standard ComfyUI nodes. Depending on the model and the available metadata, the graph can represent:

- Positive and negative prompts, seed, steps, guidance, dimensions, sampler/scheduler, and checkpoint filename.
- Supported LoRAs and their stage/weight semantics, CLIP-related information, and VAE decoding options (including tiled decoding where appropriate).
- Architecture-dependent text encoders, model sampling and guidance, plus recognizable Hires Fix, upscaling, or other processing stages when the available metadata justifies them.
- A visible **ComfyUI Note** containing the original generation information, the **complete original Draw Things JSON**, model-family identification, and any uncertainty or approximations.

Model identification first checks an embedded catalog of exact Draw Things model filenames. When a confident identification is unavailable, the script can query **Civitai** for corroborating model/version/file information. The model filename is used in that request; **images, prompts, and full generation metadata are not uploaded**. Civitai may be unavailable or return no reliable match. Use `--no-civitai` for offline operation. Unknown model families still receive best-effort conversion.

**This is a metadata converter, not a reproduction engine.** Some samplers, schedules (such as Trailing), refiners, advanced conditioning, and architecture-specific behavior cannot be represented exactly by vanilla ComfyUI. These distinctions are explained in the Note rather than silently asserted as equivalent. A connected graph does not guarantee it can execute without compatible model files or manually corrected node parameters. The script does not install models or automatically make incompatible model formats runnable.

## Image integrity and file safety

- **Original Draw Things XMP is preserved**, including original model identifiers and generation settings.
- **Original PNG compressed image (`IDAT`) bytes are preserved.** Images are not decoded or recompressed.
- Files are modified **in place**, not replaced with a newly created image. Rename-only operations do not change any PNG bytes.
- On **Windows**, native creation, modification, and access timestamps are captured, restored, and checked. On **macOS**, modification and access times are restored and the original file object (and its creation/birth date) is retained.
- A staged image and rollback copy protect against ordinary write errors. `--backup` keeps persistent numbered backups; an already existing backup is not overwritten.
- PNG chunk CRCs and embedded workflow structure are checked during processing; `--verify` also checks the written image's preservation invariants.
- Existing ComfyUI metadata is skipped unless `--overwrite` is specified. With `--rename-images`, already converted PNGs can instead be renamed without rebuilding their workflows.

**In-place writes are not power-loss atomic.** For valuable collections, first try a small copied directory, then use `--backup --verify` and retain a separate backup of the originals.

## Limitations and troubleshooting

**A workflow opens but won't execute.** The referenced checkpoint, LoRAs, text encoders, VAEs, or other model assets may not be installed or usable by vanilla ComfyUI. This does not make the original metadata invalid. Model filenames inside ComfyUI nodes may require manual adjustment to match local filenames. The converter does **not** change model weights or automatically rename `.ckpt` files to `.safetensors`.

**Some settings are informational only.** An unfamiliar sampler or a source stage without a suitable ComfyUI node may require a best-effort approximation or a Note-only presentation. The untouched original XMP remains the authoritative record.

**An image was skipped.** It may already contain `workflow`/`prompt` metadata, or may lack readable Draw Things generation JSON in XMP. Use `--verbose` or `--report` to identify the reason. Use `--overwrite` only when you explicitly want to rebuild a previously converted workflow.

**`--rename-images` leaves `_000000` intact.** If Date Created actually has zero microseconds, the name is correct. It is not a fingerprint and does not need a nonzero value.

**Renaming failed.** The filesystem must provide an accessible creation/birth timestamp. Renaming never overwrites an occupied destination name. Check file permissions and destination collisions; when possible, move files to a filesystem that preserves their original creation dates.

**Conversion is slow.** PNGs are scanned and copied sequentially, and Civitai lookups can add latency when model names are not cached. Use `--no-civitai` if you prefer predictable offline runs.

## Changes from v1.00 → v1.01

- **Architecture-aware workflow reconstruction:** Added more accurate ComfyUI workflow generation for recognized model architectures, including SDXL, FLUX, Z-Image, and Qwen Image variants.
- **Expanded model identification:** Added an embedded Draw Things model catalog to identify model families using their original filenames.
- **Civitai integration:** Added online model identification for models not recognized by the embedded catalog.
- **Offline operation:** Added `--no-civitai` to disable online model lookups.
- **Improved handling of unknown models:** Unrecognized models continue to receive best-effort ComfyUI metadata rather than being rejected.
- **Comprehensive metadata Notes:** Added the complete original Draw Things generation JSON, model identification, generation settings, and conversion warnings directly into the ComfyUI workflow.
- **Improved sampler handling:** Unrecognized samplers no longer automatically cause conversion failure. The original sampler is documented, and an explicitly identified approximation is used where appropriate.
- **Architecture-specific metadata:** Improved handling of text encoders, VAE dependencies, LoRAs, model shifts, and guidance settings.
- **Advanced generation stages:** Expanded reconstruction of Hires Fix, upscaling, and other multi-stage operations when sufficient metadata is available.
- **Improved workflow layout:** Better node positioning, clearer informational Notes, and consistent placement of output nodes.
- **Optional image renaming:** Added `--rename-images` to rename first-time or previously converted PNGs to the `DrawThings_YYYYMMDD_HHMMSS_ffffff.png` convention using filesystem Date Created, including microseconds, with collision protection and dry-run CSV previews.

---

**Release 1.01** — standalone metadata conversion of existing Draw Things PNGs; no project-database access required.
