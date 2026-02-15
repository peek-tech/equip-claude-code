---
name: setup-claude-agents
description: Analyze a project and install the right Claude Code agents, skills, slash commands, MCP servers, and plugins from community registries
allowed-tools: Read, Write, Edit, Glob, Grep, WebFetch, Bash(curl *), Bash(gh api *), Bash(mkdir *), Bash(ls *), Bash(wc *), Bash(rm *)
---

# Setup Claude Agents

Analyze the current project and install the right Claude Code agents, skills, slash commands, MCP servers, and plugins from three registries:

1. **[awesome-claude-code](https://github.com/hesreallyhim/awesome-claude-code)** — curated skills, commands, resources (CSV)
2. **[Official MCP Registry](https://registry.modelcontextprotocol.io)** — searchable API for MCP servers
3. **[Plugin marketplace](https://github.com/ComposioHQ/awesome-claude-plugins)** — curated Claude Code plugins (JSON)

Falls back to a hardcoded registry if external sources are unavailable.

## FORBIDDEN COMMANDS

**NEVER run `claude mcp add`, `claude mcp remove`, or any `claude` CLI command via Bash.** This causes a fatal "nested session" crash. The ONLY way to install MCP servers is by editing `.mcp.json` directly with Read and Write tools (see Step 5a).

## Arguments

`/setup-claude-agents [filter-level]`

Controls quality filtering for MCP Registry API results. Parse `$ARGUMENTS` for:

| Level | Description |
|-------|-------------|
| `strict` | Trusted publishers pass. Others: stars ≥10 AND downloads ≥1K/month. Repo URL required. |
| `moderate` | **(default)** Trusted publishers pass. Others: stars ≥5 OR downloads ≥500/month. Repo URL required. |
| `light` | Only check: `repository.url` exists, `status == "active"`. No API calls. |
| `unfiltered` | No quality checks. |

Unrecognized values default to `moderate`.

**Trusted publishers** (bypass quality checks at all levels):
`@modelcontextprotocol/*`, `@anthropic-ai/*`, `awslabs.*`, `com.stripe/*`, `@playwright/*`, `@twilio-alpha/*`

**Quality check commands** (for non-trusted registry results):
- **GitHub stars**: `gh api repos/{owner}/{repo} --jq '.stargazers_count'`
- **npm downloads**: `curl -sL "https://api.npmjs.org/downloads/point/last-month/<pkg>"` → `.downloads`
- **PyPI downloads**: `curl -sL "https://pypistats.org/api/packages/<pkg>/recent"` → `.data.last_month`

If a check fails (404, timeout), treat that signal as absent. In `strict`, the server must pass the remaining check. In `moderate`, it passes if the other check succeeds.

## Workflow

### Step 1: EXPLORE the project

Scan the project to build a raw inventory. Skip missing files silently. Steps 1a-1d are independent — run them in parallel.

**1a. Dependencies** — Read all manifests that exist (`package.json`, `pyproject.toml`, `requirements.txt`, `Cargo.toml`, `go.mod`, `Gemfile`, `composer.json`, `build.gradle(.kts)`, `pom.xml`, `mix.exs`, `pubspec.yaml`) and extract dependency names. For monorepos, also scan workspace directories.

**1b. Languages** — Glob for source file extensions at top-level and `src/` (e.g., `*.ts` → TypeScript, `*.py` → Python, `*.rs` → Rust, `*.go` → Go).

**1c. Infrastructure** — Check for: `Dockerfile`, `docker-compose.yml`, `.github/workflows/`, `.gitlab-ci.yml`, `*.tf`, `cdk.json`, `serverless.yml`, `prisma/`, `migrations/`, `*.sql`, `.claude-plugin/`, `.claude/`, `.mcp.json`, `pnpm-workspace.yaml`, `nx.json`, `turbo.json`.

**1d. Documentation** — Read `README.md`, `SPEC.md`, `PLAN.md` if present. Run a top-level `ls` for overall structure.

**1e. Existing manifest** — Read `.claude/.setup-manifest.json` if it exists. This tracks items installed by a previous run. Pass this to Step 4 so recommendations can distinguish new vs already-installed items, and to Step 7 for reconciliation.

### Step 2: FETCH registries

Run these downloads in parallel with Step 1 where possible.

**Community registry:**
```bash
curl -sL -o /tmp/awesome-claude-code-registry.csv "https://raw.githubusercontent.com/hesreallyhim/awesome-claude-code/main/THE_RESOURCES_TABLE.csv"
```
Verify the header contains `ID,Display Name,Category,...`. On failure, set `registry_available = false`. Do NOT read the entire CSV — use Grep to search by keyword later. CSV columns: `ID, Display Name, Category, Sub-Category, Primary Link, Secondary Link, Author Name, Author Link, Active, Date Added, Last Modified, Last Checked, License, Description, Removed From Origin, Stale, ...`. When reviewing grep results, check the Category column to confirm relevance — keyword matches in Description can be false positives.

**Plugin marketplace:**
```bash
curl -sL -o /tmp/claude-plugins-marketplace.json "https://raw.githubusercontent.com/ComposioHQ/awesome-claude-plugins/master/marketplace.json"
```
On failure, set `plugins_available = false`. The JSON is an object — plugins are in the `.plugins[]` array, each with `name`, `category`, `description`, `author`, `tags`.

**MCP Registry** — queried directly in Step 4, no upfront download needed.

### Step 3: ANALYZE

Build a **tech stack fingerprint** from Step 1:

- **languages** — from file extensions
- **dependencies** — package names from manifests
- **infrastructure** — patterns from 1c

Derive **search terms**: lowercase keywords from the fingerprint. Include language names, dependency names (strip scopes, split compound names), infrastructure patterns, and terms from docs. Add known associations (Next.js → `react`, Nuxt → `vue`, SvelteKit → `svelte`). Skip utility deps (`lodash`, `chalk`, `dotenv`, etc.).

### Step 4: RECOMMEND

Search registries and present recommendations in **7 sections**. Note credential requirements. If a manifest exists from a previous run (Step 1e), mark already-installed items as such and highlight what's new vs what will be removed. **Ask the user to confirm before proceeding to Step 5.**

#### 4.1 MCP Servers

**a) Search the Official MCP Registry** for relevant search terms (frameworks, databases, cloud providers — skip generic language names):

```bash
curl -sL "https://registry.modelcontextprotocol.io/v0.1/servers?search=<term>&version=latest&limit=5"
```

Response shape:
```json
{
  "servers": [{
    "server": {
      "name": "org/server-name", "description": "...",
      "repository": {"url": "https://github.com/owner/repo", "source": "github"},
      "packages": [{"registryType": "npm|pypi", "identifier": "@scope/pkg", "transport": {"type": "stdio"}}],
      "remotes": [{"type": "streamable-http", "url": "https://..."}]
    },
    "_meta": {"io.modelcontextprotocol.registry/official": {"status": "active"}}
  }]
}
```
A server may have `packages`, `remotes`, or both — they are not guaranteed. Use `repository.url` for quality filtering.

Convert results to `.mcp.json` config entries:

| Registry field | Config |
|----------------|--------|
| `registryType == "npm"` | `{"type":"stdio","command":"npx","args":["-y","<identifier>"]}` |
| `registryType == "pypi"` | `{"type":"stdio","command":"uvx","args":["<identifier>"]}` |
| `remotes[0].type == "streamable-http"` | `{"type":"http","url":"<url>"}` |

Prefer `packages` over `remotes`. Skip servers with `status` != `"active"`. Cap at **10 registry results**. Never skip servers that need credentials.

**b) Apply quality filtering** (see Arguments section) to non-trusted registry results. Show filtered-out servers in a collapsed note.

**c) Always install `sequential-thinking`** — it is universally useful and does not depend on the project's tech stack:

