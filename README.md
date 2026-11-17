# baconpaul apt-repo

A signed apt repository for baconpaul plugins and games, for sndstation and
Debian (trixie or newer), on arm64 and amd64. The files live as assets on this
repo's GitHub releases; each release is one channel.

## Using it

The `distro` channel is the only one today.

```sh
sudo mkdir -p /etc/apt/keyrings
sudo curl -fsSL -o /etc/apt/keyrings/baconpaul-apt-repo.asc \
    https://github.com/baconpaul/apt-repo/releases/download/distro/baconpaul-apt-repo.asc

sudo tee /etc/apt/sources.list.d/baconpaul.sources >/dev/null <<'EOF'
Types: deb
URIs: https://github.com/baconpaul/apt-repo/releases/download/distro/
Suites: ./
Signed-By: /etc/apt/keyrings/baconpaul-apt-repo.asc
EOF

sudo apt update
sudo apt install six-sines
```

The key is ed25519, uid `baconpaul apt-repo signing key`, fingerprint

```
1205 EA4E F83A DEBE D7BA  7AE7 DC0A 6195 7EAA EC15
```

Check it with `gpg --show-keys /etc/apt/keyrings/baconpaul-apt-repo.asc`. It
expires in November 2028 and will be extended before then; a machine then needs
the refreshed key file, fetched the same way.

## Packages

| package     | what                                        | arches       |
|-------------|---------------------------------------------|--------------|
| six-sines   | Six Sines synth (CLAP, VST3, standalone)    | arm64, amd64 |
| leeward     | A Band Called Leeward, a sndstation game    | arm64        |
| bacon-beats | Bacon Beats, a sndstation app               | arm64        |

leeward and bacon-beats need the `sndstation` package, which comes from
sndstation's own source.

Nightlies are versioned `X.Y.Z~nightly.<date>.<count>.<hash>` (before the
X.Y.Z release) or `X.Y.Z+nightly...` (after it), so apt moves between nightlies
and releases in the right order on its own.

## Maintaining it

```sh
scripts/add-debs channels/distro.json <deb paths or URLs>   # update the config from the debs
scripts/create-pkg-info channels/distro.json                # stage and sign build/distro/
scripts/verify channels/distro.json                         # install them in trixie, in docker
```

`private_key.asc` is not in the repo. `PLAN.md` has the design and what is left.
