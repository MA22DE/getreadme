# getreadme

Download only the **README** of GitHub repositories and GitHub **star lists** —
one folder per repository, without cloning anything.

Driven entirely by the [GitHub CLI](https://cli.github.com); no API keys, no
tokens to manage, and incremental re-runs that skip anything unchanged.

## Requirements

- [`gh`](https://cli.github.com), authenticated (`gh auth login`). Required for
  private repositories, star lists and reliable error messages.
- `git` or `shasum` for change detection. Without either, the tool still works,
  it just re-downloads every README instead of skipping unchanged ones.

## Install

```sh
git clone https://github.com/MA22DE/getreadme.git
ln -s "$PWD/getreadme/getreadme" ~/.local/bin/getreadme   # or copy it anywhere on PATH
```

## Usage

```sh
getreadme https://github.com/OWNER/REPO                 # one repository
getreadme https://github.com/stars/USER/lists/SLUG      # one star list
getreadme https://github.com/stars/USER                 # every starred repository
```

Several sources can be given in one run, and flags may appear anywhere:

```sh
getreadme https://github.com/vercel/next.js https://github.com/stars/USER/lists/ai-tools -flat
```

| Flag | Effect |
|---|---|
| `-flat`, `--flat` | Skip the star list / `USER-stars` folder (see layout below) |
| `-list`, `--list` | Accepted for convenience; star list URLs are recognised by their shape |
| `-h`, `--help` | Show help |

### Output layout

READMEs are written relative to the current directory, one folder per repository:

| Source | Written to |
|---|---|
| `github.com/OWNER/REPO` | `./OWNER/REPO/README.md` |
| `.../stars/USER/lists/SLUG` | `./SLUG/OWNER/REPO/README.md` |
| `.../stars/USER` | `./USER-stars/OWNER/REPO/README.md` |
| any of the above with `-flat` | `./OWNER/REPO/README.md` |

Because the owner is part of the path, two repositories with the same name never
overwrite each other.

## Incremental re-runs

Run the same command again and nothing is downloaded again unless something
actually changed:

- **unchanged README** → the local file is not touched
- **changed README on GitHub** → replaced
- **repository added to a list** → fetched automatically
- **README edited or deleted locally** → restored to the GitHub version
- **repository removed from a list** → its folder is left in place, never deleted

This uses no state file. GitHub's ETag for a README is the Git blob hash of the
file, so the hash of the local copy is sent back as `If-None-Match`: GitHub
answers `304 Not Modified` and nothing is transferred.

## API cost

Use of the GitHub API is **free** — the limits below are abuse-prevention
rate limits, not a billing meter. There is nothing to enable and no payment
method involved.

| Operation | Cost against the hourly limit |
|---|---|
| Download a README (`200`) | 1 REST request |
| Re-check an unchanged README (`304`) | **0 — not counted** |
| Repository/repository-README missing (`404`) | 2 REST requests |
| Enumerate a star list | ~1 GraphQL point per 100 entries |
| Enumerate all stars | 1 REST request per 100 stars |

Authenticated budgets are **5,000 REST requests/hour** and **5,000 GraphQL
points/hour** (separate pools, reset hourly). Examples:

- Daily refresh of a star list with 50 entries: 50 README requests are sent
  but all answer `304`, so **none of them count** — the only counted cost is the
  single list query (1 GraphQL point).
- First import of 1,203 stars: `13 + 1,203 = 1,216` requests ≈ 25% of the
  hourly REST budget; every run after that costs only the 13 enumeration
  requests as long as nothing changed.

Other GitHub features can cost money (Actions minutes, Packages, Codespaces,
Copilot, LFS storage); none of them are touched by this tool.

## Exit codes

| Code | Meaning |
|---|---|
| `0` | Everything processed — unchanged files count as success |
| `1` | At least one repository or source failed (missing repo, no README, auth) |
| `2` | Usage error (no URL, unknown option) |

## Notes and limitations

- The README is always saved as `README.md`, even if the repository calls it
  `README.rst` or `readme.txt`; the bytes are always the original file.
- `gh` is asked for the raw README, so `.md` content is not reformatted.
- Star lists are a GraphQL-only feature of the GitHub API; the REST API has no
  lists endpoint, which is why this script uses both APIs.
- Tested on macOS with the system bash 3.2 (no bash 4+ features are used).

## Legacy version

The pre-star-list version (188 lines, single-repository URLs only, writing to
`./REPO/README.md`) is not kept as a file — it lives in the repository
history, where the first commit `3cba618` holds it byte for byte:

```sh
# browse it
gh repo view MA22DE/getreadme --branch 3cba618 --web
git show 3cba618:getreadme

# restore it next to the current script
git show 3cba618:getreadme > getreadme.old && chmod +x getreadme.old
```