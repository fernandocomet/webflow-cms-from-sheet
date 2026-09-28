# webflow-cms-from-sheet

A Claude skill that builds a complete **Webflow CMS** from a spreadsheet: collections, fields, **help text on every field**, field groups and items, through the Webflow MCP.

Define your CMS in a sheet (one tab per collection), ask Claude to build it, and get it in Webflow without creating fields one by one.

## What it does

1. **Validates the sheet before touching Webflow.** It checks types, references and slugs, and treats a missing help text as an error.
2. **Creates the collections in dependency order.** Referenced collections come first; circular references are handled in two passes.
3. **Creates every field with its help text**, including `Name` and `Slug` when the API allows it.
4. **Sets up field groups** so the editor is organized.
5. **Imports the items as drafts.** It does a one-item dry run first and waits for your approval before importing the rest. It never publishes.
6. **Verifies the result** and reports per collection.

If a collection already exists, it doesn't duplicate it: it only adds missing fields and shows you the differences.

## Sheet format

Use `template/webflow-cms-template.xlsx` as a starting point (it works in Excel and Google Sheets). It includes a blog example with **Categories**, **Authors** and **Posts**.

| Tab | Purpose |
|---|---|
| `_readme` | Instructions for whoever fills in the sheet. Ignored by the skill. |
| `_collections` | One row per collection: `tab`, `displayName`, `singularName`, `slug`. |
| Any other tab | One collection. |

Inside each collection tab, column A holds the row labels and fields start in column B:

| Row | Label | Content |
|---|---|---|
| 1 | `field` | Field name. Columns B and C are always `Name` and `Slug`. |
| 2 | `type` | Field type (dropdown). |
| 3 | `required` | `yes` / `no` |
| 4 | `help text` | Text shown to editors under the field. **Required for every field.** |
| 5 | `config` | Option: choices separated by `\|`. Reference / MultiReference: the tab it points to. |
| 6 | `group` | Optional field group. |
| 8+ | `item` | One CMS item per row. |

### Supported field types

`PlainText`, `RichText`, `Image`, `MultiImage`, `Link`, `Email`, `Phone`, `Number`, `DateTime`, `Switch`, `Color`, `VideoLink`, `File`, `Option`, `Reference`, `MultiReference`

### Item values

| Type | Format |
|---|---|
| Reference | Slug of the referenced item (`jane-doe`) |
| MultiReference | Slugs separated by `\|` (`webflow\|cms`) |
| Image / File | Public URL |
| MultiImage | URLs separated by `\|` |
| Switch | `yes` / `no` |
| DateTime | `YYYY-MM-DD` or `YYYY-MM-DD HH:MM` |
| RichText | Clean HTML (`p`, `h2`, `h3`, `ul`, `ol`, `li`, `a`, `strong`, `em`, `blockquote`, `img`) |
| Option | The choice name |

## Requirements

- Claude with the **Webflow connector (MCP)** connected.
- Code execution enabled in Claude.

## Installation

**Claude (web / desktop):** download this repo as a ZIP and upload it in *Customize > Skills > Add*.

**Claude Code:**

```bash
git clone https://github.com/fernandocomet/webflow-cms-from-sheet ~/.claude/skills/webflow-cms-from-sheet
```

## Usage

Attach your sheet and ask something like:

> Build the CMS from this sheet in my site X

Tips:

- Keep every field as `required: no` while building the site and tighten it later.
- Use public image URLs. If your images aren't hosted anywhere, upload them to the Webflow asset panel first and use those URLs.

## Known limitations

- Help text on the built-in `Name` and `Slug` fields may have to be set manually in the Designer if the API doesn't allow it.
- Items are created as drafts; publishing is up to you.
- Up to 100 items per request; larger imports are split into batches.

## Author

Fernando Comet