```json
{"type":"stdio","command":"npx","args":["-y","@modelcontextprotocol/server-sequential-thinking"]}
```

All other MCP servers should come from the registry search in (a) above. The trusted-publisher filter ensures servers from `awslabs.*`, `com.stripe/*`, `@twilio-alpha/*`, `@playwright/*`, and `@anthropic-ai/*` pass quality checks automatically.

#### 4.2 Agent Skills (from community registry)

Search CSV for matching "Agent Skills" entries. Exclude rows where Active is `FALSE` or Stale is `TRUE`.

| Name | Author | Why | Link |
|------|--------|-----|------|

#### 4.3 Slash Commands (from community registry)

Matching "Slash-Commands" entries, **capped at 10**. Always include "Version Control & Git" matches.

| Command | Sub-Category | Why | Link |
|---------|--------------|-----|------|

#### 4.4 Reference Resources (from community registry)

Matching "CLAUDE.md Files" and "Workflows" entries. **Not auto-installed** — links only.

| Resource | Category | Description | Link |
|----------|----------|-------------|------|

#### 4.5 Hooks (from community registry)

Matching "Hooks" entries. Hooks run shell commands automatically, so prompt the user **per hook**: _"Install \<name\>? This runs `<command>` on every \<event\>. (y/n)"_

| Hook | Event | Shell Command | Description | Link |
|------|-------|---------------|-------------|------|

#### 4.6 Agents

Match from the Agent Roles table against the fingerprint.

