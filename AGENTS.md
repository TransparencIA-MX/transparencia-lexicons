# Agent Instructions

This project uses **bd** (beads) for issue tracking. Run `bd onboard` to get started.

## Quick Reference

```bash
bd ready              # Find available work
bd show <id>          # View issue details
bd update <id> --claim  # Claim work atomically
bd close <id>         # Complete work
bd dolt push          # Push beads data to remote
```

## Non-Interactive Shell Commands

**ALWAYS use non-interactive flags** with file operations to avoid hanging on confirmation prompts.

Shell commands like `cp`, `mv`, and `rm` may be aliased to include `-i` (interactive) mode on some systems, causing the agent to hang indefinitely waiting for y/n input.

**Use these forms instead:**
```bash
# Force overwrite without prompting
cp -f source dest           # NOT: cp source dest
mv -f source dest           # NOT: mv source dest
rm -f file                  # NOT: rm file

# For recursive operations
rm -rf directory            # NOT: rm -r directory
cp -rf source dest          # NOT: cp -r source dest
```

**Other commands that may prompt:**
- `scp` - use `-o BatchMode=yes` for non-interactive
- `ssh` - use `-o BatchMode=yes` to fail instead of prompting
- `apt-get` - use `-y` flag
- `brew` - use `HOMEBREW_NO_AUTO_UPDATE=1` env var

## Lexicon Schemas

This repository holds the AT Protocol Lexicon JSON files under `lexicons/tech/transparencia/`. See `README.md` for the full field tables.

**Important schema notes for agents editing news/article records:**

- `description` is a **short summary only** (RSS `<description>`, Atom `<summary>`). It must NOT contain the article body; the body lives in `content`. News fetchers / mappers that previously fell back to `<content:encoded>` for `description` must stop doing so.
- `content` (new in v2) carries the full HTML body when the feed exposes one (`<content:encoded>`, Atom `<content>`, JSON Feed `content_html`/`content_text`).
- `tags[]` (new in v2) holds every category the feed exposed (RSS `<category>`+, `dc:subject`, Atom `<category>`+, `news:keywords`).
- `feedCategory` is **DEPRECATED**. It now equals `tags[0]` and is kept only for back-compat with v1 consumers; it will be removed in a future revision.
- `updatedAt`, `originalSource` (`{name?, url?}`), and `mediaCaption` are optional v2 fields that should be populated whenever the feed exposes them, but absent (not `null`) when it does not.

When extending the lexicons, prefer optional fields with strict `maxLength` over required additions — they keep older records valid without backfill.

## For any AI coding agent (Claude, Gemini, Cursor, Aider, …)

This file is **model-agnostic** — same rules apply regardless of which agent edits the repo.

- **Conversation language: Spanish** (maintainer is in Mexico). Commits are in English using conventional commits (`feat:`, `fix:`, `docs:`).
- This repo is the **single source of truth** for `tech.transparencia.*` ATProto Lexicon schemas. Downstream repos (`news_fetcher`, `transparencia-indexer`) consume these via submodule / generated code; any schema change cascades.
- **Schema changes are breaking by default.** Use `goat lex breaking` (when available) before any PR. Mark new fields as optional; new required fields demand a migration plan for existing records.
- The maintainer's GitHub account (`JoKradept`) has read-only access here — open PRs from `JoKradept/transparencia-lexicons` fork. The project lead (Diego) merges.
- After merging a schema change, downstream repos need to **bump their `lexicons` submodule pointer** to pick it up. Don't assume the bump is automatic.
- Sister repos: `TransparencIA-MX/news_fetcher` (ingester that produces records) and `TransparencIA-MX/transparencia-indexer` (Tap + GraphQL that consumes them).

<!-- BEGIN BEADS INTEGRATION v:1 profile:minimal hash:ca08a54f -->
## Beads Issue Tracker

This project uses **bd (beads)** for issue tracking. Run `bd prime` to see full workflow context and commands.

### Quick Reference

```bash
bd ready              # Find available work
bd show <id>          # View issue details
bd update <id> --claim  # Claim work
bd close <id>         # Complete work
```

### Rules

- Use `bd` for ALL task tracking — do NOT use TodoWrite, TaskCreate, or markdown TODO lists
- Run `bd prime` for detailed command reference and session close protocol
- Use `bd remember` for persistent knowledge — do NOT use MEMORY.md files

## Session Completion

**When ending a work session**, you MUST complete ALL steps below. Work is NOT complete until `git push` succeeds.

**MANDATORY WORKFLOW:**

1. **File issues for remaining work** - Create issues for anything that needs follow-up
2. **Run quality gates** (if code changed) - Tests, linters, builds
3. **Update issue status** - Close finished work, update in-progress items
4. **PUSH TO REMOTE** - This is MANDATORY:
   ```bash
   git pull --rebase
   bd dolt push
   git push
   git status  # MUST show "up to date with origin"
   ```
5. **Clean up** - Clear stashes, prune remote branches
6. **Verify** - All changes committed AND pushed
7. **Hand off** - Provide context for next session

**CRITICAL RULES:**
- Work is NOT complete until `git push` succeeds
- NEVER stop before pushing - that leaves work stranded locally
- NEVER say "ready to push when you are" - YOU must push
- If push fails, resolve and retry until it succeeds
<!-- END BEADS INTEGRATION -->
