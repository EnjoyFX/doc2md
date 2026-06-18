# doc2md

A thin wrapper around [Microsoft MarkItDown](https://github.com/microsoft/markitdown)
that converts Office documents (`.docx`, `.xlsx`, `.pptx`) to Markdown **and**
extracts the embedded images into a folder of your choice, with predictable,
sequentially numbered filenames.

Unlike vanilla `markitdown`, which by default strips images entirely (or, with
`--keep-data-uris`, inlines them as base64 blobs that make the Markdown file
unreadable), `doc2md`:

- writes every image as a real file on disk (`<doc>_01.png`, `<doc>_02.jpg`, ...),
- rewrites the Markdown so each `![](...)` link points at the on-disk file with a
  relative path that stays correct regardless of where you put the output,
- falls back to extracting images straight from the Office ZIP when MarkItDown
  does not emit references for a given format (typical for `.xlsx`).

## Requirements

- Python 3.10 or newer (uses only the standard library; no third-party deps for
  the wrapper itself)
- [`markitdown`](https://github.com/microsoft/markitdown) on your `PATH`.
  Recommended install:
  ```sh
  uv tool install --python 3.12 'markitdown[docx,xlsx,pptx]==0.1.6'
  ```

## Installation

Clone the repo and symlink the script into a directory on your `PATH`:

```sh
git clone <repo-url> ~/projects/doc2md
ln -s ~/projects/doc2md/doc2md ~/.local/bin/doc2md
```

Verify:

```sh
doc2md --help
```

## Usage

```
doc2md <file.docx|.xlsx|.pptx> [-i IMAGES_DIR] [-o OUTPUT_DIR]
```

### Options

| Flag | Default | Meaning |
| --- | --- | --- |
| `-o, --output-dir DIR` | `.` (current dir) | Where the resulting `.md` file is written. |
| `-i, --images-dir DIR` | `images` | Where extracted images are written. **Relative paths are resolved against `--output-dir`**, so `-i images` always lands next to the Markdown. Pass an absolute path to override. |
| `-h, --help` | – | Show usage. |

### Output naming

- Markdown file: `<docname>.md` (same base name as the input).
- Image files: `<docname>_NN.<ext>`, numbered from `01` in the order in which
  MarkItDown emits them (which matches document order for `.docx`).

MIME types from inline data URIs are normalised to common file extensions
(`jpeg → jpg`, `svg+xml → svg`, `x-emf → emf`, `x-wmf → wmf`, `tiff → tif`).

## Examples

### Minimum

```sh
doc2md report.docx
```

Produces:

```
./report.md
./images/report_01.png
./images/report_02.jpg
...
```

### Output and images side by side

```sh
doc2md report.docx -o ./out
```

Produces:

```
./out/report.md
./out/images/report_01.png
...
```

`-i` was not passed, so it defaulted to `images`, resolved relative to `./out/`.

### Custom images sub-folder

```sh
doc2md report.docx -o ./out -i resource/diagrams
```

Produces:

```
./out/report.md
./out/resource/diagrams/report_01.png
...
```

The links inside `report.md` will look like
`![](resource/diagrams/report_01.png)` — the script computes the relative path
from the Markdown file to each image.

### Absolute images path (override)

```sh
doc2md report.docx -o ./out -i /tmp/shared-img
```

Images go to `/tmp/shared-img/`; Markdown links use an absolute path.

## Behaviour by format

| Format | What MarkItDown emits | What `doc2md` does |
| --- | --- | --- |
| `.docx` | Inline base64 data URIs (via `mammoth`). | Extracts every image to a file, **rewrites the Markdown links** to point at the file. |
| `.pptx` | Usually no inline image references. | Falls back to ZIP extraction; images are saved to disk but **not linked** in the Markdown. The script tells you when this happens. |
| `.xlsx` | No inline image references. | Same as `.pptx`: ZIP extraction, no inline links. |

The fallback exists because workbook and slide-deck images are not part of the
linear text flow that Markdown represents, so there is no unambiguous insertion
point for the link.

## How it works (internals)

1. Runs `markitdown <input> --keep-data-uris`, capturing the Markdown output.
2. Scans the output with a regular expression for `data:image/<mime>;base64,<...>`
   URIs.
3. For each match, in document order:
   - decodes the base64 payload,
   - writes it to `<images-dir>/<docname>_NN.<ext>`,
   - replaces the data URI in the Markdown with a relative path computed by
     `os.path.relpath(image_path, markdown_dir)`.
4. If step 2 found zero URIs (e.g. for `.xlsx`), opens the document as a ZIP
   archive and extracts everything under `word/media/`, `xl/media/`, or
   `ppt/media/` (whichever applies), naming the files with the same
   `<docname>_NN.<ext>` scheme.
5. Writes the rewritten Markdown to `<output-dir>/<docname>.md`.

## Limitations

- **Image filenames are positional, not semantic.** You get `report_01.png`,
  not `report_architecture_diagram.png`. Generating descriptive names would
  require running a vision model over each image, which is out of scope.
- **Duplicate images are written twice.** If the same picture appears multiple
  times in the document it gets multiple files. This keeps the link rewriting
  trivially correct; deduplication would require content hashing and is left as
  a possible enhancement.
- **`.xlsx` and `.pptx` images are not linked in the Markdown.** They are
  extracted to disk so nothing is lost, but you have to insert the references
  yourself if you want them.
- **Vector formats (`.emf`, `.wmf`) are saved as-is.** Most Markdown viewers and
  browsers cannot render them. Converting them to PNG/SVG is out of scope.
- **Old binary formats (`.doc`, `.xls`, `.ppt`) are not supported.** Convert
  them to the modern OOXML format first.

## License

MIT.
