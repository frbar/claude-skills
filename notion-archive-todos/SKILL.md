---
name: notion-archive-todos
description: Archives completed (checked) tasks from the Notion todos page into a DONE section, formatted with completion date, creation date, source section, and original text.
allowed-tools: mcp__claude_ai_Notion__authenticate, mcp__claude_ai_Notion__complete_authentication
---

# Notion Archive Todos

Scans the personal Notion todos page, finds all checked tasks, moves them to a `DONE 🎉` section with a structured archive format.

Always use English for all output and messages.

---

## Step 1 — Notion authentication

Check whether real Notion tools are available (beyond `authenticate` and `complete_authentication`).

If only auth tools are present:
1. Call `mcp__claude_ai_Notion__authenticate` to get the authorization URL
2. Display the URL to the user:
   > "Open this link in your browser to connect Notion, then paste the redirect URL here."
3. Wait for the user to provide the callback URL
4. Call `mcp__claude_ai_Notion__complete_authentication` with that URL
5. Continue with the real Notion tools now available

---

## Step 2 — Read the todos page

The page is named `TO-DO w/ Claude`.

Retrieve all blocks from the page (`notion_retrieve_block_children`). If the page has nested blocks (heading children), retrieve children recursively as needed to capture all `to_do` blocks.

Build an in-memory map of the page structure:
```
[
  { type: "heading_2", text: "🚀 Advisor", id: "...", children: [
    { type: "to_do", checked: false, text: "`17/05` - Fix retry bug", id: "..." },
    { type: "to_do", checked: true,  text: "`15/05` - Add logging",   id: "..." },
  ]},
  { type: "heading_2", text: "DONE 🎉", id: "...", children: [ ... ] },
  ...
]
```

---

## Step 3 — Collect checked tasks

Find all `to_do` blocks where `checked: true`, across all sections.

For each checked task, record:
- `block_id`: the Notion block ID (for deletion)
- `section`: the text of the nearest preceding `heading_2` or `heading_3` (the source section), stripped of leading/trailing emojis and whitespace for the archive label (e.g. `"🚀 Advisor"` → `"Advisor"`)
- `creation_date`: extracted from the date prefix at the start of the task text. Try two patterns in order:
  1. Backtick-formatted: `` /^`(\d{2}\/\d{2})`\s*-\s*/ `` (e.g. `` `15/05` - Fix bug ``)
  2. Plain (no backticks): `/^(\d{2}\/\d{2})\s*-\s*/` (e.g. `15/05 - Fix bug`)

  Both are treated as valid, well-formed dates. If neither matches, use `??/??`.
- `original_text`: the task text with the date prefix (and its surrounding `` ` `` if present) removed (trim leading/trailing whitespace)

If no checked tasks are found, stop and inform the user:
> "No completed tasks found on the todos page."

---

## Step 4 — Format archive entries

Today's date is the completion date. Format it as `dd/mm` (zero-padded).

For each checked task, build the archive line:

```
`dd/mm` - `dd/mm` - Section - Original task text
  ↑           ↑        ↑
completion  creation  source section
```

Example:
```
`17/05` - `15/05` - Advisor - Add logging to the retry handler
```

If the creation date was not found in the original text, use `??/??`:
```
`17/05` - `??/??` - Advisor - Some task without a date prefix
```

Do **not** translate or alter the original task text beyond removing the date prefix.

---

## Step 5 — Find or create the DONE section

Look for a `heading_2` block whose text contains `DONE` (case-insensitive) in the page structure.

- **If it exists**: use it as the target section. Append new archive entries as children of this heading.
- **If it does not exist**: create a new `heading_2` block at the **end** of the page with the text `DONE 🎉`, then append archive entries as its children.

---

## Step 6 — Insert archive entries

For each formatted archive entry, create a `bulleted_list_item` block (not a `to_do` — these are records, not actionable items).

Insert all entries in a **single** `notion_append_block_children` call on the DONE heading block.

---

## Step 7 — Delete the original checked blocks

Delete each original checked `to_do` block by its `block_id` using `notion_delete_block` (or equivalent delete call).

Delete them all — do not leave any checked tasks in their original sections.

Example: if the collected checked tasks were:
```
{ block_id: "abc-123", section: "Advisor", text: "`15/05` - Add logging..." }
{ block_id: "def-456", section: "API",     text: "`14/05` - Update OpenAPI spec" }
```
Then delete block `abc-123` and `def-456` from the page — their content has already been preserved in the DONE section.

---

## Step 8 — Confirm to the user

Display a concise summary:

```
✓ 3 task(s) archived to DONE 🎉

  • `17/05` - `15/05` - Advisor - Add logging to the retry handler
  • `17/05` - `14/05` - API - Update the OpenAPI spec
  • `17/05` - `??/??` - Personal Tools - Clean up old scripts
```

If the DONE section was created, mention it:
```
  (DONE 🎉 section created)
```

---

## Rules

- **No upfront questions.** This skill is fully automatic — it archives all checked tasks without asking.
- **Do not modify unchecked tasks.** Only `checked: true` blocks are touched.
- **Do not reorder sections.** The DONE section is always at the bottom if created fresh.
- **On Notion error**: inform the user clearly and suggest checking that the page is shared with the Notion integration.
