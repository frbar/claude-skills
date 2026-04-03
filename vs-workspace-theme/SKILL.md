---
name: vs-workspace-theme
description: Configure VS Code workspace theme colors and title bar identifier so you can instantly recognize which workspace you're in.
allowed-tools: Read, Write
---

## What this skill does

1. Asks for a **context description** (e.g. "production AWS infra", "client billing frontend")
2. Asks for an **accent color** — or suggests one if you skip it
3. Generates a `.vscode/settings.json` with:
   - Title bar and status bar colored with the accent
   - Window title prefixed with a SHORT_CAPS_IDENTIFIER derived from the context

## Prompt

You are helping the user configure a distinct VS Code workspace theme so they can instantly identify which workspace they're looking at.

### Step 1 — Gather inputs

Ask the user two questions (you may ask both at once):

1. **Context**: "What is this workspace for?" (e.g. "production Terraform", "mobile app frontend", "data pipeline"). This will be turned into a SHORT_CAPS_IDENTIFIER like `PROD-TF`, `MOB-FRONTEND`, `DATA-PIPE`.

2. **Accent color** (optional): "What hex color should I use for the title bar and status bar? Leave blank and I'll suggest one." The color must be dark enough that white text (`#ffffff`) is readable on top of it (luminance < 0.35). If the user provides a light color, adjust it darker automatically. If they skip it, pick a distinctive dark color — vary suggestions across common workspaces (avoid always defaulting to the same shade).

### Step 2 — Derive values

From the context string, generate a SHORT_CAPS_IDENTIFIER:
- Extract 1–3 meaningful words, abbreviate each to 2–5 uppercase letters
- Join with a hyphen
- Keep it under 15 characters total
- Examples: "production AWS infra" → `PROD-AWS`, "client billing frontend" → `BILLING-FE`, "data ingestion pipeline" → `DATA-INGEST`

If no color was provided, suggest a visually distinct dark color. Good defaults vary by context type:
- Infrastructure / ops → deep slate blue `#1a2a4a`
- Frontend / UI → deep teal `#1a3a35`
- Backend / API → dark purple `#2d1b4e`
- Data / analytics → dark green `#1a3320`
- Staging / QA → dark amber `#3a2800`
- Production → dark red `#3a1010`
- Default fallback → `#1e2a3a`

### Step 3 — Output the settings file

Show the user the two derived values before writing:
- Identifier: `SHORT_CAPS_IDENTIFIER`
- Color: `#xxxxxx`

Then write the following content to `.vscode/settings.json` in the current workspace (merge with existing content if the file already exists — do not overwrite unrelated settings):

```json
{
  "workbench.colorCustomizations": {
    "titleBar.activeBackground": "#ACCENT_COLOR",
    "titleBar.activeForeground": "#ffffff",
    "titleBar.inactiveBackground": "#ACCENT_COLOR",
    "titleBar.inactiveForeground": "#cccccc",
    "statusBar.background": "#ACCENT_COLOR",
    "statusBar.foreground": "#ffffff"
  },
  "window.title": "SHORT_CAPS_IDENTIFIER - ${folderName} - ${activeEditorShort}",
  "workbench.colorTheme": "Bluloco Light"
}
```

Confirm the file was written and tell the user to reload the window (`Ctrl+Shift+P` → "Reload Window") if the colors don't apply immediately.
