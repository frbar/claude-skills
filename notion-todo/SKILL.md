---
name: notion-todo
description: Adds one or more tasks to a personal Notion todos page, routing them to the right section based on project or type. Creates the section if it does not exist yet.
allowed-tools: Bash, AskUserQuestion, mcp__claude_ai_Notion__authenticate, mcp__claude_ai_Notion__complete_authentication
---

# Notion Todo

Adds tasks to a personal Notion todos page organized in sections by project/type.

---

## Step 1 — Parse user input

Extract from what the user provided:
- **Tasks**: one or more descriptions (one per line, or comma/semicolon-separated)
- **Project hints**: explicit keywords ("for Advisor", "in Shine", "Architecture", etc.)
- **Priority or due date** if mentioned

---

## Step 2 — Infer the project

Attempt to identify the project in this priority order:

1. **Explicit mention** in the task text (e.g. "fix the Advisor bug") → use that mention
2. **Current working directory**: extract the main folder name (e.g. `advisor-prod-agentic-workflow` → `Agentic Workflow`, `shine-api` → `API`, `claude-skills` → `Claude / Personal Tools`)
3. **Current git branch** if available (`git branch --show-current`): extract the business context (e.g. `feature/implement-api-gateway` → `API Gateway`)
4. **Keywords in the task description**: known project names, services, components

If after these 4 steps the project is still ambiguous, or if the task seems cross-cutting (not tied to a specific technical project), mark it as **`to clarify`** and move to Step 3.

---

## Step 3 — Upfront question (if needed)

If and only if information remains ambiguous after Step 2, ask **one single question** with `AskUserQuestion` **before any Notion call**.

Group all ambiguities into one question. Examples:
- Project not identifiable → offer existing sections + "Other / Misc"
- Multiple tasks with different projects → confirm the mapping all at once
- Urgent task with no priority → suggest a priority

Never ask a question after starting to call Notion.

---

## Step 4 — Notion authentication

Check whether real Notion tools are available (beyond `authenticate` and `complete_authentication`).

If only auth tools are present:
1. Call `mcp__claude_ai_Notion__authenticate` to get the authorization URL
2. Display the URL to the user with the message:
   > "Open this link in your browser to connect Notion, then paste the redirect URL here."
3. Wait for the user to provide the callback URL
4. Call `mcp__claude_ai_Notion__complete_authentication` with that URL
5. Continue with the real Notion tools now available

---

## Step 5 — Find the Notion todos page

It is named `TO-DO w/ Claude`, it's a private page owned by the user.

If no page is found, inform the user and ask them to provide the title or ID of their todos page.

---

## Step 6 — Find or create the section

Read the page content (`notion_retrieve_block_children`) to identify existing blocks.

### Map the project to a section
- Look for a `heading_2` or `heading_3` block whose text matches (case-insensitive) the identified project
- Accept partial matches (e.g. block "🚀 Advisor" → project "Advisor")

### If the section exists
Identify its block ID to insert todos under it.

### If the section does not exist
Create a new `heading_2` block at the end of the page with the project name (title-cased), then use that new block as the target section.

---

## Step 7 — Insert the tasks

For each task identified in Step 1, create a `to_do` block with:
- `checked: false`
- The task text prefixed with the current date as `` `dd/mm` `` (zero-padded, always 4 characters, e.g. `` `17/05` ``) followed by ` - ` (space-dash-space), then the task text. Example: `` `17/05` - Fix the retry bug on webhooks ``

Insert these blocks **after the section heading** and **before the next heading** (i.e. at the end of the section, not anywhere on the page).

Use `notion_append_block_children` on the section's heading block (or directly on the page if the API requires it), positioning new blocks after the last child of the section.

---

## Step 8 — Confirm to the user

Display a concise summary:

```
✓ 2 task(s) added to Notion
  → Section: 🚀 Advisor
  • `17/05` - Fix the retry bug on webhooks
  • `17/05` - Review the routing module docs
```

If a section was created, mention it:
```
  (new section created)
```

---

## Rules

- **Always use English** for all output, questions, confirmation messages, and content inserted into Notion — regardless of the language the user writes in. Translate task text to English if needed.
- **Never ask a question after Step 3.** All clarifications happen upfront.
- **Do not reformulate tasks**: keep the text as the user provided it, possibly with a leading capital letter.
- **Do not add invented properties** (priority, date, assignee) if the user did not mention them.
- **On Notion error** (page not found, permission denied): inform the user clearly and suggest a fix (check that the page is shared with the Notion integration).
- **Multiple tasks at once**: insert them all in a single `notion_append_block_children` call to minimize API calls.
