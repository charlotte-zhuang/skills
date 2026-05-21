---
name: private-fork
description: Set up a private fork of a public git repo.
argument-hint: "[private-url] [public-url] [target-dir]"
when_to_use: Use this skill whenever the user wants to create a private fork, private mirror, internal copy, or vendored copy of a public repository — phrases like "make a private fork", "mirror this repo to a private one", "set up a private copy of <repo>", "redirect this clone to my org's private GitHub", or when they hand over a private repo URL and ask to point this repo (or a public one) at it for pushing. Also trigger when they're in a public repo and mention wanting to push their work somewhere private instead.
---

# Private fork setup

GitHub's "Fork" button can't fork a public repo into a private one — you have to do it manually with a clone + remote swap. This skill automates that flow: take a public repo (either already cloned locally, or to be cloned fresh) and rewire its remotes so `origin` points at a new private repo for pushes, while the original public source sticks around as a fetch-only `upstream`.

## Inputs the skill needs

By the time the git-mutating steps run, three things must be known:

- `PUBLIC_URL` — the source of truth being mirrored from.
- `PRIVATE_URL` — the new push destination.
- `MODE` — either `clone` (we'll `git clone <PUBLIC_URL>` fresh) or `in-place` (the current directory is already a clone of `PUBLIC_URL`).
- `TARGET_DIR` — optional folder name passed to `git clone` in `clone` mode; ignored otherwise.

Step 0 below resolves all of these so later steps can assume they're valid. The user may give arguments in any order or use ambiguous terms like "upstream" or "new origin" — if anything is unclear about which URL is which, ask before proceeding. Clobbering the wrong remote is hard to recover from.

Note: do not set these as environment variables. These are simply names that we will refer to in the workflow below to avoid ambiguity.

## Workflow

### 0. Resolve all arguments

Do this before touching any git state. After this step, `PUBLIC_URL`, `PRIVATE_URL`, and `MODE` are all set.

**Public URL and mode.**

- If the user provided a public URL, set `PUBLIC_URL` to it and `MODE = clone`.
- Otherwise, check whether the current directory is a git working tree:

  ```bash
  git rev-parse --is-inside-work-tree 2>/dev/null
  ```

  If yes, read its `origin` and use that:

  ```bash
  git remote get-url origin
  ```

  Set `PUBLIC_URL` to that value and `MODE = in-place`. Show the URL to the user and confirm it really is the public source they want to mirror (the existing `origin` might already be a private repo or some unrelated remote).

- If neither path worked (no URL given and not in a git repo), ask the user for the public URL, then set `MODE = clone`.

**Private URL.**

- If the user provided one, use it directly. Done.
- Otherwise, ask the user which they want:
  1. **Paste an existing URL** — ask for it.
  2. **Create a new private repo** — confirm the git host (e.g. GitHub), repo name (default to the public repo's basename), and the owner (defaults to the user's personal GitHub account; for an org, use `<org>/<name>`). Then for GitHub:

     ```bash
     gh repo create <owner>/<name> --private --description "Private fork of <PUBLIC_URL>"
     ```

     `gh` prints the new repo URL on success. Prefer the SSH form (`git@github.com:<owner>/<name>.git`) unless the public source's URL is HTTPS, in which case match that convention so auth stays consistent.

     If `gh` isn't installed or isn't authenticated (`gh auth status` fails), say so plainly and ask the user to either run `gh auth login` themselves or to create the repo at <https://github.com/new> and paste the URL back.

     Steps will differ for other git hosts. Use other skills or online documentation; don't guess.

**Target directory.** Carry through unchanged if `MODE = clone`; ignore otherwise.

Restate the resolved values back to the user before continuing — something like "OK, I'll clone `<PUBLIC_URL>` into `<TARGET_DIR>` and set the push destination to `<PRIVATE_URL>`." This is the last safe checkpoint before mutating anything.

### 1. Get into the right working directory

**If `MODE = clone`:**

```bash
git clone <PUBLIC_URL> [<TARGET_DIR>]
cd <TARGET_DIR-or-derived-name>
```

Pass `TARGET_DIR` as the second arg if it was provided; otherwise omit it and let `git clone` derive the folder name from the URL.

**If `MODE = in-place`:** no action — you're already in the right directory.

### 2. Inspect the current remotes before touching anything

```bash
git remote -v
```

Walk through these checks and surface anything surprising to the user _before_ changing config:

- If `origin` already points at `PRIVATE_URL`, the swap is essentially done. Confirm with the user and skip ahead to step 6 (verify) — don't try to re-rename.
- If an `upstream` remote already exists, surface its URL. The user may have done part of this setup before; merging your changes with what's already there is safer than overwriting.
- If `origin` doesn't match `PUBLIC_URL`, double-check with the user before proceeding — Step 0 should have caught this, but URLs can differ trivially (SSH vs HTTPS) and worth confirming.

### 3. Rename `origin` to `upstream` and disable pushing to it

```bash
git remote rename origin upstream
git remote set-url --push upstream DISABLED
```

The second command is the standard idiom for making a remote fetch-only: `DISABLED` isn't a valid URL, so any future `git push upstream …` fails loudly instead of silently leaking changes back to the public repo. This is the whole reason the private fork pattern is safer than just adding a second remote — accidents become impossible, not just unlikely.

### 4. Add the private repo as the new `origin`

```bash
git remote add origin <private-url>
```

### 5. Push everything to the private origin

Push all branches and tags so the private repo is a complete mirror, and set tracking on the current branch so plain `git push` works going forward:

```bash
git push -u origin --all
git push origin --tags
```

If the private repo already has commits (e.g., GitHub auto-created a README when the repo was made), `--all` will fail with a non-fast-forward error. **Stop and ask the user** — don't silently force-push. The two reasonable options are:

- Have the user delete and recreate the private repo empty (cleanest), then retry.
- Force-push with `git push -u origin --all --force-with-lease` if the user confirms the private repo has nothing worth keeping.

`--force-with-lease` is safer than `--force` because it refuses to overwrite remote commits you haven't seen — but it's still a destructive operation on the remote, so get explicit confirmation.

### 6. Show the result

```bash
git remote -v
```

The expected final state looks like:

```
origin    git@github.com:org/private-repo.git (fetch)
origin    git@github.com:org/private-repo.git (push)
upstream  git@github.com:org/public-repo.git (fetch)
upstream  DISABLED (push)
```

Then briefly tell the user what changed in their day-to-day workflow:

- `git push` and `git pull` go to the private repo.
- `git fetch upstream` pulls new commits from the public source; merge or rebase them in deliberately.
- `git push upstream …` will fail by design. To contribute back to the public repo, open a PR there the normal way (fork on GitHub, push to your personal fork, PR).

## Edge cases worth handling carefully

- **Uncommitted changes in the working tree.** If the user is already inside a repo and has dirty state, the remote swap itself is safe (it only touches `.git/config`), but flag it so they don't conflate "private fork set up" with "my work is pushed somewhere." Show `git status` if anything is staged or modified.
- **Detached HEAD or non-default branch.** `git push -u origin --all` pushes every local branch regardless of which one is checked out, so this still works — but `-u` only sets tracking on the currently checked out branch. If the user is on a detached HEAD, skip the `-u` and tell them to check out a branch first.
- **SSH vs HTTPS URL mismatches.** If the user gives an HTTPS private URL but their existing remotes are SSH (or vice versa), don't try to be clever — use exactly the URL they gave. Different auth setups have real reasons.
- **Submodules.** This skill doesn't rewrite `.gitmodules`. If `git submodule status` shows submodules, mention to the user that submodule URLs still point at their original locations and they may want to redirect those separately.

$ARGUMENTS
