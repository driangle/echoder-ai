# Echoder — Claude Code plugins

The public marketplace for [Echoder](https://echoder.dev) Claude Code plugins.

> **This repository is generated.** It is published from the Echoder monorepo by
> the `publish-plugin` workflow on each `plugin-v*` tag. Open issues and pull
> requests against the monorepo, not here — edits made directly to this repo are
> overwritten on the next release.

## Install

```
/plugin marketplace add driangle/echoder-ai
/plugin install echoder@echoder-marketplace
/reload-plugins
```

## Plugins

| Plugin | What it does |
| --- | --- |
| `echoder` | Write, run, and verify Echoder sketches — generative video and audio. See [`claude-code-plugin/README.md`](./claude-code-plugin/README.md). |

## Requires

The `echoder` CLI, which the skill uses to look up DSL signatures and to run and
render sketches:

```bash
brew install driangle/tap/echoder
```
