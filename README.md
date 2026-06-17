# recipe-tools

A [Claude Code](https://claude.com/claude-code) plugin marketplace for turning recipes
into clean, copy-ready markdown.

It currently contains one plugin, **`recipe-to-markdown`**, which parses a recipe from
[aniagotuje.pl](https://aniagotuje.pl) into a plain-text markdown file that pastes neatly
into Google Drive / Google Docs.

## What it does

Given an `aniagotuje.pl/przepis/...` URL, it produces a `<slug>.md` file with:

- The recipe **title** as the document heading.
- **Metadata** (source, prep time, bake/cook time, servings) — each as its own `##`
  header with the value below.
- **Ingredients** as raw text, one per line, no bullets. Ingredients that appear in more
  than one ingredient group are **combined** into a single line with quantities summed,
  and **weights (g/kg) are preferred** over loose measures like cups (`szklanka`).
- **Preparation steps** as raw-text paragraphs — no numbers, bullets, or step headings —
  separated by blank lines, with the **full text** of each step (never summarized).
- `Porada` (tips) and `Uwagi`/`UWAGA` (notes) blocks are **dropped**.

The recipe content is kept in its original Polish.

## Install

In Claude Code:

```
/plugin marketplace add Smoczysko/recipe-tools
/plugin install recipe-to-markdown@recipe-tools
```

The first command registers this marketplace; the second installs the plugin from it.

To update after new changes are pushed:

```
/plugin marketplace update recipe-tools
```

## Use

Three equivalent ways, once installed:

1. **Slash command (explicit):**

   ```
   /recipe-to-markdown https://aniagotuje.pl/przepis/ciasto-jogurtowe
   ```

2. **Natural language (auto-triggers the skill):**

   > Convert this recipe to markdown: https://aniagotuje.pl/przepis/ciasto-jogurtowe

3. **Skill, invoked by name:** `/recipe-to-markdown` then provide the URL.

The result is written to a `<slug>.md` file in the current directory (e.g.
`ciasto-jogurtowe.md`), ready to copy into Google Drive. See
[`examples/ciasto-jogurtowe.md`](examples/ciasto-jogurtowe.md) for sample output.

## Structure

```
recipe-tools/
├── .claude-plugin/
│   └── marketplace.json                     # marketplace catalog
└── plugins/
    └── recipe-to-markdown/
        ├── .claude-plugin/
        │   └── plugin.json                  # plugin manifest
        ├── commands/
        │   └── recipe-to-markdown.md        # /recipe-to-markdown <URL> command
        └── skills/
            └── recipe-to-markdown/
                └── SKILL.md                 # parsing rules
```

## Note

The parsing is tuned specifically to aniagotuje.pl's page structure (section headings
like `Składniki` and `Sposób przygotowania`). Other recipe sites are not supported.
