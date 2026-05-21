# charlotte-zhuang/skills

Claude Code skills. Each skill lives under `skills/<skill-name>/SKILL.md`.

## Installing

```text
/plugin marketplace add charlotte-zhuang/skills
/plugin install skills@charlotte-zhuang
```

After install, the skills become available in any Claude Code session — no per-repo setup needed.

To update later:

```text
/plugin marketplace update charlotte-zhuang
```

## Skills in this plugin

- **private-fork** — Set up a private fork of a public git repo: redirect `origin` to a private remote and keep the public source as a fetch-only `upstream`. Invoke with `/private-fork [private-url] [public-url] [target-dir]` or trigger by phrasing like "make a private fork of …".

## Adding a new skill

1. Create `skills/<new-skill>/SKILL.md` with YAML frontmatter. See the [Claude Code Docs](https://code.claude.com/docs/en/skills) for all features.
2. Add a bullet for it under "Skills in this plugin" above.
3. Commit and push. Users pick it up via `/plugin marketplace update c`.