**Selection rules:**
1. Always include `code-reviewer` and `security-auditor`
2. Always include at least one **implementation agent** matching the project
3. Cap at **6 agents total**

| Role | Tools | Match When |
|------|-------|------------|
| frontend-dev | Read, Write, Edit, Bash, Glob, Grep | UI framework deps OR `.svelte`/`.vue`/`.jsx`/`.tsx` files |
| backend-dev | Read, Write, Edit, Bash, Glob, Grep | Server framework deps (express, fastapi, django, flask, gin, rails, laravel, phoenix, hono, etc.) |
| aws-architect | Read, Write, Edit, Bash, Glob, Grep | `cdk.json` OR `aws-cdk-lib` OR CloudFormation/SAM OR significant `@aws-sdk/*` |
| ai-engineer | Read, Write, Edit, Bash, Glob, Grep | `.claude-plugin/` OR SKILL.md authoring OR AI/LLM deps |
| mcp-developer | Read, Write, Edit, Bash, Glob, Grep | `@modelcontextprotocol/sdk` in deps OR MCP server code |
| devops | Read, Write, Edit, Bash, Glob, Grep | CI/CD configs OR Dockerfiles OR deployment scripts |
| dba | Read, Write, Edit, Bash, Glob, Grep | Database schemas, migration files, ORM deps |
| terraform-engineer | Read, Write, Edit, Bash, Glob, Grep | `*.tf` OR `terraform/` OR `pulumi/` |
| test-automator | Read, Write, Edit, Bash, Glob, Grep | Test framework deps (vitest, jest, playwright, pytest, etc.) |
| python-dev | Read, Write, Edit, Bash, Glob, Grep | Python primary (`pyproject.toml`/`requirements.txt` + `*.py`) |
| rust-dev | Read, Write, Edit, Bash, Glob, Grep | Rust primary (`Cargo.toml` + `*.rs`) |
| go-dev | Read, Write, Edit, Bash, Glob, Grep | Go primary (`go.mod` + `*.go`) |
| code-reviewer | Read, Grep, Glob, Bash | Always |
| security-auditor | Read, Grep, Glob, Bash | Always |

#### 4.7 Plugins (from plugin marketplace)

If `plugins_available`, read `/tmp/claude-plugins-marketplace.json` and iterate the `.plugins[]` array. Match entries whose `tags` overlap with search terms. **Not auto-installed** — present links for `claude plugin add <repo-url>`.

| Plugin | Category | Why | Repo |
|--------|----------|-----|------|

---

**If `registry_available = false`**: Skip sections 4.2-4.5. Use the Fallback Skills Registry instead:

| Name | URL | Good For |
|------|-----|----------|
| svelte5-development | `https://raw.githubusercontent.com/splinesreticulating/claude-svelte5-skill/main/SKILL.md` | Svelte 5 / SvelteKit |
| mcp-builder | `https://raw.githubusercontent.com/anthropics/skills/main/mcp-builder/SKILL.md` | Building MCP servers |
| webapp-testing | `https://raw.githubusercontent.com/anthropics/skills/main/webapp-testing/SKILL.md` | Playwright testing |
| better-auth | `https://raw.githubusercontent.com/VoltAgent/awesome-agent-skills/main/skills/better-auth/SKILL.md` | OAuth, magic links, auth |
| owasp-security | `https://raw.githubusercontent.com/VoltAgent/awesome-agent-skills/main/skills/owasp-security/SKILL.md` | Security best practices |
| stripe | `https://raw.githubusercontent.com/VoltAgent/awesome-agent-skills/main/skills/stripe-best-practices/SKILL.md` | Stripe integration |

### Step 5: INSTALL

After user confirmation, install each artifact type:

#### 5a. MCP Servers

**Do NOT use Bash. Do NOT run any `claude` CLI command.** Follow this exact procedure:

1. **Read** `.mcp.json` with the Read tool (if missing, start with `{"mcpServers":{}}`)
2. **Merge** new server entries into `mcpServers`
3. **Write** the complete JSON back with the Write tool

For servers needing credentials, still install them. Tell the user which env vars to set and where to get them.

#### 5b. Agent Skills (from community registry)

1. Extract `{owner}/{repo}` from the Primary Link and inspect the repo structure:
   ```bash
   gh api repos/{owner}/{repo}/contents/ --jq '.[].name'
   ```
2. If repo has `.claude-plugin/marketplace.json` → tell user to install as plugin, skip
3. If repo has a root `SKILL.md` or a `skills/` directory:
   - Check for SKILL.md files: `gh api repos/{owner}/{repo}/contents/skills --jq '.[].name'`
   - Download each: `curl -sL -o .claude/skills/<name>/SKILL.md "https://raw.githubusercontent.com/{owner}/{repo}/main/<path-to-SKILL.md>"`
   - Create dirs first: `mkdir -p .claude/skills/<name>`
