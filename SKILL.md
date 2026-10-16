---
name: agent-harness-setup
description: >-
  Framework-, tool-, and language-agnostic guide for setting up or extending an AI Coding & Code Review Agent Harness in any repository.
  Designed for projects that do not have an agent harness configured yet or want to extend existing rules into a complete harness.
  Automates AI workflows when starting fresh chats (chat-per-issue coding and actionable-only code reviews) so critical project rules
  are reliably followed even by faster, cheaper models.
---

# Agent Harness Setup & Architecture Skill

## Purpose

Use this skill on projects that **do not have an AI agent harness configured yet** or **want to upgrade scattered `.md` docs/rules into a deterministic harness**.

Its goal is to configure a repository so you can:
1. **Start a new empty chat per issue/ticket for coding** (even with faster, cheaper models like Gemini Flash or Haiku) where the agent immediately knows how to fetch the ticket, what conventions to follow, what *not* to do, and how to validate its work without wasting tool calls or running test suites twice.
2. **Start a new empty chat (or trigger a cloud reviewer) for code review** where the reviewer knows what to inspect, outputs **only actionable findings**, never wastes quota running test suites, and never conflicts with repository rules.
3. **Trap rule violations deterministically** via pre-push hooks and architecture/drift tests rather than hoping an LLM remembers every line of prose.

---

## Core Principles of a High-Signal Agent Harness

1. **Zero-Tool-Call Visibility for Core Rules (`AGENTS.md <= 24,000` bytes):**
   - Faster/cheaper models optimize to minimize tool calls. If core coding conventions, style guides, or translation rules live in separate `docs/*.md` files (or across 6+ tiny skills), cheaper models will skip reading them and write non-compliant code.
   - Inline all daily coding conventions, architectural invariants, and workflow rules directly into root `AGENTS.md` (which all modern agents inject into the system prompt on Turn 0).
   - Keep `AGENTS.md` under **`24,000` bytes** (`24 KB`):
     - **Tool-level limit:** AI IDEs/CLIs (such as Antigravity at 24 KB, GitHub Copilot, and OpenAI Codex at 32 KB) hard-truncate rule files past their byte cap, silently dropping rules at the bottom of the file.
     - **Model-level limit:** Keeping always-on instructions concise prevents attention dilution ("lost in the middle") on cheaper/faster models.
2. **Progressive Disclosure Only for Heavy Procedural Runbooks (`.agents/skills/`):**
   - Move bulky, domain-specific operational runbooks (e.g., server setup, VPS/Nginx/Docker configs, complex deployment troubleshooting) out of `AGENTS.md` and `docs/` into `.agents/skills/<skill-name>/SKILL.md`.
   - Include a **Project Skills Map** table in `AGENTS.md` with relative links so both local agents and cloud agents (GitHub Copilot, Claude, Jules) can locate them on demand.
3. **Codebase as the Single Source of Truth (Zero Schema/Prompt Duplication):**
   - Never hardcode database schemas, ORM model definitions, API payloads, or LLM prompts inside `AGENTS.md`, `README.md`, or skills once the code exists. They inevitably drift and teach AI agents to write outdated code.
   - State only the **architectural invariants** (e.g., encryption requirements, layer boundaries, DTO patterns, roll-forward migrations) and link directly to the live source directories/files.
4. **Never Call the Project an "MVP" in Agent-Facing Docs:**
   - Phrasing like *"MVP scope"* or *"adapted for MVP velocity"* in `README.md` or `AGENTS.md` biases LLMs toward cutting corners, skipping edge cases, and writing throwaway code. Always frame the project around **production-grade engineering standards**.
5. **Enforce Deterministically Whenever Possible:**
   - Any rule in `AGENTS.md` that can be checked programmatically belongs in a formatter, linter, static analyzer, or an **Architecture & Documentation Drift Test** (`ArchTest`) executed by the standard test runner.

---

## Step-by-Step Workflow to Build or Extend a Project Harness

### Step 1: Audit the Repository (Inspect First, Never Assume)

