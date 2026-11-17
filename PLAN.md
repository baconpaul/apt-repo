# apt-infra plan

An https apt repository whose files live as assets on GitHub releases. Each
release is one *channel* (today just `distro`), described by one config file.

## How it works

Apt's flat repository format is a single directory with no `dists/` or `pool/`,
which is exactly what a release's asset list is. A machine adds:

```
# /etc/apt/sources.list.d/baconpaul.sources
Types: deb
URIs: https://github.com/baconpaul/apt-repo/releases/download/distro/
Suites: ./
Signed-By: /etc/apt/keyrings/baconpaul-apt-repo.asc
```

and apt fetches `InRelease`, `Packages.gz` and the `.deb`s from the release
(GitHub 302-redirects to its object store; apt follows https redirects).

Each release holds:

- every `.deb` the channel config lists, named `{pkg}_{version}_{arch}.deb`
- `Packages`, `Packages.gz` with bare-filename `Filename:` entries
- `Release`, `InRelease` (clearsigned), `Release.gpg` (detached)
- the public key, so users can fetch it from the same place

## Layout

```
channels/<name>.json     one per channel / release tag
public_key.asc           committed
private_key.asc          gitignored
keys/                    gitignored, other archives' public keys for verify_sources
scripts/
  aptlib.py              shared code (config, deb parsing, versions, index, signing)
  create-keys            one-time ed25519 signing key pair
  add-debs               debs (paths or URLs) -> channel config, from their control fields
  create-pkg-info        channel config + debs -> build/<channel>/ (signed)
  verify                 debian trixie in docker: add repo, update, fetch, install
  update-release         build/<channel>/ -> the GitHub release      (step 3)
content/                 gitignored, local debs
cache/                   gitignored, downloaded debs
build/                   gitignored, staged output per channel
tests/                   local end-to-end test with fake debs and a throwaway key
.github/workflows/       (step 5)
```

## Channel config

```json
{
  "channel": "distro",
  "github_repo": "baconpaul/apt-repo",
  "origin": "baconpaul",
  "label": "baconpaul apt-repo",
  "description": "baconpaul plugins and games for sndstation and debian",
  "architectures": ["arm64", "amd64"],
  "public_key_asset": "baconpaul-apt-repo.asc",
  "verify_sources": [
    { "name": "sndstation", "uris": "https://sndstation.s3.us-west-1.amazonaws.com",
      "suites": "stable", "components": "main",
      "key": "keys/sndstation-archive-keyring.asc", "architectures": ["arm64"] }
  ],
  "packages": [
    {
      "name": "leeward",
      "versions": [
        { "version": "1.1.0-1", "debs": ["content/leeward_1.1.0-1_arm64.deb"] }
      ]
    }
  ]
}
```

- `channel` is the release tag and the apt suite. Several configs, several releases, one repo.
- Prior versions stay listed so `apt install leeward=1.0.0-1` works; apt picks the newest by default.
- A deb is a local path (relative to the repo root) or an http(s) URL. `add-debs` copies
  URL debs into `content/` by default, since nightly URLs are replaced by the next build.
- Each deb's control `Package`, `Version` and `Architecture` must match the config
  (`Architecture: all` is allowed); mismatches fail the build.
- `verify_sources` are apt sources the packages depend on that a real machine has
  (sndstation's own, for its games). Only `verify` uses them; nothing from them is published.

## Key

ed25519, signing only, 2 year expiry, no passphrase (an optional one is read from
`APT_SIGNING_PASSPHRASE`). Trixie's apt verifies with sqv, which handles ed25519.
One key signs every channel. Signing happens in a throwaway `GNUPGHOME`; your
personal keyring is never touched. CI gets the private key from the
`APT_SIGNING_KEY` secret.

Expiry breaks every client's `apt update`, so extend it before then
(`gpg --quick-set-expire`) and re-export.

## update-release (step 3)

1. create the release for `channel` if missing; list its assets
2. upload debs not already present (same name and size)
3. upload index files, `InRelease` last (`--clobber` is delete-then-upload, so order
   keeps the inconsistent window tiny)
4. delete debs no longer referenced
5. `--dry-run` prints the plan only
6. a deb missing from `content/` is fetched back from the release, so a fresh
   checkout or CI can rebuild without the original URLs

## Steps

1. done: `create-keys`, channel config schema
2. done: `create-pkg-info` + `verify` against a local http server, arm64 and amd64 trixie
3. `update-release` to the `distro` tag, `verify --remote`
4. README install instructions
5. `workflow_dispatch` workflow (choose channel, build, publish)
6. later: product CI dispatches new versions by URL to `add-debs`
