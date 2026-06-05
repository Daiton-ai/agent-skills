# Contributing

Thanks for helping connect more agents to the Daiton brain.

## Shape of this repo

- **`using-the-daiton-brain/SKILL.md`** is the single source of truth — the reusable playbook. Keep platform-specific detail OUT of it; it must stay portable across every host.
- **`install/<platform>.md`** files are thin wrappers. They cover *packaging and connection only* and defer to the core skill for behavior. Don't duplicate the read/write rules into an install file — link to the core skill instead.

## Adding a new platform

1. Copy the closest existing `install/*.md` as a template.
2. Fill in only what's platform-specific: where the MCP config lives, which auth mode the host supports (OAuth/DCR vs `dai_` bearer token), and where the skill/instructions attach.
3. Use the real config filenames and the server URL `https://mcp.daiton.ai/mcp/`. **Ground every platform claim** in that platform's actual, documented config mechanics — don't invent features.
4. Add a row to the install table in `README.md`.

## Editing the core skill

The core skill follows the Agent Skills `SKILL.md` convention: YAML frontmatter (`name`, `description`) then the body. The `description` is the trigger — keep it about *when to use* the skill, with concrete symptoms. Preserve the two load-bearing rules:

- **Provenance is mandatory for `assert_fact`** — never document a way to fabricate or placeholder a source.
- **IDs are always surfaced as labeled, copyable pills** — never bury a raw id in prose.

## Conventions

- Markdown, wrapped sensibly, fenced code blocks for every config snippet.
- Don't commit real `dai_` tokens or secrets. Use placeholders (`dai_xxxx`) and runtime-prompt patterns.
- Reference the repo as `Daiton-ai/agent-skills` consistently, so a future rename stays a clean find/replace.
