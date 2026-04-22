\---

name: create-new-branch-and-prep-pr

description: Create a new git branch from current changes, switch to it, and prepare a meaningful commit message (without staging).

allowed-tools: Bash, Read, Glob, Grep

\---



\# Create Branch



\## Goal



Create a new branch, switch to it, and prepare a commit message for the pending changes — \*\*without staging anything\*\*.



\## Steps



\### 1. Understand the current changes



Run these in parallel:

\- `git status` — list modified/untracked files

\- `git diff` — inspect unstaged changes

\- `git log --oneline -5` — understand recent commit style



\### 2. Determine the branch name



Analyse the changes to infer:

\- \*\*Type\*\*: `feat`, `fix`, `chore`, `refactor`, `docs`

\- \*\*Ticket ID\*\* (if the user mentioned one, e.g. `EE-123`, `COM-456`) — include it

\- \*\*Short slug\*\*: kebab-case, max 5 words describing what the change does



Branch naming conventions observed in this repo:

\- `feat/<TICKET-ID>-short-description` (e.g. `feat/EE-603-ecr-lambda-image`)

\- `fix/<TICKET-ID>-short-description` (e.g. `fix/advisor-gw-disable-kafka-creds`)

\- `chore-<TICKET-ID>-short-description` (e.g. `chore-PCS-5272-identity-service-ssm-params`)



If the user provided a branch name or ticket ID in their message, use it directly.

If unclear, ask the user before creating the branch.



\### 3. Create and switch to the branch



```bash

git checkout -b <branch-name>

```



\### 4. Prepare the commit message



Analyse the diff to write a conventional commit message:



```

<type>(<optional-scope>): <short imperative summary>



<optional body: what changed and why, bullet points if multiple files>

```



Commit message rules:

\- First line ≤ 72 characters, imperative mood (e.g. "add", "update", "remove")

\- Body only if there are multiple distinct changes worth explaining

\- Reference ticket ID in the body if known (e.g. `Refs: EE-603`)

\- \*\*Do not mention IDE/editor config file updates\*\* (e.g. `.vscode/`, `.idea/`, `.editorconfig`) in the commit message — focus on the meaningful code/infrastructure changes only



\*\*Do NOT run `git add` or `git commit`.\*\* Only display the suggested message.



\### 5. Prepare the PR description



This is the template:



```
# Mandatory



\## WHAT

\*Provide a synthetic and technical explanation, an overview of what is changed by this PR. No need to be exhaustive, give us the big picture.\*



\## WHY

\*Provide the reason(s) and/or the context leading to this change. It is probably more on the product/feature side.\*



\## Security Impacts

\*Provide a description of the security impacts of this changes. Addition, modification or deletion of permissions, etc.\*



\# Optional



\## Changes / Technical explanations

\*If needed, provide a complete or deeper technical explanations (e.g. complex logic description, global architecture description)\*



\## Expected Impacts

\*Provide the expected consequences of this change\*



\[ ] this change impacts a Terraform shared module



\## Changes outside the codebase / Additional informations

\*If applicable, provide other changes in other repos linked to this PR, or any other useful informations (documentation, link, ...)\*

```

PR description rules:

\- Be brief, synthetic
- Focus on important changes, there shouldn't be 10 different topics
- Ignore changes made to editor config files



\### 6. Output



Tell the user:

1\. The branch name created and that they are now on it

2\. The suggested commit message (in a fenced code block)
3. The suggest PR description (in a fenced code block, markdown style, in a way the user can copy/paste directly in GitHub)

4. A reminder: files are \*\*not staged\*\* — they can review, adjust, then stage manually

5. A hint: once files are staged, they can run `/commit` (or ask Claude) to refine the commit message based on what was actually staged vs. the full diff



