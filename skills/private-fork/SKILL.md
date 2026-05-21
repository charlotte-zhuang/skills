---
name: private-fork
description: Set up a private fork of a public git repo.
argument-hint: "[public-url] [private-url] [target-dir]"
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

Note: these are placeholder names used in this doc; don't literally `export PUBLIC_URL=...` in the shell.

## Workflow

### 0. Resolve all arguments

Do this before touching any git state.

**Public URL and mode.**

- If the user provided a public URL, make sure it is correctly formatted and set `PUBLIC_URL` to it.
  - If the URL is not valid and it would be trivial to correct (e.g. `github.com/<org>/<name>` -> `git@github.com:<org>/<name>.git`), correct the URL without asking the user.
  - Prefer the SSH form if no protocol is given.
  - Do not correct typos, infer the git host, or infer `<org>/<name>` without asking the user.
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

- If the user provided a private URL, make sure it is correctly formatted and set `PRIVATE_URL` to it.
  - If the URL is not valid and it would be trivial to correct (e.g. `github.com/<org>/<name>` -> `git@github.com:<org>/<name>.git`), correct the URL without asking the user.
  - Prefer the SSH form if no protocol is given.
  - Do not correct typos, infer the git host, or infer `<org>/<name>` without asking the user.
- Otherwise, ask the user which they want:
  1. **Paste an existing URL** — ask for it.
  2. **Create a new private repo** — confirm the git host (e.g. GitHub), repo name (default to the public repo's basename), and the owner (defaults to the user's personal GitHub account; for an org, use `<org>/<name>`). Then for GitHub:

     First, confirm `gh` is installed and authenticated before asking for any more inputs:

     ```bash
     gh auth status
     ```

     If that fails, say so plainly and ask the user to either run `gh auth login` themselves or to create the repo at <https://github.com/new> and paste the URL back. Don't proceed to `gh repo create` until auth is sorted — otherwise the user invests in confirming details only to hit an auth error.

     Once auth is confirmed:

     ```bash
     gh repo create <owner>/<name> --private --description "Private fork of <PUBLIC_URL>"
     ```

     `gh` prints the new repo URL on success.

     Steps will differ for other git hosts. Use other skills or online documentation; don't guess.

**Target directory.**

- Ignore if `MODE = in-place`.
- If the user didn't provide a target directory, try using the repo name from `PRIVATE_URL`.
- Double check that `TARGET_DIR` doesn't exist or is empty. Flag to the user that `TARGET_DIR` already exists otherwise.

Restate the resolved values back to the user before continuing — something like "OK, I'll clone `<PUBLIC_URL>` into `<TARGET_DIR>` and set the push destination to `<PRIVATE_URL>`." This is the last safe checkpoint before mutating anything.

### 1. Get into the right working directory

**If `MODE = clone`:**

```bash
git clone <PUBLIC_URL> <TARGET_DIR>
cd <TARGET_DIR>
```

**If `MODE = in-place`:** no action — you're already in the right directory.

### 2. Inspect the current remotes before touching anything

```bash
git remote -v
```

Walk through these checks and surface anything surprising to the user _before_ changing config:

- If `origin` already points at `PRIVATE_URL`, the swap is essentially done. Confirm with the user and skip ahead to step 6 (verify) — don't try to re-rename. After verification, you may need to set `upstream` to `PUBLIC_URL` if it was not set already.
- If an `upstream` remote already exists, surface its URL. The user may have done part of this setup before; merging your changes with what's already there is safer than overwriting.
- If `origin` doesn't match `PUBLIC_URL`, double-check with the user before proceeding — Step 0 should have caught this, but URLs can differ trivially (SSH vs HTTPS) and worth confirming.

**Check for Git LFS.** If the repo uses LFS and you don't prep it now, the push in step 5 will leave LFS objects on the public server and the private repo will have dangling pointers.

```bash
git lfs ls-files | head -n 1
```

If that prints anything — or `.gitattributes` contains `filter=lfs`, or a `.lfsconfig` file exists at the repo root — LFS is in use. Then:

1. Make sure LFS is available locally (no-op if already installed; safe to re-run):

   ```bash
   git lfs install
   ```

2. Pull LFS objects for every ref, not just HEAD — otherwise objects reachable only from non-default branches or tags won't be on disk to re-upload:

   ```bash
   git lfs fetch --all
   ```

