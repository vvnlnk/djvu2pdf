# djvu2kindle

Convert a scanned DjVu book into a compact, searchable PDF suitable for a
Kindle Scribe or iPad.

`ddjvu -format=pdf` flattens every page into one big raster image, which throws
away what makes DjVu small (a 112 MB PDF from a 22 MB DjVu). This script keeps
DjVu's own layer split instead:

| DjVu layer | Goes into the PDF as |
|---|---|
| Text mask (JB2, 1-bit, 600 dpi) | Lossless JBIG2 at native resolution |
| Background (IW44, illustrations) | JPEG at `--dpi`, colour kept (greyscale if the scan is grey) |
| Foreground colours (IW44, low-res) | Small JPEG shown through the text mask |
| *(none in DjVu)* | Invisible OCR text layer (Tesseract), so the PDF is searchable |
| Outline / bookmarks | PDF bookmarks |

For the test book (336 pages, 22.5 MB DjVu) the result is about 32 MB.

## Prerequisites

### What is needed

| Tool | Used for |
|---|---|
| `ddjvu`, `djvudump`, `djvused`, `djvuextract`, `djvumake` (DjVuLibre) | rendering layers, reading page structure, outline |
| `jbig2` (the `jbig2enc` encoder) | lossless JBIG2 compression of the text mask |
| `ocrmypdf` | adds the OCR text layer |
| `tesseract` + language data | OCR engine; install the data for every language you pass to `--lang` |

All of these must be on your `PATH`; the script checks for them at startup and
exits with `missing required tool: <name>` otherwise.

Only Linux (Ubuntu) was tested, with DjVuLibre 3.5.28, jbig2enc 0.28 and
ocrmypdf 15.2. The macOS and Windows steps below follow each project's normal
installation route but have **not been verified** with this script.

### Linux (Ubuntu / Debian)

```bash
sudo apt install djvulibre-bin jbig2 ocrmypdf tesseract-ocr-rus tesseract-ocr-eng
```

The apt package that provides the `jbig2enc` encoder is called `jbig2`
(`jbig2enc` is not a package name). If your distribution has no package, build
it from <https://github.com/agl/jbig2enc> (needs `libleptonica-dev`, autoconf,
automake, libtool, a C++ compiler).

### macOS (Homebrew)

```bash
brew install djvulibre jbig2enc ocrmypdf tesseract tesseract-lang
```

`tesseract-lang` installs all Tesseract language packs (including Russian).
`ocrmypdf` pulls in Ghostscript and its own Python, so pick the Python you
install `pikepdf`/`Pillow` into (see below) accordingly, or simply use
`python3 -m pip install --user pikepdf Pillow` with a Homebrew Python.

### Windows

The easiest route is **WSL 2**: install Ubuntu from the Microsoft Store, open
it, and follow the Linux steps above. Everything then behaves as tested.

Native Windows is possible but needs more manual work, because `jbig2enc` has
no official Windows package:

1. **DjVuLibre**: install the Windows package from <https://djvu.org/> (or
   `choco install djvulibre`) and add its folder (typically
   `C:\Program Files (x86)\DjVuLibre`) to `PATH`.
2. **Tesseract**: install the UB Mannheim build (`choco install tesseract` or
   its installer), choosing the languages you need (e.g. Russian), and add it
   to `PATH`.