4. If no installable structure → provide link for manual review

#### 5c. Slash Commands (from community registry)

1. Convert blob URLs to raw (replace `github.com` → `raw.githubusercontent.com`, remove `/blob/`)
2. If `.claude/commands/<name>.md` exists and isn't in the manifest, skip and warn
3. Download:
   ```bash
   mkdir -p .claude/commands
   curl -sL -o .claude/commands/<name>.md "<raw-url>"
   ```

#### 5d. Hooks (user-approved only)

1. WebFetch the Primary Link to extract event type, matcher, and shell command
2. Read `.claude/settings.json` (or start with `{}`)
3. Merge into `hooks.<event>[]` — skip duplicates (same event + matcher + command). Each hook entry must use this format:
   ```json
   {
     "hooks": {
       "PreToolUse": [
         {
           "matcher": "Write",
           "hooks": [
             {"type": "command", "command": "./scripts/pre-write-check.sh"}
           ]
         }
       ]
     }
   }
   ```
   The `hooks` array contains objects with `"type"` (`"command"`, `"prompt"`, or `"agent"`) and the corresponding field (`"command"` or `"prompt"`). Do NOT use plain strings.
4. Write `.claude/settings.json`

#### 5e. Agents

Write `.md` files to `.claude/agents/<name>.md`. See Agent Prompt Guidelines below.

#### 5f. Project Rules

1. Read `CLAUDE.md` (or start empty)
2. Collect matching rules, skip those already present
3. Append new rules under `# Project Rules` (create section if needed)

| Rule | Match When |
|------|------------|
| Follow existing file structure and naming conventions. | Always |
| Use Typescript strict mode. | `typescript` in languages |
| Use functional components and hooks. | `react`, `preact`, or `next` in deps |
| Write unit tests for new features. | Always |
| Always output full code blocks. | Always |

#### 5g. Fallback Skills (only if `registry_available = false`)

```bash
mkdir -p .claude/skills/<name>
curl -sL -o .claude/skills/<name>/SKILL.md "<url>"
```

### Step 6: UPDATE MANIFEST

Write `.claude/.setup-manifest.json` tracking everything installed:

```json
{
  "mcp": [], "mcp_from_registry": [], "skills": [], "agents": [],
  "commands": [], "hooks": [{"name":"","event":"","matcher":"","command":""}],
  "rules_added": [], "plugins": [],
  "references": [{"name":"","type":"","link":""}],
  "detected": {"languages":[],"dependencies":[],"infrastructure":[],"search_terms":[]},
  "registries_used": [],
  "registry_fetched_at": "<ISO 8601>",
  "_generated_at": "<ISO 8601>",
  "_comment": "Managed by /setup-claude-agents. Do not edit manually."
}
```

`registries_used` values: `"awesome-claude-code-csv"`, `"mcp-registry-api"`, `"plugin-marketplace"`, `"fallback"`.

### Step 7: RECONCILE on re-run

If `.claude/.setup-manifest.json` exists:

1. Read manifest, compare against new recommendations
2. Remove stale items: agents/skills/commands → delete files; MCP servers → remove from `.mcp.json`; hooks → remove from `.claude/settings.json` if unmodified by user
3. Write updated manifest

Only touch items listed in the manifest. Never remove manually-added items. Rules in `CLAUDE.md` are never removed.

### Error Handling

- **Registry fetch fails**: Set `registry_available`/`plugins_available = false`, continue with remaining sources
- **MCP registry search 404s or times out**: Skip that term, continue with others
- **Quality check API fails**: Treat signal as absent (see Arguments)
- **Skill/command download fails**: Warn user, provide link for manual install, continue
- **`.mcp.json` has invalid JSON**: Warn user, back up the file, start fresh with `{"mcpServers":{}}`

## Agent Prompt Guidelines

Agent `.md` files use frontmatter: `name`, `description`, `tools`, `model` (`sonnet`), and optionally `skills`, `mcpServers`, `memory` (`project` for implementation agents, `user` for review-only). Adapt prompts to the detected tech stack. Reference `SPEC.md`/`PLAN.md` if they exist.

```markdown
---
name: frontend-dev
description: SvelteKit frontend specialist using Svelte 5 runes.
tools: Read, Write, Edit, Bash, Glob, Grep
model: sonnet
skills:
  - svelte5-development
memory: project
---
You are a senior SvelteKit developer specializing in Svelte 5 with runes ($state, $derived, $effect, $props).

[... project-specific prompt ...]

Read SPEC.md for product requirements and PLAN.md for architecture decisions before starting any work.
```
