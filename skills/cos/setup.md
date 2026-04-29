# Setup

Setup and environment bootstrap for cocli. The agent reads this file when preflight reports a non-OK status.

## Install

Detect the user's OS and architecture, then install cocli.

**Supported platforms:** macOS (amd64, arm64), Linux (amd64, arm64). No Windows native binary — use WSL.

| Method | When to use | Command |
|---|---|---|
| curl (recommended) | macOS or Linux, no Go toolchain | See below |
| go install | Go toolchain available | `go install github.com/coscene-io/cocli@latest` |

### curl install

Single command — downloads, decompresses, installs to `/usr/local/bin`:

```bash
curl -fsSL "https://download.coscene.cn/cocli/latest/$(uname -s | tr A-Z a-z)-$(uname -m | sed 's/x86_64/amd64/').gz" | gzip -d > cocli && chmod +x cocli && sudo mv cocli /usr/local/bin/
```

If `sudo` is unavailable or the user prefers a local install:

```bash
mkdir -p ~/.local/bin
curl -fsSL "https://download.coscene.cn/cocli/latest/$(uname -s | tr A-Z a-z)-$(uname -m | sed 's/x86_64/amd64/').gz" | gzip -d > ~/.local/bin/cocli && chmod +x ~/.local/bin/cocli
```

Ensure `~/.local/bin` is in `$PATH`.

**AskUserQuestion:** "Which install method do you prefer? (recommend curl unless you have Go set up)"

Post-install verify:

```bash
cocli --version
```

Expected output: `cocli version vX.Y.Z` (release) or `cocli version v0.0.0+<commit>.dirty` (dev build).

## Update

For outdated versions or manual upgrade:

```bash
cocli update
```

`cocli update` replaces the binary in-place via go-selfupdate. If cocli was installed to `/usr/local/bin` with sudo, the update also requires sudo:

```bash
sudo cocli update
```

To install a specific version instead:

```bash
curl -fsSL "https://download.coscene.cn/cocli/v1.2.3/$(uname -s | tr A-Z a-z)-$(uname -m | sed 's/x86_64/amd64/').gz" | gzip -d > cocli && chmod +x cocli && sudo mv cocli /usr/local/bin/
```

## Profile Bootstrap

First-time auth setup. Requires a coScene bearer token and project slug.

### Gather credentials

**AskUserQuestion:** "What is your coScene bearer token? (Find it in the platform UI: Settings > API Keys)"

**AskUserQuestion:** "What is your default project slug? (Format: org-slug/project-slug)"

**AskUserQuestion:** "What is your coScene endpoint? (Default: https://openapi.coscene.cn — most users keep the default)"

### Add profile

Execute with token hygiene — leading space prevents the token from entering shell history:

```bash
HISTCONTROL=ignorespace
 cocli login add -n default -t '<bearer-token>' -p '<project-slug>' -e '<endpoint>'
```

The leading space before `cocli` is intentional. With `HISTCONTROL=ignorespace`, commands starting with a space are excluded from shell history, keeping the bearer token out of `~/.bash_history` / `~/.zsh_history`.

### Verify

```bash
cocli login current
cocli project list -o json
```

Both should succeed without errors. If `project list` fails with a permission error, the token may lack project access — check with the organization admin.

### Secure config file

```bash
chmod 0600 ~/.cocli.yaml
```

`~/.cocli.yaml` stores bearer tokens in plaintext. Restrict read access to the owner.

## Preferences

Agent communication preferences — optional but recommended. Stored at `~/.coscene/agent-prefs.md`.

**AskUserQuestion:** "Preferred language? (zh / en / auto-detect) — recommend auto-detect"

**AskUserQuestion:** "Communication style? (concise / explainer / beginner) — recommend concise for developers, explainer for new users"

Write preferences:

```bash
mkdir -p ~/.coscene
cat > ~/.coscene/agent-prefs.md << 'PREFS'
# coScene agent preferences

## Language
<user-choice>

## Communication style
<user-choice>

## Last updated
<YYYY-MM-DD>
PREFS
```

Replace `<user-choice>` placeholders with the user's answers and `<YYYY-MM-DD>` with the current date.

If `mkdir` or write fails (permissions, disk full), warn the user and continue — preferences are optional. The CLI works without them.

### Prefs file format

Both the `cos` and `coscene-docs` skills read this file at first interaction via `cat ~/.coscene/agent-prefs.md 2>/dev/null`. The sections are parsed by heading:

| Section | Values | Default (if missing) |
|---|---|---|
| `## Language` | `zh`, `en`, `auto-detect` | `en` |
| `## Communication style` | `concise`, `explainer`, `beginner` | `concise` |
| `## Last updated` | ISO 8601 date | — |

## Help

### Platform concepts and troubleshooting

Load the `coscene-docs` skill, then read `troubleshooting.md` for diagnosis guides covering auth failures, upload issues, action execution problems, and device connectivity.

### Access troubleshooting

If preflight returned `ACCESS_DENIED`:

1. Check that the token has not expired — regenerate in Platform UI > Settings > API Keys
2. Verify the project slug matches an accessible project — `cocli project list -o json`
3. Check organization membership with the admin

### Human support

For issues beyond what the agent or documentation can resolve: contact@coscene.io
