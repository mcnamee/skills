# printouts\

Throwaway PDFs made to be printed on paper: the day planners and team task
sheets the `outlook` plugin writes. Nothing here is a source document, so
nothing here is converted or indexed.

| | |
|---|---|
| **Setting** | none of its own - it sits **beside** `EVA_DOCUMENTS_DIR` (its parent, with `printouts` appended). `OUTLOOK_DOCS_DIR` overrides it with a full path, or `off` to stop printing |
| **Default** | `H:\Eva\printouts` |
| **Access** | `outlook` writes here, only when asked to print. No plugin reads it |
| **Output** | none - print it and bin it |

## What lands here

- `Calendar - <date> <weekday>.pdf` - "today's printable calendar" or
  "tomorrow's planner". A4 landscape, printed single-sided at 100% and folded
  (or double-sided, flip on short edge, when a team view is configured).
- `Team Tasks - <date>.pdf` - "print my team's tasks". A4 landscape, printed
  double-sided, flip on short edge.

Re-printing the same day overwrites that day's sheet rather than piling up
copies.

## Why it is not in `documents\pdf`

`documents\pdf` is the source folder `pdf-to-md` converts into the knowledge
base. Printouts used to land there, and a bulk conversion filled the index with
last week's meetings. Keeping this folder beside `documents\`, not inside it,
means no converter ever sees it - even with `PDF2MD_RECURSIVE=1`.

## Tidying up

Nothing deletes old sheets for you. Clear the folder out whenever you like;
`outlook` recreates it at startup if it is missing.
