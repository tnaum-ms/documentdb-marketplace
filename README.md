# DocumentDB marketplace (prototype)

A GitHub Copilot plugin marketplace with experimental DocumentDB tools. This repository contains
build output only.

| Plugin | What it does |
| --- | --- |
| `documentdb-explain-plan` | **DocumentDB Query Insights** canvas: paste DocumentDB or MongoDB API explain output into chat and get a stage visualization, plus explanations and optimization advice from Copilot. See [its README](plugins/documentdb-explain-plan/README.md). |

These plugins are prototypes: unsupported, and they may change or be removed.

## Install in the GitHub Copilot app

1. Open **Customize → Plugins** and add a marketplace with this repository's URL:
   `https://github.com/tnaum-ms/documentdb-marketplace`
2. Install **documentdb-explain-plan** from the `documentdb-marketplace` marketplace.
3. Start a new chat and paste the output of `.explain("executionStats")`.

## Install with the GitHub Copilot CLI

```shell
copilot plugin marketplace add tnaum-ms/documentdb-marketplace
copilot plugin install documentdb-explain-plan@documentdb-marketplace
```

## Updates

Refresh the marketplace and update the plugin in the app, then start a new chat. With the CLI:

```shell
copilot plugin marketplace update documentdb-marketplace
copilot plugin update documentdb-explain-plan
```

The panel footer shows the installed build ID.
