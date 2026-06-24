The repo's `.gitattributes` export-ignores itself, causing the git fetcher and
github fetcher to produce different hash producing different source trees for
the same rev:

| `6785394a3b156a8c85732df3064c5b2c70a5c291` | narHash                                               | `.gitattributes` in tree? |
| ------------------------------------------ | ----------------------------------------------------- | ------------------------- |
| `github` tarball (honors `export-ignore`)  | `sha256-VO8KzRYOgpVx36ZrJgyhKBIqs/LCSt75RVaCYxQv9TY=` | no                        |
| `git` checkout (ignores `export-ignore` )  | `sha256-87RQXrLzn8kt1/LIoJ3zHJlPIvOs6Jag7bVfYzRqNQg=` | yes                       |

The collision is not visible to users whenever the gh tarball's store path is
already realized or substitutable (because `Input::getAccessorUnchecked`
substitution fast path at `src/libfetchers/fetchers.cc:318`) then uses it
directly and "heals" the cache.

We need a isolated nix store (tarball path absent) with substitution disabled,
and a fresh fetcher cache. This is triggered almost every time when I update the
inputs in <https://github.com/stepbrobd/ysun> or
<https://github.com/stepbrobd/rfm> with dependabot, and pulls locally.

```bash
#!/usr/bin/env bash
set -euo pipefail

REPO=github:stepbrobd/nix-cache-hash-mismatch-mre
GIT=git+https://github.com/stepbrobd/nix-cache-hash-mismatch-mre
REV=6785394a3b156a8c85732df3064c5b2c70a5c291

# get github tarball narHash for $REV
# nix flake metadata --json "$REPO/$REV" | jq -r .locked.narHash
TARBALL_HASH='sha256-VO8KzRYOgpVx36ZrJgyhKBIqs/LCSt75RVaCYxQv9TY='

# new cache
TMP=$(cd "$(mktemp -d)" && pwd -P)
export XDG_CACHE_HOME="$TMP/cache"

NIX=(nix --extra-experimental-features "nix-command flakes" --store "$TMP/store" --option substituters "" --option substitute false)

# fetch rev via the git: (which includes .gitattributes)
"${NIX[@]}" flake metadata "$GIT?ref=refs/heads/master&rev=$REV" >/dev/null

# fetch the same rev with github: with the locked tarball narHash
"${NIX[@]}" flake metadata "$REPO/$REV?narHash=$TARBALL_HASH"

chmod -R u+w "$TMP"; rm -rf "$TMP"
```