3. Check for a `.lfsconfig` file at the repo root:

   ```bash
   cat .lfsconfig 2>/dev/null
   ```

   If present, it may pin the LFS endpoint to the public repo's URL — meaning even after the remote swap, LFS traffic still hits the public LFS server. Surface this to the user and let them pick:
   - Delete `.lfsconfig` if there's no project-specific LFS config worth keeping (LFS will then default to `<origin>.git/info/lfs`).
   - Edit it to point at the private repo's LFS endpoint, then commit the change.
   - Override per-command with `-c lfs.url=<url>` (least durable; only useful for one-off pushes).

   Don't silently delete or edit it — the user may have put it there for a reason.

After this prep, the normal push in step 5 uploads LFS objects automatically via the pre-push hook.

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

### 5. Push to the private origin

Default behavior: push only the default branch (whatever `HEAD` is on after the clone — usually `main` or `master`), with tracking set, and no tags. Don't mirror other branches or tags unless the user explicitly asked for them — most public repos carry stale topic branches and release tags that the user doesn't want copied into their private fork.

```bash
git push -u origin HEAD
```

Only if the user explicitly asked for more:

- **Other branches.** Push them by name (no `-u`, so tracking stays on the default branch):

  ```bash
  git push origin <branch1> <branch2> ...
  ```

  Or, if they asked for every branch:

  ```bash
  git push origin --all
  ```

- **Tags.**

  ```bash
  git push origin --tags
  ```

Read the push output carefully — when pushing multiple refs, git keeps going after a per-ref failure, so a single rejected branch or tag can hide in the middle of an otherwise-green push. If any ref is missing or rejected, treat it as a failure and resolve it before moving on.

If the repo uses LFS, the pre-push hook uploads objects for the refs being pushed. Objects reachable only from refs you didn't push won't be uploaded — fine if those refs aren't being mirrored. If the user did opt to push everything, you can re-run as an idempotent sanity check:

```bash
git lfs push origin --all
```

Note: `-u` sets tracking only on the branch you push it with. If `HEAD` is detached, skip `-u` and tell the user to check out a branch first, then re-run `git push -u origin <branch>` to set tracking.

If the private repo already has commits (e.g., GitHub auto-created a README when the repo was made), the push will fail with a non-fast-forward error. **Stop and ask the user** — don't silently force-push. The two reasonable options are:

- Have the user delete and recreate the private repo empty (cleanest), then retry.
- Force-push with `git push -u origin HEAD --force-with-lease` if the user confirms the private repo has nothing worth keeping.

`--force-with-lease` is safer than `--force` because it refuses to overwrite remote commits you haven't seen — but it's still a destructive operation on the remote, so get explicit confirmation.

### 6. Verify the result

Check both the remote config and that branches are actually tracking the new `origin`:

```bash
git remote -v
git branch -vv
```

The expected `git remote -v` output looks like:

```
origin    git@github.com:org/private-repo.git (fetch)
origin    git@github.com:org/private-repo.git (push)
upstream  git@github.com:org/public-repo.git (fetch)
upstream  DISABLED (push)
```

`git branch -vv` should show the default branch tracking `origin/<branch>` (not `upstream/<branch>`). Other local branches may still track `upstream` since we didn't push them — that's expected unless the user opted to mirror them. If a ref the user explicitly asked to push is missing on the private side, go back and resolve before telling the user the setup is done.

Then briefly tell the user what changed in their day-to-day workflow:

- `git push` and `git pull` go to the private repo.
- `git fetch upstream` pulls new commits from the public source; merge or rebase them in deliberately.
- `git push upstream …` will fail by design. To contribute back to the public repo, open a PR there the normal way (fork on GitHub, push to your personal fork, PR).

## Edge cases worth handling carefully

- **Uncommitted changes in the working tree.** If the user is already inside a repo and has dirty state, the remote swap itself is safe (it only touches `.git/config`), but flag it so they don't conflate "private fork set up" with "my work is pushed somewhere." Show `git status` if anything is staged or modified.
- **Submodules.** This skill doesn't rewrite `.gitmodules`. If `git submodule status` shows submodules, mention to the user that submodule URLs still point at their original locations and they may want to redirect those separately.

$ARGUMENTS
