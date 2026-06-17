---
description: Convert an aniagotuje.pl recipe URL into a clean, copy-ready markdown file.
argument-hint: <aniagotuje.pl recipe URL>
---

Convert the recipe at the following URL into clean markdown using the
`recipe-to-markdown` skill: $ARGUMENTS

Follow the skill's rules exactly:
- Fetch the page and use the **original Polish** content, verbatim and in full (re-fetch
  demanding untranslated Polish if the first fetch returns English or a summary).
- Combine ingredients that appear in multiple ingredient groups into one flat list with
  quantities summed; prefer weights (g/kg) over loose measures.
- Render ingredients as raw text, one per line, no bullets, single newline between them.
- Render preparation steps as raw-text paragraphs with no headings, numbers, or bullets,
  separated by blank lines; include the full text of each step, never a summary.
- Drop any `Porada` and `Uwagi`/`UWAGA` blocks.
- Render metadata (źródło, czas przygotowania, czas pieczenia/gotowania, porcje) as `##`
  headers with the value on the line below.
- Save the result to `<slug>.md` (slug = last path segment of the URL) and report the path.

If no URL was provided in $ARGUMENTS, ask the user for the aniagotuje.pl recipe URL.
