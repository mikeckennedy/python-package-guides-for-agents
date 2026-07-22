# content-types: Complete API Reference

> A zero-dependency Python library that maps file extensions to MIME content types — and back. Supports 364 extensions with a simple, focused API: filename → type via `get_content_type()`, and type → extension(s) via `guess_extension()` / `guess_all_extensions()`. Designed as a more complete, more accurate replacement for Python's built-in `mimetypes` module. Lookup only — it never opens, reads, or sniffs file bytes.

- **Version**: 0.5.7
- **Package**: `content-types` (PyPI)
- **Import**: `import content_types`
- **Python**: >= 3.10
- **Dependencies**: none (pure Python, zero runtime dependencies)
- **License**: MIT
- **Author**: Michael Kennedy
- **Repository**: https://github.com/mikeckennedy/content-types
- **Documentation**: https://mkennedy.codes/docs/content-types/

---

## Agent & documentation resources

The project publishes machine-readable docs at [mkennedy.codes/docs/content-types](https://mkennedy.codes/docs/content-types/). If you can fetch URLs, these are authoritative and stay in sync with the released package:

- **Full docs site:** https://mkennedy.codes/docs/content-types/
- **llms.txt** (indexed API map): https://mkennedy.codes/docs/content-types/llms.txt
- **llms-full.txt** (full text for LLM ingestion): https://mkennedy.codes/docs/content-types/llms-full.txt
- **Agent skill (Markdown):** https://mkennedy.codes/docs/content-types/SKILL.md — also at `/.well-known/agent-skills/content-types/SKILL.md`
- **Skills page (install commands):** https://mkennedy.codes/docs/content-types/skills.html

Every documentation page also has a plain-Markdown twin — swap `.html` for `.md` to get token-efficient source without site chrome. For example https://mkennedy.codes/docs/content-types/reference/get_content_type.html → https://mkennedy.codes/docs/content-types/reference/get_content_type.md

---

## Installation

```bash
# With pip
pip install content-types

# With uv
uv add content-types

# CLI tool (standalone)
uv tool install content-types
# or
pipx install content-types
```

---

## Quick Start

```python
import content_types

# From a filename
content_types.get_content_type("photo.jpg")        # "image/jpeg"

# From an extension (with or without dot)
content_types.get_content_type(".png")              # "image/png"
content_types.get_content_type("png")               # "image/png"

# From a Path object
from pathlib import Path
content_types.get_content_type(Path("doc.pdf"))     # "application/pdf"

# Reverse lookup: MIME type -> extension(s)
content_types.guess_extension("application/pdf")     # ".pdf"
content_types.guess_all_extensions("image/jpeg")     # [".jpg", ".jpeg", ".jpe"]

# Use convenience constants
content_types.jpg    # "image/jpeg"
content_types.json   # "application/json"
content_types.pdf    # "application/pdf"
```

---

## API Reference

### `get_content_type(filename_or_extension, treat_as_binary=True, fallback=_UNSET)`

The primary forward-lookup function. Returns the MIME type string for a given filename or extension.

**Signature:**

```python
def get_content_type(
    filename_or_extension: str | Path,
    treat_as_binary: bool = True,
    fallback: Optional[str] | _Unset = _UNSET,
) -> Optional[str]
```

**Parameters:**

| Parameter | Type | Default | Description |
|---|---|---|---|
| `filename_or_extension` | `str \| Path` | *(required)* | A filename (`"photo.jpg"`), bare extension (`"jpg"`), dotted extension (`".jpg"`), full path (`"images/photo.jpg"`), or `pathlib.Path` object. URLs with query strings or fragments are also accepted. |
| `treat_as_binary` | `bool` | `True` | Controls the default fallback for unknown extensions when `fallback` is omitted. `True` returns `"application/octet-stream"`, `False` returns `"text/plain"`. |
| `fallback` | `Optional[str]` | *(unset sentinel)* | Explicit value returned for unknown extensions; when provided it takes precedence over `treat_as_binary`. Pass a MIME type string to substitute your own placeholder, or `None` to get `None` back for unknowns (so you can branch on a miss). When omitted entirely, the `treat_as_binary` behavior applies — the default is a private `_UNSET` sentinel, not `None`, precisely so an explicit `fallback=None` is distinguishable from "not passed". |

**Returns:** `Optional[str]` — the MIME type for a known extension; otherwise the explicit `fallback` (which may be `None`) or the `treat_as_binary` default.

**Raises:** `TypeError` if `filename_or_extension` is `None`.

**Behavior Details:**

| Input | Result | Notes |
|---|---|---|
| `"photo.jpg"` | `"image/jpeg"` | Standard filename |
| `".webp"` | `"image/webp"` | Dotted extension |
| `"webp"` | `"image/webp"` | Bare extension (dot added automatically) |
| `Path("doc.pdf")` | `"application/pdf"` | Path objects — uses `.suffix` |
| `"file.mp3?cache_id=123"` | `"audio/mpeg"` | Query strings stripped |
| `"file.pdf#page=5"` | `"application/pdf"` | URL fragments stripped |
| `"archive.tar.gz"` | `"application/gzip"` | Compound extensions — uses last segment |
| `"FILE.JPG"` | `"image/jpeg"` | Case-insensitive |
| `".gitignore"` | `"application/octet-stream"` | Dotfiles with no extension — unknown |
| `"Makefile"` | `"application/octet-stream"` | No extension — unknown |
| `""` | `"application/octet-stream"` | Empty string — unknown |
| `"unknown.xyz"` | `"application/octet-stream"` | Unknown, binary mode (default) |
| `"unknown.xyz"` (binary=False) | `"text/plain"` | Unknown, text mode |
| `"unknown.xyz"` (fallback="application/x-custom") | `"application/x-custom"` | Unknown, explicit fallback |
| `"unknown.xyz"` (fallback=None) | `None` | Unknown, explicit None — branch on a miss |
| `None` | raises `TypeError` | None not allowed |

**Examples:**

```python
from content_types import get_content_type

# Web server content-type headers
get_content_type("index.html")     # "text/html"
get_content_type("styles.css")     # "text/css"
get_content_type("app.js")         # "text/javascript"
get_content_type("logo.svg")       # "image/svg+xml"
get_content_type("data.json")      # "application/json"

# S3 / cloud storage metadata
get_content_type("report.pdf")     # "application/pdf"
get_content_type("data.parquet")   # "application/vnd.apache.parquet"
get_content_type("db.sqlite")      # "application/vnd.sqlite3"

# Data science workflows
get_content_type("notebook.ipynb") # "application/x-ipynb+json"
get_content_type("model.pkl")      # "application/octet-stream"
get_content_type("config.yaml")    # "application/yaml"
get_content_type("config.toml")    # "application/toml"

# Unknown extension with text fallback
get_content_type("notes.xyz", treat_as_binary=False)  # "text/plain"

# Unknown extension with explicit fallback (overrides treat_as_binary)
get_content_type("notes.xyz", fallback="application/x-custom")  # "application/x-custom"

# fallback=None: get None back for unknowns so you can branch on a miss
if (ct := get_content_type("notes.xyz", fallback=None)) is None:
    ct = sniff_bytes_some_other_way()
```

---

### `guess_extension(content_type, with_dot=True)`

Reverse lookup: returns the canonical file extension for a MIME / content type. The inverse of `get_content_type()`. Added in v0.5.7.

**Signature:**

```python
def guess_extension(content_type: str, with_dot: bool = True) -> Optional[str]
```

**Parameters:**

| Parameter | Type | Default | Description |
|---|---|---|---|
| `content_type` | `str` | *(required)* | A MIME type to look up, optionally with parameters (e.g. `"application/pdf"` or `"text/html; charset=utf-8"` — parameters are stripped). Case-insensitive. |
| `with_dot` | `bool` | `True` | When `True` (default) the returned extension has a leading dot (`".pdf"`), ready to concatenate onto a filename. When `False`, the bare extension (`"pdf"`). |

**Returns:** `Optional[str]` — the canonical extension, or `None` if the content type is unknown (including empty, whitespace-only, or parameter-only input).

**Raises:** `TypeError` if `content_type` is `None`.

**Behavior Details:**

- **Parameter stripping:** an HTTP `Content-Type` header value like `"text/html; charset=utf-8"` resolves the same as `"text/html"`.
- **Alias normalization:** well-known non-canonical / legacy spellings resolve to the canonical type — `"text/json"` → `.json`, `"image/jpg"` → `.jpg`, `"text/xml"` → `.xml`, `"application/javascript"` → `.js`, `"application/x-zip-compressed"` → `.zip`, `"audio/mp3"` → `.mp3`, `"image/x-icon"` → `.ico`, `"text/yaml"` → `.yaml`, and more (see `_CONTENT_TYPE_ALIASES` below).
- **Canonical choice:** when a type maps to several extensions, the canonical one wins — `"image/jpeg"` → `.jpg` (not `.jpeg`), `"text/html"` → `.html` (not `.htm`), `"image/tiff"` → `.tiff`, `"video/mpeg"` → `.mpeg`, `"application/x-msdownload"` → `.exe`. This follows this library's table and may differ from the standard library `mimetypes` module.

**Examples:**

```python
from content_types import guess_extension

guess_extension("application/pdf")               # ".pdf"
guess_extension("image/jpeg")                    # ".jpg"
guess_extension("text/html; charset=utf-8")      # ".html"
guess_extension("text/json")                     # ".json"  (alias -> application/json)
guess_extension("application/pdf", with_dot=False)  # "pdf"
guess_extension("application/x-does-not-exist")  # None
```

---

### `guess_all_extensions(content_type, with_dot=True)`

Like `guess_extension()`, but returns **every** known extension for the type, canonical first. Added in v0.5.7.

**Signature:**

```python
def guess_all_extensions(content_type: str, with_dot: bool = True) -> list[str]
```

**Parameters:**

| Parameter | Type | Default | Description |
|---|---|---|---|
| `content_type` | `str` | *(required)* | A MIME type to look up, optionally with parameters. Same case-insensitivity, parameter-stripping, and alias handling as `guess_extension()`. |
| `with_dot` | `bool` | `True` | When `True` (default) each extension has a leading dot; when `False`, bare extensions. |

**Returns:** `list[str]` — all extensions for the type, canonical first (so `result[0] == guess_extension(content_type)`), or `[]` if the type is unknown. The order follows this library's table and is not guaranteed to match `mimetypes`.

**Raises:** `TypeError` if `content_type` is `None`.

**Examples:**

```python
from content_types import guess_all_extensions

guess_all_extensions("image/jpeg")                    # [".jpg", ".jpeg", ".jpe"]
guess_all_extensions("text/html")                     # [".html", ".htm"]
guess_all_extensions("application/pdf")               # [".pdf"]
guess_all_extensions("image/jpeg", with_dot=False)    # ["jpg", "jpeg", "jpe"]
guess_all_extensions("application/x-does-not-exist")  # []
```

---

### `cli()`

Command-line entry point. Registered as the `content-types` console script.

**Signature:**

```python
def cli() -> None
```

**CLI Usage:**

```bash
$ content-types photo.jpg
image/jpeg

$ content-types .webp
image/webp

$ content-types unknown.xyz
application/octet-stream
```

Exits with code 1 and prints usage if no argument is provided.

---

### Convenience Constants

Pre-computed module-level variables for the most commonly used MIME types. These are evaluated at import time via `get_content_type()`.

```python
import content_types
```

**General:**

| Constant | Value | Type |
|---|---|---|
| `content_types.webp` | `"image/webp"` | `str` |
| `content_types.png` | `"image/png"` | `str` |
| `content_types.jpg` | `"image/jpeg"` | `str` |
| `content_types.mp3` | `"audio/mpeg"` | `str` |
| `content_types.json` | `"application/json"` | `str` |
| `content_types.pdf` | `"application/pdf"` | `str` |
| `content_types.zip` | `"application/zip"` | `str` |
| `content_types.xml` | `"application/xml"` | `str` |
| `content_types.csv` | `"text/csv"` | `str` |
| `content_types.md` | `"text/markdown"` | `str` |

**Data Science:**

| Constant | Value | Type |
|---|---|---|
| `content_types.parquet` | `"application/vnd.apache.parquet"` | `str` |
| `content_types.ipynb` | `"application/x-ipynb+json"` | `str` |
| `content_types.pkl` | `"application/octet-stream"` | `str` |
| `content_types.yaml` | `"application/yaml"` | `str` |
| `content_types.toml` | `"application/toml"` | `str` |
| `content_types.sqlite` | `"application/vnd.sqlite3"` | `str` |

**Usage:**

```python
import content_types

# Use as Content-Type headers
headers = {"Content-Type": content_types.json}

# Compare types
if detected_type == content_types.pdf:
    handle_pdf(file)
```

---

### `EXTENSION_TO_CONTENT_TYPE`

The core dictionary mapping all known extensions to MIME types. Extensions are stored **without** leading dots, all lowercase.

**Signature:**

```python
EXTENSION_TO_CONTENT_TYPE: dict[str, str]
```

**Size:** 364 entries (as of v0.5.7)

**Direct access:**

```python
from content_types import EXTENSION_TO_CONTENT_TYPE

EXTENSION_TO_CONTENT_TYPE['jpg']      # "image/jpeg"
EXTENSION_TO_CONTENT_TYPE['parquet']  # "application/vnd.apache.parquet"
EXTENSION_TO_CONTENT_TYPE.get('xyz')  # None (not found)

# Iterate all known extensions
for ext, mime in EXTENSION_TO_CONTENT_TYPE.items():
    print(f".{ext} -> {mime}")
```

> **Note:** Prefer `get_content_type()` for lookups — it handles filenames, Path objects, case normalization, query strings, and fallback behavior. Use the dictionary directly only when you need raw access to the mapping (e.g., iteration, bulk operations, checking if an extension is known).

---

### `__version__`

The package version, dynamically pulled from distribution metadata at import time.

```python
import content_types
print(content_types.__version__)  # e.g., "0.5.7"
```

> **Note:** `__version__` is read from *installed* distribution metadata via `importlib.metadata.version('content-types')` at import time, so importing the package requires it to be installed (a bare `import content_types` against an uninstalled source checkout raises `PackageNotFoundError`).

---

## Internal API

Not part of the public surface — useful for understanding behavior, but subject to change. Do not import these in application code.

- **`_reverse_map() -> dict[str, list[str]]`** — builds the content-type → extensions reverse of `EXTENSION_TO_CONTENT_TYPE`, extensions listed canonical-first. Decorated with `@functools.cache`, so it is constructed lazily on the first reverse lookup and cached for the process lifetime; callers who only use `get_content_type()` never pay for it.
- **`_CANONICAL_OVERRIDES: dict[str, str]`** — promotes the preferred extension to the front of its reverse-map list where the forward dict's first-listed extension is the less-common alias: `text/html` → `html` (not `htm`), `image/tiff` → `tiff`, `video/mpeg` → `mpeg`, `application/x-msdownload` → `exe`.
- **`_CONTENT_TYPE_ALIASES: dict[str, str]`** — maps well-known non-canonical / legacy MIME spellings to the canonical type, applied on reverse lookup only (the forward table stays pure). Includes `text/json`, `text/xml`, `application/javascript`, `application/x-javascript`, `image/jpg`, `image/pjpeg`, `image/x-png`, `image/x-icon`, `image/svg`, `application/x-yaml`, `text/yaml`, `text/x-yaml`, `text/toml`, `audio/mp3`, `application/x-zip-compressed`, `application/x-zip`, `application/x-gzip`, `application/x-pdf`, `application/csv`, `text/x-csv`, and `text/x-markdown`.
- **`_normalize_content_type(content_type: str) -> str`** — trims whitespace, strips `;`-delimited MIME parameters, lowercases, then applies `_CONTENT_TYPE_ALIASES`.
- **`_Unset` / `_UNSET`** — the private sentinel class/instance used as `get_content_type()`'s `fallback` default so an explicit `fallback=None` is distinguishable from an omitted argument.

---

## Complete Extension Map

All 364 supported extensions organized by category.

### Text & Markup

| Extension | MIME Type |
|---|---|
| `txt` | `text/plain` |
| `htm`, `html` | `text/html` |
| `css` | `text/css` |
| `csv` | `text/csv` |
| `tsv` | `text/tab-separated-values` |
| `md`, `markdown` | `text/markdown` |
| `rst` | `text/x-rst` |
| `rtx` | `text/richtext` |
| `rtf` | `text/rtf` |
| `n3` | `text/n3` |
| `sgm`, `sgml` | `text/x-sgml` |
| `etx` | `text/x-setext` |
| `vcf` | `text/x-vcard` |
| `vtt` | `text/vtt` |
| `srt` | `text/plain` |
| `adoc`, `asciidoc` | `text/asciidoc` |
| `org` | `text/org` |
| `bib` | `text/x-bibtex` |
| `wiki` | `text/plain` |

### JavaScript & JSON

| Extension | MIME Type |
|---|---|
| `js` | `text/javascript` |
| `mjs` | `text/javascript` |
| `json` | `application/json` |
| `map` | `application/json` |

### XML

| Extension | MIME Type |
|---|---|
| `xml` | `application/xml` |
| `xsl` | `application/xml` |
| `rdf` | `application/xml` |
| `wsdl` | `application/xml` |
| `xpdl` | `application/xml` |

### Images — Standard

| Extension | MIME Type |
|---|---|
| `jpg`, `jpeg`, `jpe` | `image/jpeg` |
| `png` | `image/png` |
| `gif` | `image/gif` |
| `bmp` | `image/bmp` |
| `webp` | `image/webp` |
| `avif` | `image/avif` |
| `ico` | `image/vnd.microsoft.icon` |
| `svg` | `image/svg+xml` |
| `tif`, `tiff` | `image/tiff` |
| `heic` | `image/heic` |
| `heif` | `image/heif` |
| `ief` | `image/ief` |

### Images — RAW Photography

| Extension | MIME Type | Camera |
|---|---|---|
| `cr2` | `image/x-canon-cr2` | Canon |
| `cr3` | `image/x-canon-cr3` | Canon |
| `nef` | `image/x-nikon-nef` | Nikon |
| `nrw` | `image/x-nikon-nrw` | Nikon |
| `arw` | `image/x-sony-arw` | Sony |
| `srf` | `image/x-sony-srf` | Sony |
| `sr2` | `image/x-sony-sr2` | Sony |
| `dng` | `image/x-adobe-dng` | Adobe DNG |
| `orf` | `image/x-olympus-orf` | Olympus |
| `rw2` | `image/x-panasonic-rw2` | Panasonic |
| `pef` | `image/x-pentax-pef` | Pentax |
| `raf` | `image/x-fuji-raf` | Fuji |
| `raw` | `image/x-raw` | Generic |

### Images — Legacy & Specialty

| Extension | MIME Type |
|---|---|
| `ras` | `image/x-cmu-raster` |
| `pnm` | `image/x-portable-anymap` |
| `pbm` | `image/x-portable-bitmap` |
| `pgm` | `image/x-portable-graymap` |
| `ppm` | `image/x-portable-pixmap` |
| `rgb` | `image/x-rgb` |
| `xbm` | `image/x-xbitmap` |
| `xpm` | `image/x-xpixmap` |
| `xwd` | `image/x-xwindowdump` |

### Audio

| Extension | MIME Type |
|---|---|
| `mp3`, `mp2` | `audio/mpeg` |
| `ogg` | `audio/ogg` |
| `wav` | `audio/wav` |
| `aac` | `audio/aac` |
| `flac` | `audio/flac` |
| `m4a` | `audio/mp4` |
| `weba` | `audio/webm` |
| `opus` | `audio/opus` |
| `aif`, `aifc`, `aiff` | `audio/x-aiff` |
| `au`, `snd` | `audio/basic` |
| `ra` | `audio/x-pn-realaudio` |
| `midi`, `mid` | `audio/midi` |
| `ape` | `audio/x-ape` |
| `wma` | `audio/x-ms-wma` |
| `alac` | `audio/x-alac` |
| `dsd` | `audio/dsd` |
| `dsf` | `audio/x-dsf` |
| `adts`, `loas` | `audio/aac` |

### Video

| Extension | MIME Type |
|---|---|
| `mp4`, `m4v` | `video/mp4` |
| `mov`, `qt` | `video/quicktime` |
| `avi` | `video/x-msvideo` |
| `wmv` | `video/x-ms-wmv` |
| `mpg`, `mpeg`, `m1v`, `mpa`, `mpe`, `vob` | `video/mpeg` |
| `ogv` | `video/ogg` |
| `webm` | `video/webm` |
| `mkv` | `video/x-matroska` |
| `flv` | `video/x-flv` |
| `m2ts`, `mts` | `video/mp2t` |
| `f4v` | `video/x-f4v` |
| `movie` | `video/x-sgi-movie` |
| `3gp`, `3gpp` | `audio/3gpp` |
| `3g2`, `3gpp2` | `audio/3gpp2` |

### Documents & Office

| Extension | MIME Type |
|---|---|
| `pdf` | `application/pdf` |
| `epub` | `application/epub+zip` |
| `doc`, `dot`, `wiz` | `application/msword` |
| `xls`, `xlb` | `application/vnd.ms-excel` |
| `ppt`, `pot`, `ppa`, `pps`, `pwz` | `application/vnd.ms-powerpoint` |
| `docx` | `application/vnd.openxmlformats-officedocument.wordprocessingml.document` |
| `xlsx` | `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` |
| `pptx` | `application/vnd.openxmlformats-officedocument.presentationml.presentation` |
| `odt` | `application/vnd.oasis.opendocument.text` |
| `ods` | `application/vnd.oasis.opendocument.spreadsheet` |
| `odp` | `application/vnd.oasis.opendocument.presentation` |
| `odg` | `application/vnd.oasis.opendocument.graphics` |

### Archives & Compression

| Extension | MIME Type |
|---|---|
| `zip` | `application/zip` |
| `gz`, `tgz` | `application/gzip` |
| `tar` | `application/x-tar` |
| `7z` | `application/x-7z-compressed` |
| `rar` | `application/vnd.rar` |
| `bz2`, `tbz`, `tbz2` | `application/x-bzip2` |
| `xz`, `txz` | `application/x-xz` |
| `lz` | `application/x-lzip` |
| `lzma` | `application/x-lzma` |
| `zst`, `zstd` | `application/zstd` |
| `br` | `application/x-br` |
| `bcpio` | `application/x-bcpio` |
| `cpio` | `application/x-cpio` |
| `shar` | `application/x-shar` |
| `sv4cpio` | `application/x-sv4cpio` |
| `sv4crc` | `application/x-sv4crc` |
| `ustar` | `application/x-ustar` |
| `gtar` | `application/x-gtar` |

### Disk Images & Installers

| Extension | MIME Type |
|---|---|
| `iso` | `application/x-iso9660-image` |
| `dmg` | `application/x-apple-diskimage` |
| `img` | `application/x-raw-disk-image` |
| `cab` | `application/vnd.ms-cab-compressed` |
| `msi` | `application/x-msi` |

### Executables & Binaries

| Extension | MIME Type |
|---|---|
| `bin` | `application/octet-stream` |
| `a` | `application/octet-stream` |
| `so` | `application/octet-stream` |
| `o` | `application/octet-stream` |
| `dll`, `exe` | `application/x-msdownload` |

### Fonts

| Extension | MIME Type |
|---|---|
| `otf` | `font/otf` |
| `ttf` | `font/ttf` |
| `woff` | `font/woff` |
| `woff2` | `font/woff2` |

### Programming Languages

| Extension | MIME Type |
|---|---|
| `py` | `text/x-python` |
| `rs` | `text/x-rust` |
| `go` | `text/x-go` |
| `swift` | `text/x-swift` |
| `kt`, `kts` | `text/x-kotlin` |
| `java` | `text/x-java-source` |
| `scala` | `text/x-scala` |
| `rb` | `text/x-ruby` |
| `ts` | `text/typescript` |
| `tsx` | `text/tsx` |
| `jsx` | `text/jsx` |
| `vue` | `text/x-vue` |
| `dart` | `text/x-dart` |
| `lua` | `text/x-lua` |
| `r` | `text/x-r` |
| `jl` | `text/x-julia` |
| `cs` | `text/x-csharp` |
| `cpp`, `cxx`, `cc` | `text/x-c++src` |
| `hpp`, `hxx`, `hh` | `text/x-c++hdr` |
| `f90`, `f95`, `f03` | `text/x-fortran` |
| `m` | `text/x-objcsrc` |
| `asm`, `s` | `text/x-asm` |
| `c`, `h`, `ksh`, `pl`, `bat` | `text/plain` |
| `sh` | `application/x-sh` |
| `php` | `application/x-httpd-php` |
| `pyc`, `pyo` | `application/x-python-code` |

### Data Science & Scientific

| Extension | MIME Type |
|---|---|
| `parquet`, `parq` | `application/vnd.apache.parquet` |
| `ipynb` | `application/x-ipynb+json` |
| `pkl`, `pickle` | `application/octet-stream` |
| `npy` | `application/octet-stream` |
| `npz` | `application/zip` |
| `arrow`, `feather` | `application/vnd.apache.arrow.file` |
| `hdf5` | `application/x-hdf5` |
| `h5` | `application/x-hdf5` |
| `hdf` | `application/x-hdf` |
| `yaml`, `yml` | `application/yaml` |
| `toml` | `application/toml` |
| `proto` | `text/plain` |
| `pb` | `application/octet-stream` |
| `avro` | `application/avro` |
| `rda`, `rdata`, `rds` | `application/octet-stream` |
| `dta` | `application/x-stata-dta` |
| `sas` | `text/x-sas` |
| `sas7bdat` | `application/x-sas-data` |
| `sql` | `application/sql` |
| `sav` | `application/x-spss-sav` |
| `mat` | `application/x-matlab-data` |
| `sqlite`, `sqlite3`, `db` | `application/vnd.sqlite3` |
| `fits`, `fit` | `application/fits` |
| `nii` | `application/x-nifti` |
| `dcm` | `application/dicom` |
| `pdb` | `chemical/x-pdb` |
| `cdf`, `nc` | `application/x-netcdf` |

### Configuration & Infrastructure

| Extension | MIME Type |
|---|---|
| `ini`, `conf`, `cfg`, `config` | `text/plain` |
| `properties` | `text/plain` |
| `env` | `text/plain` |
| `editorconfig` | `text/plain` |
| `gitignore`, `gitattributes` | `text/plain` |
| `dockerignore` | `text/plain` |
| `npmrc`, `yarnrc` | `text/plain` |
| `babelrc`, `eslintrc`, `prettierrc` | `application/json` |
| `dockerfile` | `text/plain` |
| `tf`, `tfvars` | `text/plain` |
| `nomad`, `hcl` | `text/plain` |
| `kubeconfig` | `application/yaml` |
| `gradle` | `text/plain` |

### 3D Models & CAD

| Extension | MIME Type |
|---|---|
| `gltf` | `model/gltf+json` |
| `glb` | `model/gltf-binary` |
| `stl` | `model/stl` |
| `obj` | `model/obj` |
| `fbx` | `application/octet-stream` |
| `dwg` | `application/acad` |
| `dxf` | `application/dxf` |
| `skp` | `application/vnd.sketchup.skp` |
| `blend` | `application/x-blender` |
| `step`, `stp` | `application/step` |
| `iges`, `igs` | `application/iges` |
| `3ds` | `application/x-3ds` |
| `max` | `application/x-3dsmax` |
| `c4d` | `application/x-cinema4d` |

### Adobe Creative Suite

| Extension | MIME Type |
|---|---|
| `psd`, `psb` | `image/vnd.adobe.photoshop` |
| `indd`, `idml` | `application/x-indesign` |
| `prproj` | `application/x-premiere` |
| `aep` | `application/x-aftereffects` |
| `xd` | `application/x-xd` |
| `ai`, `eps`, `ps` | `application/postscript` |

### Database

| Extension | MIME Type |
|---|---|
| `accdb`, `mdb` | `application/msaccess` |
| `odb` | `application/vnd.oasis.opendocument.database` |
| `frm`, `myd`, `myi`, `ibd` | `application/octet-stream` |

### Packages & Distribution

| Extension | MIME Type |
|---|---|
| `apk` | `application/vnd.android.package-archive` |
| `deb` | `application/x-debian-package` |
| `rpm` | `application/x-rpm` |
| `whl`, `egg` | `application/zip` |
| `nuspec` | `application/xml` |
| `gemspec`, `podspec` | `text/x-ruby` |

### Blockchain & Smart Contracts

| Extension | MIME Type |
|---|---|
| `sol` | `text/x-solidity` |
| `vy` | `text/x-vyper` |

### Game Development

| Extension | MIME Type |
|---|---|
| `unity` | `text/plain` |
| `unitypackage` | `application/gzip` |
| `uasset`, `pak`, `bsp` | `application/octet-stream` |

### Email & Messages

| Extension | MIME Type |
|---|---|
| `eml`, `mht`, `mhtml`, `nws` | `message/rfc822` |

### Playlists

| Extension | MIME Type |
|---|---|
| `m3u`, `m3u8` | `application/vnd.apple.mpegurl` |

### Subtitles & Captions

| Extension | MIME Type |
|---|---|
| `vtt` | `text/vtt` |
| `srt` | `text/plain` |
| `ssa`, `ass` | `text/x-ssa` |
| `sub` | `text/x-microdvd` |
| `idx` | `application/octet-stream` |

### Logs & System Files

| Extension | MIME Type |
|---|---|
| `log`, `out`, `pid`, `lock` | `text/plain` |
| `tmp`, `bak`, `backup`, `cache` | `application/octet-stream` |

### Miscellaneous Application Types

| Extension | MIME Type |
|---|---|
| `webmanifest` | `application/manifest+json` |
| `wasm` | `application/wasm` |
| `nq` | `application/n-quads` |
| `nt` | `application/n-triples` |
| `trig` | `application/trig` |
| `oda` | `application/oda` |
| `p7c` | `application/pkcs7-mime` |
| `p12`, `pfx` | `application/x-pkcs12` |
| `latex` | `application/x-latex` |
| `mif` | `application/x-mif` |
| `ram` | `application/x-pn-realaudio` |
| `swf` | `application/x-shockwave-flash` |
| `tcl` | `application/x-tcl` |
| `tex` | `application/x-tex` |
| `texi`, `texinfo` | `application/x-texinfo` |
| `dvi` | `application/x-dvi` |
| `csh` | `application/x-csh` |
| `src` | `application/x-wais-source` |
| `roff`, `t`, `tr` | `application/x-troff` |
| `man` | `application/x-troff-man` |
| `me` | `application/x-troff-me` |
| `ms` | `application/x-troff-ms` |

---

## Why content-types Over Python's `mimetypes`?

Python's built-in `mimetypes` module is missing many common types that `content-types` handles correctly:

| Extension | `content-types` | `mimetypes` |
|---|---|---|
| `.webp` | `image/webp` | Missing |
| `.avif` | `image/avif` | Missing |
| `.flac` | `audio/flac` | Missing |
| `.mkv` | `video/x-matroska` | Missing |
| `.heic` | `image/heic` | Missing |
| `.woff2` | `font/woff2` | Missing |
| `.parquet` | `application/vnd.apache.parquet` | Missing |
| `.ipynb` | `application/x-ipynb+json` | Missing |
| `.toml` | `application/toml` | Missing |
| `.yaml` | `application/yaml` | Missing |
| `.ts` | `text/typescript` | Returns `video/mp2t` |
| `.md` | `text/markdown` | Missing |
| `.wasm` | `application/wasm` | Missing |

Additionally, `content-types` requires no file I/O (unlike `python-magic`) and has zero runtime dependencies.

---

## Architecture

The entire library is a single module (`content_types/__init__.py`, ~750 lines):

1. **`EXTENSION_TO_CONTENT_TYPE` dict** — the core mapping of 364 extensions to MIME types, grouped by category
2. **Reverse-lookup machinery** — `_CANONICAL_OVERRIDES`, `_CONTENT_TYPE_ALIASES`, the `@functools.cache`-lazy `_reverse_map()`, and `_normalize_content_type()`
3. **`guess_extension()` / `guess_all_extensions()`** — the public reverse-lookup functions
4. **`get_content_type()` function** — forward lookup with normalization, `Path` support, URL query/fragment stripping, and the `_UNSET`-sentinel `fallback` parameter
5. **Convenience constants** — 16 pre-computed shortcuts evaluated at import time
6. **`cli()` function** — console script entry point (forward lookup only)

No classes besides the private `_Unset` sentinel. No inheritance. No runtime dependencies. Type hints throughout (`Optional[X]` for nullables, per project convention). `py.typed` marker included for mypy compatibility.
