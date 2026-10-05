# pdf-inspector CLI releases (fork)

This fork of [firecrawl/pdf-inspector](https://github.com/firecrawl/pdf-inspector)
adds no features. It only publishes prebuilt `pdf2md` and `detect-pdf` binaries
on the [Releases](../../releases) page.

## Branches

- `main`: an exact mirror of upstream `main`. Never commit to it, so syncs always fast-forward.
- `release` (default branch): holds only this README and `.github/workflows/release.yml`.
  The workflow checks out `main` and builds it.

## Release flow

1. Sync `main` from upstream: the **Sync fork** button on the `main` branch view, or
   `gh repo sync gechelberger/pdf-inspector --branch main`.
2. Trigger a build: `gh workflow run release.yml --repo gechelberger/pdf-inspector`,
   or wait for the daily scheduled run.

The workflow reads the version from `main`'s `Cargo.toml`. If a release `v<version>` already
exists it does nothing, so new releases appear only when upstream bumps the version.
To rebuild an existing version, run it with `-f force=true`. To build a different
branch, tag, or SHA, use `-f ref=<ref>`.

### Fully automatic (optional)

Add a repo secret `SYNC_TOKEN` (a fine-grained PAT for this repo with **Contents: write** and
**Workflows: write**). The scheduled run will then sync `main` from upstream on its own before
it checks the version. `GITHUB_TOKEN` can't do this, because it isn't allowed to push upstream's
changes to `.github/workflows/`.

## Artifacts

| Target | Notes |
|---|---|
| `x86_64-unknown-linux-gnu`, `aarch64-unknown-linux-gnu` | glibc ≥ 2.17 |
| `x86_64-unknown-linux-musl`, `aarch64-unknown-linux-musl` | fully static |
| `aarch64-apple-darwin`, `x86_64-apple-darwin` | macOS |
| `x86_64-pc-windows-msvc`, `aarch64-pc-windows-msvc` | Windows (`.zip`) |

These archives do text extraction only. `SHA256SUMS` is attached to each release.

### OCR builds (`-ocr` archives)

| Target | Bundled runtimes |
|---|---|
| `x86_64-unknown-linux-gnu-ocr`, `aarch64-unknown-linux-gnu-ocr` | PDFium native-v7988, ONNX Runtime 1.27.0 |
| `aarch64-apple-darwin-ocr` | same |
| `x86_64-pc-windows-msvc-ocr` | same |

These are built with `--features ocr`. The PDFium and ONNX Runtime shared libraries sit next to
`pdf2md`, where it finds them automatically, so you don't need `PDFIUM_LIB_PATH` or `ORT_DYLIB_PATH`.
Keep the libraries in the same directory as the binary. Their licenses are in `licenses/`.

```bash
pdf2md scan.pdf --ocr auto     # OCR only pages that need it
pdf2md scan.pdf --ocr force    # OCR every page
```

- The first OCR run downloads the PP-OCRv6 Small models (~31 MB) into the platform cache dir.
  Set `PDF_INSPECTOR_MODEL_CACHE` to change where they go. For hosts without internet access,
  see [Offline OCR models](#offline-ocr-models).
- Linux OCR builds need glibc. ONNX Runtime doesn't ship musl builds.
- No OCR builds exist for Intel macOS or Windows ARM64, because upstream PDFium/ONNX Runtime
  don't publish matching binaries for them.
- macOS: if you download with a browser, clear the quarantine flag first or the unsigned
  binaries and dylibs won't load: `xattr -dr com.apple.quarantine pdf-inspector-*-ocr/`.

### Offline OCR models

Each release includes `pdf-inspector-ocr-models-<id>-<revision>.tar.gz`, a mirror of the exact model
set that release's `pdf2md` pins. CI checks every file against those pinned SHA-256 hashes, and `pdf2md`
checks them again each time it loads them. The archive uses the model cache layout,
`<id>/<revision>/<files>`, plus the models' Apache-2.0 license. There are two ways to use it:

```bash
# 1. Extract into a model cache root
tar xzf pdf-inspector-ocr-models-*.tar.gz -C /opt/pdf-inspector-models
export PDF_INSPECTOR_MODEL_CACHE=/opt/pdf-inspector-models
pdf2md scan.pdf --ocr auto --ocr-offline

# 2. Point directly at the model directory
pdf2md scan.pdf --ocr auto --ocr-offline \
  --ocr-model-dir /opt/pdf-inspector-models/pp-ocrv6-small/oar-ocr-v0.7.0
```

If you extract into the default cache root instead, you need neither the variable nor the flag.
The default root is `~/.cache/pdf-inspector/models` on Linux,
`~/Library/Caches/pdf-inspector/models` on macOS, and `%LOCALAPPDATA%\pdf-inspector\models` on Windows.
`--ocr-offline` makes a missing or corrupt model an error instead of a download attempt.
Windows 10+ ships `tar`, so the same command works there.

Upstream pins these runtime versions in `docs/ocr-runtime.md`. If a future upstream release
changes them, update the URLs and SHA-256 hashes in the `build` matrix of `release.yml`.