Before writing or editing any harness files, inspect the repository to discover:
1. **Tech Stack & Manifests:** Check `package.json`, `composer.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`, `*.csproj`, `Makefile`, etc. Never assume framework or dependency versions.
2. **Existing Validation Commands & Hooks:** Inspect `.husky/`, `.git/hooks/`, `lefthook.yml`, `.pre-commit-config.yaml`, and `.github/workflows/` to catalog existing formatters, linters, type checkers, static analyzers, and test runners.
3. **Existing Documentation & Rules:** Read `README.md`, `AGENTS.md`, `CLAUDE.md`, `.github/copilot-instructions.md`, `.github/instructions/`, `.agents/skills/`, and `docs/`.
4. **Identify Anti-Patterns & Drift:**
   - Flag any "MVP" language in `README.md` or `AGENTS.md`.
   - Flag any `docs/*.md` files that contradict the live codebase or duplicate schemas/models/routes.
   - Decide with the user which `docs/*.md` files should be absorbed into `AGENTS.md` (daily coding/testing/UX/i18n rules) vs. converted into `.agents/skills/<name>/SKILL.md` (heavy infrastructure/ops runbooks) and deleted from `docs/`.

---

### Step 2: Structure Root `AGENTS.md`

Organize `AGENTS.md` into clean, high-density sections (using short 1-line Good/Bad examples where syntax matters):

#### 1. Workflows: Issue-Per-Chat, Issue Tracker Tools & Code Review
- **Issue & PR Tooling (Chat-Per-Issue):**
  - Instruct the agent **never** to use generic web-page fetch tools (`read_url_content`, `WebFetch`) on issue tracker URLs (e.g., `github.com`).
  - Instruct it to immediately fetch issue/PR titles, bodies, labels, and comments using configured MCP tools (e.g., `github-mcp-server`: `issue_read`, `pull_request_read`) or CLI tools (`gh issue view <id> --comments`, `gh pr view <id> --comments`).
- **Branching & Parallel Session Safety:**
  - Never push directly to protected branches (`main` / `master`).
  - **Branch Safety Check:** Before creating or switching branches, always run `git status` and `git branch --show-current`. If there are uncommitted changes or the workspace is on an unmerged branch for a *different* issue, **stop and ask the user** before switching so you never overwrite another active session's work.
  - Define the branch naming convention (e.g., `<issue-number>-<short-slug>`, avoiding `/` if required by deployment/preview tooling).
- **Code Review Workflow (Local Branch & Cloud PR Reviews):**
  - When prompted with `"Review current branch"`, `"Review uncommitted changes"`, or `"Review PR #<id>"`:
    1. Consult `.agents/skills/code-review-skill/SKILL.md` (if installed) and `.github/instructions/pr-review.instructions.md` (if using GitHub Copilot).
    2. Inspect `git status` and `git diff <base>...HEAD` **without modifying any files** and **without running linters or test suites** (formatting, static analysis, and tests are enforced deterministically by pre-push hooks and CI).
    3. **Actionable-Only Output (Strict Rule):** Use severity labels `[blocking]`, `[important]`, `[suggestion]`, and `[nit]`. Output **only** findings that require an actual code, test, or documentation change (including actionable `[nit]` items) with file paths and line numbers. Do **not** output praise, introductions, general summaries, or self-cancelled observations (e.g., *"I considered X, but after checking closer no change is needed"*). If zero changes are needed, output a single sentence stating no actionable issues were found.
- **AI Behavior & Harness Maintenance:**
  - **Apply friction when appropriate:** Do not blindly execute requests that violate codebase conventions, security, architecture, workflows, or automation. State the concern concisely with reasoning and ask how to proceed.
  - **Prevent Rule Drift & Update Architecture Tests:** Rules across `AGENTS.md`, `README.md`, and `.agents/skills/` must never contradict each other or duplicate code schemas. Whenever a new deterministically verifiable rule is added to `AGENTS.md`, update the project's architecture/meta test suite.

#### 2. Project Skills Map (`.agents/skills/`)
- Include a Markdown table mapping domain tasks to `.agents/skills/<name>/SKILL.md` paths, noting that `AGENTS.md` rules and the project's existing design system/architecture always take precedence over third-party skills.

#### 3. Domain Architecture, Security & Data Invariants
- State non-negotiable architectural boundaries, security/encryption rules, transaction/atomicity rules, and migration policies, explicitly designating the live source folders (e.g., models, DTOs, services) as the source of truth.

#### 4. Language, Framework, UX & Localization Conventions
- Inline the project's specific coding conventions, type-hint/comment formatting rules, request/response validation patterns, UX guardrails, and i18n rules with concise `✅ Good` / `❌ Bad` snippets.

#### 5. Testing Strategy
- Require adding/updating tests for every new or modified feature.
- Specify how to write behavior-driven unit/integration/feature tests in the project's test frameworks (e.g., using Fakes over brittle mocks, querying UI by accessibility roles).

