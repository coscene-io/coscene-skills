---
name: git-annex-coscene
description: >
  Use when storing or retrieving git-annex content in a coScene project via the
  git-annex-remote-coscene special remote — setting up initremote, enabling the
  remote for teammates, verifying stored objects, or debugging annex transfers
  against the coScene OpenAPI gateway. Pairs with cocli (platform CLI ops) and
  coscene-docs (platform concepts).
---

# git-annex-coscene

> Use a coScene project as git-annex content storage. Annex objects land as
> files of a designated record (filename = annex key); transfers go through
> platform APIs with platform RBAC — no client-side S3 credentials.

Authoritative reference (protocol behavior, requirements, e2e): the tool doc
in the starbase repo, `docs/cli/git-annex-remote.md`. This skill is the
operator recipe.

## Prerequisites

- `git-annex` ≥ 10.x on PATH.
- `git-annex-remote-coscene` on PATH (git-annex invokes it by name). Build
  from starbase: `make -C cli build-annex-remote` → `cli/bin/git-annex-remote-coscene`.
- Credentials, resolved outside git (never written to the repo or `remote.log`):
  1. `COSCENE_TOKEN` env var, else
  2. the `~/.cocli.yaml` profile whose `endpoint` equals the remote's `url=`.

## Set up the remote

```sh
git annex initremote coscene type=external externaltype=coscene \
    encryption=none chunk=0 autoenable=true \
    url=https://openapi.<domain> project=<project-slug-or-id> record=<record-title-or-id>
```

- `url=` is the **OpenAPI** endpoint (`https://openapi.<domain>`), not the org
  web domain — e.g. `https://openapi.volc.coscene.cn` for `volc.coscene.cn`.
- `record=` by title creates the record if absent; the remote prints the
  created resource name:
  `created record "annex-store" (projects/<uuid>/records/<uuid>)`.

**Pin the record id right after the record exists** (teams especially: title
resolution races on `initremote` can create duplicate records, after which the
remote refuses with `multiple records titled …`):

```sh
git annex enableremote coscene record=<record-uuid>
```

## Teammate flow

With `autoenable=true`, a teammate needs only credentials:

```sh
git clone <repo> && cd <repo>
git annex init            # prints: (Auto enabling special remote coscene...)
git annex get <file> --from coscene
```

No `initremote` re-run, no per-user storage keys — platform RBAC on the
project governs access.

## Daily use

```sh
git annex add <file> && git commit -m '...'
git annex copy <file> --to coscene     # upload
git annex drop <file>                  # free local space (checks remote copy)
git annex get <file>                   # download back
git annex whereis <file>               # prints the platform web URL of the object
```

## Constraints

| Setting | Rule | Why |
|---|---|---|
| `chunk=0` | mandatory (default) | coScene dedups whole objects; chunk keys are refused by design |
| `encryption=none` | recommended | other modes work but store opaque `GPGHMAC*` names — no upload dedup, no size/sha cross-checks |
| `appendonly=yes` | optional, archive remotes | remote refuses `drop`/`REMOVE`: `this remote is configured appendonly=yes; refusing to remove <key>` |
| `mode=` | leave unset | reserved for later phases |

**Gotchas observed live:**

- **HTTP 402 `NO_SUBSCRIPTION` from hand-rolled `curl` probes is the protocol,
  not your token.** The gateway requires gRPC; plain Connect-JSON requests get
  `402 {"code": 8, "message": "NO_SUBSCRIPTION"}`. The remote speaks gRPC
  natively — do not debug a working remote from a failing curl.
- On tokens without `FileService/CloneFile` permission the remote prints one
  `INFO` (`transfer-level dedup is off`) and always uploads; server-side
  storage dedup still applies. Informational, not an error.
- Single presigned PUT per object: objects beyond the provider single-PUT
  limit (typically 5 GiB) are not supported in phase 1.

## Verify

```sh
git annex fsck <file> --from coscene   # downloads + hash-verifies remote content
git annex testremote coscene --fast    # protocol conformance against the live remote
```

Use `--fast`: the full matrix exercises chunksize variants, which this remote
refuses by design. In the platform web UI, the record's Files tab shows one
file per stored object, named by the annex key (`SHA256E-…`), matching the
URL `git annex whereis` prints.

Trust stance: keep the remote [semitrusted](https://git-annex.branchable.com/trust/)
(default) and `numcopies >= 2` for masters; run `fsck --from coscene`
routinely.
