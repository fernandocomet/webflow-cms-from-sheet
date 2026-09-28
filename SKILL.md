---
name: "webflow-cms-from-sheet"
description: "Build Webflow CMS collections, fields (with help text and field groups) and items from a multi-tab spreadsheet via the Webflow MCP. Use when asked to create or import a CMS from an xlsx, Google Sheet or CSV."
---

# Webflow CMS from a spreadsheet

Turn a spreadsheet into Webflow CMS collections using the Webflow MCP (`data_cms_tool`). Every field gets its help text. Items are imported as drafts.

## Input format

An `.xlsx` (or a Google Sheet exported as xlsx). A plain CSV can only hold one collection; if the user gives one CSV per collection, treat each file as a tab named after the file.

- **`_readme`**: documentation for humans. Ignore it.
- **`_collections`**: one row per collection with columns `tab`, `displayName`, `singularName`, `slug`.
- **Every other tab is one collection.** Column A holds row labels; field columns start at column B.

| Row | Label | Meaning |
|---|---|---|
| 1 | field | Field display name. Columns B and C are always `Name` and `Slug` (built-in fields). |
| 2 | type | PlainText, RichText, Image, MultiImage, Link, Email, Phone, Number, DateTime, Switch, Color, VideoLink, File, Option, Reference, MultiReference |
| 3 | required | yes / no |
| 4 | help text | Help text shown to editors under the field. |
| 5 | config | Option: choices separated by `\|`. Reference / MultiReference: the tab name it points to. |
| 6 | group | Optional field group name. |
| 7 | (empty) | |
| 8+ | item | One CMS item per row. |

Item values: Reference = target item slug; MultiReference = slugs separated by `|`; Image/File = public URL; MultiImage = URLs separated by `|`; Switch = yes/no; DateTime = `YYYY-MM-DD` or `YYYY-MM-DD HH:MM`; RichText = clean HTML; Option = the choice name.

## Step 1 - Parse and validate (before touching Webflow)

Parse the file with Python (openpyxl) into a JSON plan: collections, fields per collection and items. Then validate and report every problem in one list:

- Unknown types, or `required` values other than yes/no.
- **Missing help text on any field.** This is a hard requirement: list every field without it and ask the user to fill them in or approve help text you draft.
- Option fields without choices; item values that are not one of the choices.
- Reference/MultiReference `config` pointing to a tab that doesn't exist.
- Item references to slugs that don't exist in the target tab.
- Duplicate slugs in a tab, empty required values, invalid dates, numbers or URLs.
- Circular references between collections (flag them: see step 3).

Show a short summary (collections, field count, item count) plus the problems. Do not continue until the user confirms.

## Step 2 - Target site

If no site ID was given, list sites with `data_sites_tool` and ask which one. Then read existing collections with `get_collection_list`.

- If a collection with the same slug or name exists, don't recreate it. Compare its fields with the sheet, add only missing fields and report differences (type or help text) without overwriting unless the user approves.

## Step 3 - Create collections and fields

1. Sort collections so that referenced collections are created before the ones pointing to them. If there is a cycle, create all collections first, then add the reference fields in a second pass.
2. `create_collection` with `displayName`, `singularName` and `slug`. Keep the returned collection IDs in a map `tab -> collectionId`.
3. Create fields in column order, always passing `displayName`, `isRequired` and `helpText`:
   - Static types: `create_collection_static_field`.
   - Option: `create_collection_option_field` with `metadata.options` = [{name}...].
   - Reference / MultiReference: `create_collection_reference_field` with `metadata.collectionId` from the map.
   Batch several field creations into one `data_cms_tool` call.
4. **Name and Slug** are created by Webflow. Read the collection with `get_collection_details`, find the `name` and `slug` fields and set their help text with `update_collection_field`. If the API rejects it, tell the user those two must be set manually in the Designer.
5. **Field groups**: if any field has a group, call `get_collection_field_groups`, map field names to IDs, and call `update_collection_field_groups` with groups in first-appearance order and fields in column order.
6. Re-read the schema with `get_collection_details` to get the field slugs Webflow assigned. Build a map `field display name -> field slug` per collection.

## Step 4 - Import items

1. Import collections in the same dependency order.
2. **Dry run first**: create one item from the first collection that has items, show the user the result, and wait for approval.
3. Then create the rest with `create_collection_items` (up to 100 per call), with `isDraft: true`. Convert values:
   - Reference: look up the target item ID by slug (from items you created or `list_collection_items`).
   - MultiReference: array of IDs.
   - Image / File: `{ "url": "..." }`. MultiImage: array of those.
   - Switch: boolean. Number: number. DateTime: ISO 8601.
   - Option: the option ID from the schema (fall back to the name if the API accepts it).
4. If a required field is empty for an item, don't invent content. Report it and ask whether to make the field optional or skip the item.
5. Never publish items or the site unless the user asks.

## Step 5 - Verify and report

- Re-read each collection: field count, that every field has help text, groups, and item count.
- Spot-check one item per collection, including references.
- Report in a short table: collection, fields created, items imported, issues. List anything left for manual work.

## Notes

- If the MCP errors with type mismatches (e.g. expects an array but receives a string), refresh the Webflow connector and retry once before trying another route.
- Use a single `session_id` and `agent_id` for all calls in the task.
- If the user doesn't have a sheet yet, offer to generate a template in this exact format with an example (Categories, Authors, Posts).