#### 6. Fast Pre-Push Validation Workflow
- **Avoid Running the Full Suite Twice:** If the project uses a `pre-push` hook that runs the full linter/type-check/test suite on `git push`, instruct the agent **not** to manually run the full suite right before `git push`.
- **Before `git commit` (Fast Local Loop):**
  1. Run **only the targeted test(s)** for the modified files.
  2. Run the fast **Auto-Fixers** (formatter/linter/codegen) *before* committing and stage their changes so the `pre-push` hook does not modify files mid-push and abort on an `Uncommitted Changes` check.
- **On `git push` (or when finishing locally without pushing):**
  - Let `git push` execute the `pre-push` hook once. If a check fails, run only that specific failing command to verify the fix, commit, and `git push` again.
  - Only run the full validation suite manually when finishing work locally *without* pushing.

---

### Step 3: Optional GitHub Copilot Reviewer Adapter (`.github/instructions/pr-review.instructions.md`)

> **When to include:** This file is specifically for **GitHub Copilot Code Review** (cloud PR reviewer). Other agents (Claude, Jules, Antigravity) do not read `.github/instructions/` by default unless pointed to it. If the project does not use GitHub Copilot Code Review, you can skip creating this file because the review workflow already lives in `AGENTS.md`.

If the project uses GitHub Copilot on GitHub:
1. Create `.github/instructions/pr-review.instructions.md` starting with **mandatory YAML frontmatter** (without `applyTo`, GitHub Copilot silently ignores files in `.github/instructions/`):
   ```yaml
   ---
   applyTo: "**"
   ---
   ```
2. **Do not duplicate coding rules from `AGENTS.md`** (Copilot Code Review reads root `AGENTS.md` from the base branch natively; duplicating rules causes drift).
3. Include only:
   - Top-level links to `.agents/skills/code-review-skill/SKILL.md` (if present), `AGENTS.md` (single source of truth), and `README.md`.
   - The **Actionable-Only Output Policy** (`[blocking]`, `[important]`, `[suggestion]`, `[nit]` only; no running test suites/linters; no praise/summaries/self-cancelled notes).
   - The **Project Skills Map** table.
   - A **Rule & Documentation Consistency Check** when a PR modifies `.md` or skill files.

---

### Step 4: Configure the 3-Phase `pre-push` Verification Hook (If Needed)

If the repository does not yet have a deterministic `pre-push` hook (or wants to upgrade it), structure it in 3 phases so errors are aggregated clearly instead of failing blindly:

1. **Phase 1 — Auto-Fixing (Style, Formatting & Codegen):** Run fast auto-fixers (`prettier --write`, `eslint --fix`, `pint`, `ruff format`, code generators, etc.).
2. **Phase 2 — Strict Validation (Types, Static Analysis & Tests):** Run type checks, translation/schema checks, static analyzers, and unit/feature test suites.
3. **Phase 3 — Working Tree Cleanliness Check:** Check `if [ -n "$(git status --porcelain)" ]; then ...` and fail with a clear message if Phase 1 auto-fixers modified files that weren't committed yet.

Ensure CI (`.github/workflows/`) runs the read-only equivalents (`--check` / `--test`) of Phase 1 and Phase 2.

---

### Step 5: Add Deterministic Architecture & Drift Tests (`ArchTest`)

Whenever possible, add a fast automated test file in the project's native test runner (e.g., Pest `ArchTest.php`, Vitest/Jest, `pytest`, Go test, Rust test) that enforces:

1. **Architectural & Security Invariants:** Layer boundaries, naming conventions (e.g., suffixes for DTOs/Enums), forbidden functions/imports, and critical security invariants (e.g., encryption casts or soft-delete traits on sensitive domain models).
2. **`AGENTS.md` Size Budget (`<= 24,000` bytes):**
   - Assert that `AGENTS.md` is $\le 24,000$ bytes.
   - Include a clear failure message and comment explaining both the **Hard IDE/CLI Parser Cap** (Antigravity 24 KB, Copilot/Codex 16–32 KB) and **LLM Attention Density** (preferring `ArchTest` or `.agents/skills/` over bloating `AGENTS.md`, while noting the threshold can be raised in the future if tool/model limits evolve).
3. **Documentation Link Integrity:** Parse local relative Markdown links `[label](path)` in `AGENTS.md`, `README.md`, and `.github/instructions/pr-review.instructions.md` and assert every referenced file exists on disk.
4. **Command & Script Drift Check:** Parse CLI script commands referenced in `AGENTS.md` (e.g., `pnpm run <script>`, `npm run <script>`, `composer <script>`, `make <target>`) and assert those scripts actually exist in the project's manifest (`package.json`, `composer.json`, `Makefile`, etc.).
