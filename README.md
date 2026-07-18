# charlotte-zhuang/skills

Claude Code skills. Each skill lives under `skills/<skill-name>/SKILL.md`.

## Installing

```text
/plugin marketplace add charlotte-zhuang/skills
```

```text
/plugin install skills@charlotte-zhuang
```

After install, the skills become available in any Claude Code session - no per-repo setup needed.

To update later:

```text
/plugin marketplace update charlotte-zhuang
```

```text
/reload-plugins
```

## Skills in this plugin

- [**/private-fork**](skills/private-fork/SKILL.md) - Set up a private fork of a public git repo: redirect `origin` to a private remote and keep the public source as a fetch-only `upstream`.
- [**/hermes-tweet**](skills/hermes-tweet/SKILL.md) - Install and operate Hermes Tweet for Hermes Agent X/Twitter search, reads, and gated actions.
