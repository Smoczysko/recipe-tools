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

1. **Fetch the RAW HTML — do NOT use `WebFetch`.** `WebFetch` runs the page through a
   summarizing helper model that silently paraphrases and shortens the recipe steps; its
   output is NOT verbatim and must never be used for this skill. Instead download the raw
   HTML with `curl` via the Bash tool:

   ```bash
   curl -sL -A "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0 Safari/537.36" "<url>" -o /tmp/recipe.html
   ```

2. **Extract the recipe text deterministically.** The whole recipe lives in the element
   `itemprop="recipeInstructions" class="article-content-body"`. Step text is plain text
   and `<p>` paragraphs interleaved with `<img>` placeholders, `<br>`, and ad `<div>`s, so
   strip the markup to plain text. Run this with Bash and read the output:

   ```bash
   python3 - <<'PY'
   import re, html
   h = open('/tmp/recipe.html', encoding='utf-8').read()
   s = h.find('class="article-content-body"'); s = h.find('>', s) + 1
   e = len(h)
   for m in ['Polecane przepisy','class="article-tags','id="comments','class="comments','<footer','Zobacz podobne','class="related']:
       j = h.find(m, s)
       if j > 0: e = min(e, j)
   b = h[s:e]
   b = re.sub(r'<div[^>]*class="img-placeholder".*?</div>\s*</div>', ' ', b, flags=re.S)
   b = re.sub(r'<div[^>]*class="ads-slot-article".*?</div>\s*</div>', ' ', b, flags=re.S)
   b = re.sub(r'<(script|style|ins)\b.*?</\1>', ' ', b, flags=re.S)
   b = re.sub(r'<span class="nutrition-info".*?</span>\s*</p>', '</p>', b, flags=re.S)
   b = re.sub(r'<br\s*/?>', '\n', b)
   b = re.sub(r'</(p|div|h[1-6])>', '\n\n', b)
   b = re.sub(r'<[^>]+>', '', b)
   b = html.unescape(b).replace('\xa0', ' ').replace('#ads#', '')
   b = re.sub(r'[ \t]+', ' ', b)
   out, blank = [], False
   for ln in (l.strip() for l in b.split('\n')):
       if ln: out.append(ln); blank = False
       elif not blank: out.append(''); blank = True
   print('\n'.join(out).strip())
   PY
   ```

   The output contains, in order: marketing intro prose, the metadata line
   (`Czas przygotowania:`, `Czas pieczenia:`/`Czas smażenia:`/`Czas gotowania:`,
   `Liczba porcji:`), nutrition, the `Składniki` list, more intro prose, then the
   preparation steps (each step is a blank-line-separated block), then closing/serving
   notes. Also grab the recipe **title** from the `<h1>` (`grep -oE '<h1[^>]*>[^<]*' /tmp/recipe.html`).

3. **Assemble from the extracted text — copy step text VERBATIM.** Build the output from
   this plain text. The preparation steps are the sequential imperative instruction blocks
   (from the first "do this" paragraph through the final serving/storage paragraph). Copy
   each step **character-for-character** — see the verbatim rule below. Drop the marketing
   intro prose, nutrition block, the duplicated `Składniki` list inside the body (use it
   only to cross-check ingredients), ratings, related recipes, and newsletter text.

   Keep the recipe content in its original language (Polish) unless the user asks for a
   translation. Do not invent or "improve" quantities or steps.

   If `curl`/`python3` are unavailable or the page layout doesn't match, tell the user
   rather than falling back to `WebFetch` (which would silently summarize).

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
- **Steps must be VERBATIM — copy, never rewrite.** Reproduce each step's text exactly as
  it appears in the extracted HTML: same words, same sentences, same order, same numbers
  and units. Do NOT shorten, paraphrase, summarize, merge sentences, drop the author's
  asides, or "clean up" phrasing. The only edits allowed inside a step are removing a
  `Porada` callout and stripping a leading step *name/heading*. If you find yourself
  rewording a sentence, stop — paste the original instead. (This skill previously failed
  by summarizing steps; that is the one thing it must not do.)
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
