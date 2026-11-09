# documents\pdf\

The PDF folder: source PDFs for the `pdf-to-md` plugin to convert into the
knowledge base, and PDFs the `confluence` plugin downloads from a page's
attachments. The day planners and team task sheets `outlook` prints do NOT come
here - they go to [`..\..\printouts`](../../printouts), so they are never
converted.

| | |
|---|---|
| **Setting** | `EVA_DOCUMENTS_DIR` (the `pdf-to-md` and `confluence` plugins both append `\pdf`) |
| **Default** | `H:\Eva\documents\pdf` |
| **Access** | `pdf-to-md` reads only — conversion never alters a PDF. `confluence` writes downloaded attachments here, touches nothing else, and never replaces an existing file unless asked |
| **Output** | [`..\..\knowledge\pdf`](../../knowledge/pdf) |

Conversion preserves reading order, headings and tables, including borderless
tables inferred from column alignment.

## Sub-folders need switching on

By default only the top level of this folder is converted. Turn on the plugin's
recursive option (`PDF2MD_RECURSIVE=1`) and sub-folders are
included, with the same structure mirrored into the output folder.

## Scans convert to nothing

There is no OCR in this suite. A PDF that is an image of a page — a scan, a
photographed signature page — produces an empty or near-empty Markdown file, and
it does so quietly. If a converted file looks suspiciously thin, open the PDF
and check whether its text is selectable. Getting a scan into the knowledge base
means OCRing it elsewhere first.

## Printed day planners are not here

Before outlook 12.0.0, day planners (`Calendar - <date> <weekday>.pdf`) and
team task sheets (`Team Tasks - <date>.pdf`) were printed into this folder, and
a bulk `pdf-to-md` run converted them into the knowledge base. They now go to
[`H:\Eva\printouts`](../../printouts). Any old ones still here are safe to
delete.

## Where the PDFs stay

Converted PDFs stay here. The Markdown copy in `knowledge\pdf\` is what gets
indexed and quoted; this folder remains the source, so a conversion can be
redone after a change to chunking or after a bad run, without hunting for the
originals again.
