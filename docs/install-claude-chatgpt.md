# Use the BillionVerify Skill in Claude and ChatGPT

This guide covers installing and verifying the BillionVerify Agent Skill in Claude Code, claude.ai / Claude Desktop and ChatGPT (Codex CLI and web). You are done when every "Verify" section passes.

## 0. Skill or MCP: pick the right one

| | Skill (this repository) | MCP server ([billionverify-mcp](https://github.com/BillionVerify/billionverify-mcp)) |
|---|---|---|
| How it works | A `SKILL.md` guide; the agent calls the BillionVerify REST API with `curl` | The agent calls 11 tools on the hosted server over MCP |
| Authentication | API key (`BILLIONVERIFY_API_KEY` environment variable) | OAuth sign-in with your BillionVerify account, no API key |
| Needs | An agent that can run shell commands and reach `api.billionverify.com` | A client that supports remote MCP |
| Best for | Claude Code, Codex CLI, Cursor and other local agents; uploading files for bulk verification | ChatGPT web, claude.ai, Claude Desktop |

**On ChatGPT web and claude.ai, prefer the MCP connector.** See the [billionverify-mcp connection guide](https://github.com/BillionVerify/billionverify-mcp/blob/main/docs/connect-chatgpt-claude.md). Using the skill on claude.ai requires enabling network egress and giving the API key in the chat; see section 3.

What the skill covers: single verification, batch verification (up to 50), CSV / Excel / TXT file upload for bulk verification (max 20MB, 100,000 emails), job status, result download with status filters, credit balance, verification history and statistics, and webhook management.

## 1. Get an API key

1. Sign in at https://billionverify.com/auth/sign-in?next=/home/api-keys .
2. Create an API key and copy it (it looks like `sk_...`).
3. Check that it works:

```bash
curl -s https://api.billionverify.com/v1/credits -H "BV-API-KEY: sk_your_key"
```

Expected: `"success":true` and a `credits_balance`.

## 2. Claude Code

### 2.1 Install

Pick one:

```bash
# Option A: skills CLI (recommended), install for Claude Code
npx skills add BillionVerify/billionverify-skill -a claude-code

# Option B: clone into your personal skills directory (all projects)
git clone https://github.com/BillionVerify/billionverify-skill.git ~/.claude/skills/billionverify
```

For a single project, clone into that project's `.claude/skills/billionverify/` instead.

### 2.2 Set the API key

Claude Code inherits environment variables from the shell that starts it:

```bash
# Add to your shell profile so it persists (zsh example)
echo 'export BILLIONVERIFY_API_KEY=sk_your_key' >> ~/.zshrc
source ~/.zshrc
```

**Restart** `claude` afterwards; a running session will not see the new variable.

### 2.3 Verify

1. Start `claude` and run `/skills`; `billionverify` should be listed.
2. Send the prompts below and confirm Claude runs the matching `curl` command (approve the Bash permission prompt):

| # | Prompt | Expected request | Pass criteria |
|---|---|---|---|
| 1 | `Use billionverify to check my credit balance` | `GET /v1/credits` | Balance matches the website dashboard |
| 2 | `Use billionverify to verify support@billionverify.com` | `POST /v1/verify/single` | Returns `status` and `score` |
| 3 | `Use billionverify to verify a@gmail.com, test@mailinator.com` | `POST /v1/verify/bulk` | Two results, the second is `disposable` |
| 4 | `Show my verification stats for the last 30 days` | `GET /v1/stats?period=30d` | Returns `total_verified` and other fields |
| 5 | `List my last 5 verification records` | `GET /v1/usage?limit=5` | Returns records, including the emails from steps 2 and 3 |
| 6 | Prepare an `emails.csv` with a single `email` column, then send `Use billionverify to verify the emails in emails.csv and download the valid ones when it finishes` | `POST /v1/verify/file` → `GET /v1/verify/file/{id}` → `GET .../results?valid=true` | A task_id is returned and a CSV is downloaded when the job completes |

## 3. claude.ai / Claude Desktop

### 3.1 Requirements and limits

- Pro / Max / Team / Enterprise plan with **Code execution and file creation** enabled.
- Skills run in the claude.ai sandbox, which **cannot reach `api.billionverify.com` by default**. Allow the domain as described in 3.2.
- The claude.ai sandbox **has no environment variables**, so the skill asks for your API key in the chat. A key pasted into a chat stays in the conversation history; use the MCP connector if that is a concern.
- Skills uploaded to claude.ai only work in claude.ai / Claude Desktop and do not sync to Claude Code. On Team / Enterprise each member uploads the skill themselves.

### 3.2 Enable code execution and network egress

Personal Pro / Max:

1. **Settings → Capabilities**: turn on **Code execution and file creation**.
2. On the same page, turn on network egress (**Allow network egress**). If there is a domain allowlist, add `api.billionverify.com`.

Team / Enterprise: an Owner sets the egress policy under **Organization settings → Capabilities**. Choose "package managers + specific domains" and add `api.billionverify.com`, or choose "all domains". The default "package managers only" makes skill requests fail.

### 3.3 Package and upload

1. Build a zip whose root is a folder containing `SKILL.md`:

```bash
git clone https://github.com/BillionVerify/billionverify-skill.git billionverify
cd billionverify && git archive --format=zip --prefix=billionverify/ -o ../billionverify-skill.zip HEAD
```

   Or click **Code → Download ZIP** on GitHub; that zip's root folder is `billionverify-skill-main/` and can be uploaded as is.

2. claude.ai → **Settings → Capabilities** (**Customize → Skills** in some versions) → **Upload skill**, choose `billionverify-skill.zip`.
3. Confirm `billionverify` appears in the Skills list and is enabled.

### 3.4 Verify

1. Start a new chat and send `Use billionverify to check my credit balance`.
2. Claude says the API key is not set and asks for it; send your key.
3. Expected: Claude runs `curl` in the sandbox and returns your balance.
4. Continue with prompts 2 to 5 from 2.3. For prompt 6, attach the CSV to the chat first.

| Symptom | Cause and fix |
|---|---|
| `Could not resolve host`, connection timeout, or a proxy `403 Forbidden` | Egress to `api.billionverify.com` is not allowed; go back to 3.2 |
| `Invalid or inactive API Key` | Wrong or disabled key; re-check it with step 1 |
| Claude does not use the skill | Make sure it is enabled in the Skills list; mention "use billionverify" in the prompt |

## 4. ChatGPT

### 4.1 ChatGPT web / desktop

Use the MCP connector: follow [section 1 of the billionverify-mcp guide](https://github.com/BillionVerify/billionverify-mcp/blob/main/docs/connect-chatgpt-claude.md#1-chatgpt-web). OpenAI has no clear documentation yet on uploading custom skills in ChatGPT web or whether that sandbox can reach external APIs. We have not verified it and make no promises about it.

### 4.2 Codex CLI

Codex CLI supports the Agent Skills standard. Install the same way as Claude Code:

```bash
npx skills add BillionVerify/billionverify-skill -a codex
export BILLIONVERIFY_API_KEY=sk_your_key   # also add it to ~/.zshrc
```

Start `codex` and verify with the prompts from 2.3. Codex's default sandbox may block network access. If you see connection errors, approve network access for the command when Codex asks, or enable network access for the workspace in your Codex configuration.

## 5. Other agents (Cursor, Gemini CLI, GitHub Copilot, etc.)

```bash
npx skills add BillionVerify/billionverify-skill                # choose agents interactively
npx skills add BillionVerify/billionverify-skill -a cursor      # a specific agent
```

Every agent needs `BILLIONVERIFY_API_KEY` in its environment and access to `api.billionverify.com`.

## 6. Update and uninstall

- Installed with the skills CLI: run `npx skills add BillionVerify/billionverify-skill ...` again to update.
- Cloned manually: `cd ~/.claude/skills/billionverify && git pull`.
- claude.ai: delete the old version from the Skills list and upload the new zip.
- Uninstall: delete the skills directory, or remove it from the Skills list on claude.ai.
