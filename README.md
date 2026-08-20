# coScene Skills

Agent Skills for [coScene](https://coscene.io/) — robotics data platform automation via Claude Code, Cursor, and other [Agent Skills](https://github.com/anthropics/skills)–compatible runtimes.

## Install

Copy-paste this prompt into Claude Code:

```
Clone https://github.com/coscene-io/coscene-skills to ~/coscene-skills (skip if already exists), then symlink each skill directory under skills/ into ~/.claude/skills/ so they're available as slash commands. Verify by listing ~/.claude/skills/cocli, ~/.claude/skills/coscene-docs, and ~/.claude/skills/git-annex-coscene.
```

Or manually:

```bash
git clone https://github.com/coscene-io/coscene-skills.git ~/coscene-skills
mkdir -p ~/.claude/skills
ln -sf ~/coscene-skills/skills/cocli ~/.claude/skills/cocli
ln -sf ~/coscene-skills/skills/coscene-docs ~/.claude/skills/coscene-docs
ln -sf ~/coscene-skills/skills/git-annex-coscene ~/.claude/skills/git-annex-coscene
```

The symlinks are still needed because this repository contains multiple skills
under `skills/`; each runtime skill directory must point directly at a folder
that contains its own `SKILL.md`.

## Skills

- **`cocli`** — CLI interface to the coScene platform via [`cocli`](https://github.com/coscene-io/cocli). Upload, query, process, move.
- **`coscene-docs`** — Platform concepts, automation model, and troubleshooting grounded in official docs.
- **`git-annex-coscene`** — Operator recipe for the `git-annex-remote-coscene` special remote: a coScene project as git-annex content storage.

## License

Apache 2.0 — see [`LICENSE`](./LICENSE).
