---
name: adr
description: Helps write Architecture Decision Records. Invoke with /adr or when an architectural decision needs to be documented.
allowed-tools: Read, Write
---

# ADR Writer

## Before You Start

1. Read `CLAUDE.md` at the project root and look for an `## ADR` section.
   It should define:
   - `Brief`: path to the project brief file
   - `Template`: path to the ADR template file
   - `ADR directory`: where ADRs are stored

2. If no `## ADR` section exists in `CLAUDE.md`, fall back to these defaults:
   - Brief: `docs/brief.md`
   - Template: `docs/adr/000-adr-template.md`
   - ADR directory: `docs/adr/`

3. If neither works, ask the user to provide the paths before proceeding.

4. Read both the brief and the template files before generating anything.

## Your Role

You **reformulate and structure** what the user provides. You do not generate invented content.

## What the User Provides (in any order)

- The current state / observed problem
- Context and constraints
- Options being considered
- Optional: the decision made and its rationale

## What You Produce

**1. Reformulate** each element into the appropriate template sections:
- `Context`: the problem, factual and clear - don't rephrase the brief, as we assume the user knows the context, but make it concise and focused on the issue at hand
- `Decision`: if a decision was made, articulate it with its rationale, keeping consistency with the brief
- Options: list them clearly if several were mentioned

**2. Complete based on status:**

- ✅ **Decision made** → propose the `Consequences` section (Easier / Harder)
  using the brief to assess real project impact

- ❓ **Not yet decided** → do not make the decision, but:
  - Complete or suggest missing **decision drivers** (constraints, evaluation criteria)
  - Propose **anticipated consequences** per option if multiple were listed

**3. Generate the Markdown draft** with:
- Number `NNN` → use `???` if unknown
- Today's date
- Inferred status (`Proposed`, `WIP`, or `Accepted`)
- `## Authors` left blank

Always use English for the output, regardless of the input language.

## What You Do NOT Do
- Make the decision on behalf of the team
- Invent context not mentioned by the user
- Fill in `## Authors`

## Output

Once the ADR draft is ready, write it immedialely to the appropriate file in the ADR directory:

1. Determine the next ADR number by listing existing files in the ADR directory
   (e.g. `docs/adr/`) and incrementing the highest number found.
2. Write the file directly to `docs/adr/NNN-short-title.md`
   where `NNN` is zero-padded to 3 digits (e.g. `042`).

If the ADR directory path is defined in `CLAUDE.md`, use that path.