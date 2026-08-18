# Echoder — Claude Code plugins

The public marketplace for [Echoder](https://echoder.dev) Claude Code plugins.

> **This repository is generated.** Its contents are published from the Echoder
> source repository on each release. Edits made directly here are overwritten, so
> please open issues upstream rather than pull requests against this repo.

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
