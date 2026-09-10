# shallow-checkout-buildkite-plugin

Replaces the Buildkite agent's checkout with a single shallow fetch of the commit being built:
no git mirror, no history.

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

On hosted agents credentials come from the agent's git credential helper, the same one the default
checkout uses. On self-hosted agents the agent applies `checkout.ssh_secret` only to its own
checkout routine, so pass the secret to the plugin instead:

```yaml
    plugins:
      - https://github.com/uncountableinc/shallow-checkout-buildkite-plugin.git#<commit sha>:
          ssh_secret: NAME_OF_BUILDKITE_SECRET
```

The `environment` hook reads the key with `buildkite-agent secret get` into a temporary file and
exports `GIT_SSH_COMMAND` pointing at it, so the checkout and every later `git` call in the job
(for example fetching a pull request merge ref) authenticate the same way. The `pre-exit` hook
deletes the file. `GIT_SSH_COMMAND` runs `ssh` with `LD_LIBRARY_PATH` unset, so a Nix or devenv
shell that prepends its own libraries cannot break the host's `ssh`. For an SSH `github.com`
repository the checkout hook also adds GitHub's published host keys (from
`https://api.github.com/meta`) to `~/.ssh/known_hosts` first, because the default checkout does
that and a plugin checkout hook replaces it.

## Do not use it for jobs that need history

The clone has one commit. `git log`, `git merge-base`, `git diff <base>...HEAD`, and
`git describe` will not work. Fetch what you need explicitly, for example
`git fetch --depth=2 origin refs/pull/<n>/merge` for a pull request's changed files.
