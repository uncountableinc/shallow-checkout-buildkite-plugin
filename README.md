# shallow-checkout-buildkite-plugin

Makes a Buildkite hosted agent clone the repository with `--depth=1` and without the per-host git
mirror.

## Why

Hosted agents keep a full mirror of the repository on each machine. A machine that has never built
the repository first clones the whole mirror, which takes 3 to 13 minutes for `uncountableinc/main`.
The `checkout:` block in pipeline YAML cannot change this: hosted agents ignore checkout settings
from pipeline YAML ("protected environment variables"). Hooks and plugins are allowed to set them,
so this plugin does it from an `environment` hook, which runs before the checkout.

With the plugin, a checkout on a fresh machine takes about 20 seconds: one shallow clone of the
tree, then a shallow fetch of the commit.

## Usage

Pin the plugin to a commit:

```yaml
steps:
  - label: "Lint"
    plugins:
      - https://github.com/uncountableinc/shallow-checkout-buildkite-plugin.git#<commit sha>
    command: just lint ruff
```

## Do not use it for jobs that need history

The clone has one commit. `git log`, `git merge-base`, `git diff <base>...HEAD`, and
`git describe` will not work. Fetch what you need explicitly, for example
`git fetch --depth=2 origin refs/pull/<n>/merge` for a pull request's changed files.

## What it sets

| Variable | Value |
| --- | --- |
| `BUILDKITE_GIT_MIRRORS_PATH` | empty, which disables the mirror |
| `BUILDKITE_GIT_CLONE_FLAGS` | `-v --depth=1` |
| `BUILDKITE_GIT_FETCH_FLAGS` | `-v --prune --depth=1` |
