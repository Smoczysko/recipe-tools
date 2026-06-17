---
name: to-markdown
description: Parse a recipe from aniagotuje.pl into a clean, plain-text markdown file ready to paste into Google Drive (or Docs). Use when the user gives an aniagotuje.pl recipe URL and wants it saved or converted to markdown.
---

# Recipe to Markdown

Convert an [aniagotuje.pl](https://aniagotuje.pl) recipe page into clean markdown that
pastes cleanly into Google Drive / Google Docs, optimized for easy copying.

## When to use

The user provides an `aniagotuje.pl/przepis/...` URL (or asks to convert/save such a
recipe). If the URL is for a different site, tell the user this skill is tuned for
aniagotuje.pl — the parsing of section headings below is specific to it.

## Steps

1. **Fetch the page.** Use the `WebFetch` tool on the recipe URL. Ask it to return,
   verbatim and in full (NOT summarized):
   - the recipe title
   - every ingredient line, including the name/label of each ingredient group if the
     recipe splits ingredients into groups (e.g. "Ciasto", "Nadzienie")
   - the complete text of every preparation step
   - metadata: prep time, cook time, number of servings
   - flag any blocks labeled `Porada` so they can be dropped

2. **Map the Polish sections.** aniagotuje.pl uses these headings consistently:
   - `Składniki` → ingredients
   - `Sposób przygotowania` → preparation steps
   - Metadata labels: `Czas przygotowania:` (prep time), `Czas smażenia:` /
     `Czas pieczenia:` / `Czas gotowania:` (cook time), `Liczba porcji:` (servings)

   Keep the recipe content in its original language (Polish) unless the user asks for
   a translation. Do not invent or "improve" quantities or steps — transcribe what
   the page says.

## Content rules

These are the important transformations — follow them exactly:

- **Ingredients: combine duplicates across groups.** If the recipe lists ingredients in
  multiple groups and the same ingredient appears in more than one, merge them into a
  single line with the quantities added together (e.g. group A "200 g mąki" + group B
  "100 g mąki" → "300 g mąki"). Do not keep the group headings; produce one flat list.
- **Split comma-separated spice/seasoning lists into one ingredient each.** When a single
  line packs several seasonings together — e.g.
  `przyprawy: płaska łyżeczka soli, po 1/4 łyżeczki gałki muszkatołowej oraz oregano, szczypta kurkumy`
  — break it into separate ingredient lines, dropping the `przyprawy:` (or similar) label:
  ```
  płaska łyżeczka soli
  1/4 łyżeczki gałki muszkatołowej
  1/4 łyżeczki oregano
  szczypta kurkumy
  ```
  Distribute a shared quantity introduced by `po` across each item it covers (`po 1/4
  łyżeczki X oraz Y` → `1/4 łyżeczki X` and `1/4 łyżeczki Y`). Split on commas and on
  `oraz`/`i` that join distinct seasonings. Do not invent quantities — if an item has none,
  list it without one.
- **Sum the times into a single "Czas".** Add `Czas przygotowania` and the cook time
  (`Czas pieczenia` / `Czas smażenia` / `Czas gotowania`) together and present the total
  under one `## Czas` header (e.g. 20 min prep + 45 min baking → `1 godz 5 min`). If only
  one of the two times is given, use that single value. Use natural Polish units
  (`min`, `godz`).
- **Prefer weights.** When an ingredient gives a volume/loose measure (szklanka/cup,
  łyżka, etc.) and a weight is also available or derivable, use the weight in grams or
  kilograms. If only a loose measure is given, keep it as-is — do not guess a weight.
- **Steps as raw text — no bullets, no numbers.** Each preparation step is a plain
  paragraph of raw text with no leading `-`, `1.`, or other marker, so it can be copied
  cleanly. Separate consecutive steps with a blank line.
- **Ingredients as raw text — no bullets.** Each ingredient is on its own line with no
  leading `-` or other marker. Use a single newline between ingredients (no blank lines
  between them).
- **Metadata as headers, not bold labels.** Render each metadata field (source, times,
  servings) as its own `##` header with the value on the line below — do not use bold
  inline labels like `**Źródło:**`.
- **Full step content, no summaries.** Transcribe the entire text of each step. Do not
  shorten, paraphrase, or summarize. Omit any step *name/heading* — keep only the step's
  body text.
- **Drop `Porada` blocks.** Exclude anything labeled `Porada` (tip callouts), whether it
  is a standalone section or embedded inside a step.

## Output template

```markdown
# <Recipe title>

## Źródło

<url>

## Czas

<prep time + cook time, summed>

## Porcje

<servings>

## Składniki

<ingredient 1>
<ingredient 2>
<ingredient 3>

## Sposób przygotowania

<full text of step 1, raw paragraph>

<full text of step 2, raw paragraph>

<full text of step 3, raw paragraph>
```

Both ingredients and steps are raw, marker-free lines/paragraphs separated by blank
lines. The `## Czas` value is the prep time and cook time summed into one total.

## Write the file

Save to `<slug>.md` in the current directory (slug = the last path segment of the URL),
unless the user specified a path. Omit any metadata field the page doesn't provide rather
than leaving it blank. After writing, tell the user the file path and that it is ready to
paste into Google Drive.
