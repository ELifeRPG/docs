# ELifeRPG Docs

Documentation site for the ELifeRPG project, built with [Zensical](https://zensical.org/).
Content lives under `docs/`; site configuration is in `zensical.yml`.

## Setup

Requires Python `>=3.13` and [uv](https://docs.astral.sh/uv/).

Install dependencies:

```bash
uv sync
```

### Devcontainer

A [devcontainer](https://containers.dev/) is provided so you don't need Python/uv installed
on your host:

- VS Code with the [Dev Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) extension: **Reopen in Container**.
- CLI: `devcontainer up --workspace-folder .`, then run commands with `devcontainer exec --workspace-folder . <command>`.

`uv sync` runs automatically on first create (`postCreateCommand`).

## Development

Start the dev server on `http://localhost:8000` (rebuilds and live-reloads on file changes):

```bash
uv run zensical serve -f zensical.yml
```

> The config file is named `zensical.yml`, which Zensical does not auto-discover (it only
> looks for `zensical.toml`, `mkdocs.yml`, or `mkdocs.yaml`) — every command needs the
> explicit `-f zensical.yml` flag.

To bind a different address or open a browser tab automatically:

```bash
uv run zensical serve -f zensical.yml -a 0.0.0.0:8000 --open
```

## Production build

Outputs static files to `site/`:

```bash
uv run zensical build -f zensical.yml
```

Deployment to GitHub Pages runs the same command in CI — see
`.github/workflows/docs.yml`.
