# shallow-checkout-buildkite-plugin

Replaces the Buildkite agent's checkout with a single shallow fetch of the commit being built:
no git mirror, no history.

## Why

Hosted agents keep a full mirror of the repository on each machine. A machine that has never built
the repository first clones the whole mirror, which takes 3 to 13 minutes for `uncountableinc/main`.
Neither pipeline YAML (`checkout:` is ignored on hosted agents) nor hooks can turn the mirror off:
`BUILDKITE_GIT_MIRRORS_PATH` "cannot be modified from within a job, hook, or plugin". A plugin
`checkout` hook, however, overrides the default checkout routine entirely, so the agent never
touches the mirror.

With the plugin, a checkout on a fresh machine is one `git fetch --depth=1` of the commit plus a
checkout of the tree, about 20 seconds.

## Usage

Pin the plugin to a commit:

```yaml
steps:
  - label: "Lint"
    plugins:
      - https://github.com/uncountableinc/shallow-checkout-buildkite-plugin.git#<commit sha>
    command: just lint ruff
```

## What the hook does

1. `git init` in `BUILDKITE_BUILD_CHECKOUT_PATH` and add `BUILDKITE_REPO` as `origin`.
2. `git fetch --depth=1 origin <commit>`, or the branch when the build was started without a commit.
3. `git checkout -f FETCH_HEAD`, then `git clean -ffxdq`.

Credentials come from the agent's git credential helper, the same one the default checkout uses.

## Do not use it for jobs that need history

The clone has one commit. `git log`, `git merge-base`, `git diff <base>...HEAD`, and
`git describe` will not work. Fetch what you need explicitly, for example
`git fetch --depth=2 origin refs/pull/<n>/merge` for a pull request's changed files.