3. **Ghostscript** (`choco install ghostscript`): `ocrmypdf` requires it.
4. **ocrmypdf**: `py -m pip install ocrmypdf`
5. **jbig2enc**: build `jbig2.exe` yourself (for example with MSYS2 or
   vcpkg + CMake, from <https://github.com/agl/jbig2enc>) and put it on
   `PATH`. If you cannot get `jbig2.exe`, the script will not run as written;
   use WSL instead.

Run the script with `py djvu2kindle.py book.djvu` (the `./djvu2kindle.py` form
is for Linux/macOS).

### Python (3.8+)

```bash
pip install pikepdf Pillow
```

| Module | Used for |
|---|---|
| `pikepdf` | building the PDF page by page, bookmarks |
| `Pillow` | decoding rendered pages, grey/colour detection, JPEG encoding |

Both are also dependencies of `ocrmypdf`, so they are normally already
installed if `ocrmypdf` is. Everything else is from the standard library.

## Usage

```bash
./djvu2kindle.py book.djvu                  # writes book.pdf next to the input
./djvu2kindle.py book.djvu out.pdf          # explicit output name
./djvu2kindle.py book.djvu --pages 1-30 --no-ocr    # quick test run
./djvu2kindle.py book.djvu -l eng --dpi 150         # English book, smaller file
```

### Options

| Option | Default | Description |
|---|---|---|
| `input` | required | Input `.djvu` file (bundled multi-page documents; page order follows the DjVu). |
| `output` | `<input>.pdf` | Output PDF path. |
| `-l`, `--lang` | `rus+eng` | Tesseract language(s), `+`-separated. Each needs its `tesseract-ocr-*` package. |
| `--dpi` | `200` | Resolution of the illustration (background) layer. The DjVu stores it at about 200 dpi, so higher values only upsample. Lower it to shrink the file (`150` saves roughly 4–5 MB on the test book). Never upscales: capped at the page's own dpi. |
| `--quality` | `55` | JPEG quality (1–95) for the illustration layer. |
| `--pages A-B` | all | Convert only pages A to B (1-based, inclusive). Bookmarks are shifted to match; ones pointing outside the range go to the first page. |
| `-j`, `--jobs` | CPU count | Parallel workers, both for rendering pages and for `ocrmypdf`. |
| `--no-ocr` | off | Skip OCR. Output is the assembled PDF only (fast, no text layer). |

The text layer is always lossless JBIG2 at the DjVu's native resolution
(normally 600 dpi); there is no option to downsample it because it costs little
and is what keeps the text sharp.

### Size / quality guide (test book, with OCR)

| Illustration layer | Approx. size |
|---|---|
| `--dpi 150 --quality 55` | ~27 MB |
| `--dpi 200 --quality 55` (default) | ~32 MB |
| `--dpi 200 --quality 75` | ~36 MB |
| `--dpi 300 --quality 55` | ~38 MB |

About 19 MB of the default result is the text pages, so illustration settings
only move the remainder.

## How it works

Everything is in one file, [djvu2kindle.py](djvu2kindle.py). The flow is:

```
djvudump ─► scan_pages ─► [ProcessPool] render_page ×N ─► build_pdf ─► ocrmypdf ─► add_outline
                          (ddjvu, jbig2, djvuextract,       (pikepdf)               (pikepdf)
                           djvumake, Pillow)
```

### Page model

`scan_pages` returns one dict per page:

| Key | Meaning |
|---|---|
| `n` | 1-based page number |
| `w`, `h`, `dpi` | pixel size and resolution from the page's `INFO` chunk |
| `mask` | page has a text mask chunk (`Sjbz`/`Smmr`) |
| `bg` | page has a background chunk (`BG44`/`BGjp`) |
| `fg` | `(w, h)` of the foreground colour chunk (`FG44`), if any |

The combination decides how the page is built (see `build_pdf`):

| mask | bg | fg | Result |
|---|---|---|---|
| yes | no | no | Plain text page: the JBIG2 image is the whole page (most pages) |
| yes | yes | yes | Illustration page: background JPEG, plus foreground-colour JPEG shown through the mask |
| yes | yes | no | Background JPEG, mask painted black on top |
| no | yes | no | Full-page colour JPEG (e.g. the cover) |

### Code map

| Function | Lines | What it does |
|---|---|---|
| `scan_pages(djvu)` | [32](djvu2kindle.py:32) | Runs `djvudump` once and parses its chunk listing into the page dicts above. Parsing is regex-based on the dump text: a `FORM:DJVU … [Pn]` line starts a page, any other `FORM:` line (e.g. the shared `DJVI` dictionary) ends it, and chunk lines in between set the flags. |
| `run(cmd)` | [61](djvu2kindle.py:61) | `subprocess.run` wrapper that returns stdout bytes and raises on failure. |
| `render_page(args)` | [65](djvu2kindle.py:65) | **Worker, runs in a process pool.** Produces the raw layers of one page (see below). Takes and returns plain picklable data. |
| `bitonal_image(...)` | [112](djvu2kindle.py:112) | Creates a 1-bit JBIG2 image XObject, either as a normal image (`DeviceGray`) or as a stencil (`ImageMask`). |
| `jpeg_image(pdf, r)` | [124](djvu2kindle.py:124) | Creates a JPEG image XObject for the background layer. |
| `build_pdf(pages, results, out)` | [133](djvu2kindle.py:133) | Assembles all pages with `pikepdf`, choosing the layout from the table above. |
| `parse_sexp(text)` | [172](djvu2kindle.py:172) | Minimal s-expression parser for `djvused print-outline` output, including its `\ooo` octal-escaped UTF-8 strings. |
| `add_outline(...)` | [212](djvu2kindle.py:212) | Converts the parsed outline into PDF bookmarks (recursive for nested entries). |
| `main()` | [236](djvu2kindle.py:236) | Argument parsing, tool check, orchestration, temp directory handling. |

### `render_page` in detail

Each page uses up to three independent steps:

1. **Mask** (`p["mask"]`): `ddjvu -format=pbm -mode=black -page=N` renders only
   the 1-bit text layer at full resolution. `jbig2 -p -d` encodes it:
   `-p` emits PDF-ready data (no file header), `-d` enables duplicate-line
   removal. This is generic-region (lossless) coding, not symbol mode, so
   there is no risk of similar glyphs being merged.
2. **Background** (`p["bg"]`): `ddjvu -format=ppm -mode=background -size=WxH`
   renders the illustration layer scaled to `--dpi`. (`-mode=color` is used
   instead when the page has no mask, so the cover renders as a whole.) Pillow
   checks whether R, G and B are identical; if so the image is saved as
   greyscale, otherwise RGB, then JPEG-encoded with `--quality`.
   `-size` is used rather than `-scale` because it is unambiguous.
3. **Foreground colours** (`p["mask"]` and `p["fg"]`): the text mask is not
   simply black. DjVu colours masked pixels from a separate low-resolution
   `FG44` layer, and on illustration pages the mask contains dark parts of the
   picture whose colour comes from there. `ddjvu` cannot render that layer on
   its own, so the chunk is pulled out with `djvuextract` (`FG44=file`),
   wrapped into a minimal one-layer DjVu with `djvumake` (as a `BG44`
   chunk), and decoded with `ddjvu` to a small image (typically ~265×425),
   then saved as JPEG. Ignoring this layer paints those areas solid black.

### How the PDF page is composed (`build_pdf`)

Page size in points is `pixels / dpi × 72`. Each page gets a content stream
that draws full-page images with a `cm` matrix:

- **Plain text page:** one `/Mk` image (1-bit `DeviceGray`, JBIG2). In PDF's
  JBIG2Decode output 0 is black, which matches JBIG2's black = 1 after the
  filter's inversion, so no `/Decode` array is needed.
- **Illustration page:** `/Bg` (background JPEG) is drawn first. Then `/Fg`
  (the foreground-colour JPEG, `/Interpolate true`) is drawn with the JBIG2
  stencil attached as its explicit `/Mask`, so colour shows only where the
  mask has ink. The mask and the colour image have different resolutions;
  PDF allows this for explicit masks.
- **Fallback (no `FG44`):** the stencil is drawn as an `ImageMask` with fill
  colour `0 g` (black) over the background.

Image streams are written already encoded with
`stream.write(data, filter=Name.JBIG2Decode)` (or `DCTDecode`), so pikepdf
stores the bytes as-is and never re-encodes them.

### OCR step

`ocrmypdf` runs on the assembled PDF with:

```
-l <lang> -j <jobs> --optimize 0 --output-type pdf
```

- `--optimize 0` stops it from re-compressing the images that were built
  carefully above.
- `--output-type pdf` avoids the default PDF/A conversion (done through
  Ghostscript), which would re-encode images and enlarge the file.

OCR adds an invisible text layer on top; the page images are untouched. The
`lots of diacritics - possibly poor OCR` messages for Cyrillic pages are
harmless Tesseract warnings.

### Bookmarks

`djvused -e print-outline` prints the outline as an s-expression such as
`(bookmarks ("title" "#5" ("child" "#7")))`. `parse_sexp` turns it into nested
lists, and `add_outline` creates `pikepdf.OutlineItem`s with 0-based
destinations. It runs after OCR on the finished file so nothing can drop the
outline. With `--pages A-B`, destinations are shifted by `A-1`; targets outside
the range are sent to the first page. Non-numeric DjVu destinations (named
anchors) also go to the first page.

### Parallelism

`main()` renders pages with `ProcessPoolExecutor(--jobs)`. Each worker shells
out to `ddjvu`/`jbig2`/etc. and returns bytes; the main process collects all
results in memory (tens of MB for a book this size) and builds the PDF in a
single pass. Temporary `.pbm`/`.ppm`/`.fg44` files live in a
`tempfile.TemporaryDirectory`, deleted on exit.

## Limitations and ideas

- Only `FG44` foreground layers are supported. Pages with a JPEG foreground
  (`FGjp`) fall back to a black mask.
- Only the first `FG44` chunk of a page is read; DjVu normally stores one.
- Background is re-encoded as JPEG (the DjVu wavelet data cannot be embedded
  in PDF). JPEG 2000 would be closer but needs extra tooling.
- Lossy JBIG2 symbol mode (`jbig2 -s`) would save roughly 10 MB on the test
  book, at a small risk of merging similar-looking characters (e.g. 6/8, 3/8).
  Swap the `jbig2 -p -d` call in `render_page` and add shared-symbol handling
  in `build_pdf` to try it.
- Pages are not cropped, so margins are preserved; auto-trimming them would make
  text larger on a Kindle Scribe.
- OCR quality depends on Tesseract; for better Cyrillic accuracy try a larger
  language model or other OCR engine plugin for `ocrmypdf`.
