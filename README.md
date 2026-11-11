# abyme publish

Publish an abyme cell from your GitHub repository to abyme. Anyone can then set it up as
`hosted:<account>/<cell>#<range>`, if its visibility lets them.

This repository holds only the workflow. abyme gives a push token only to a run of this
workflow, and only from a repository that an owner of the account trusts to publish its cells.

## Before your first publish

1. Have an abyme account: your own, or an org you own.
2. An owner of that account trusts your repository to publish its cells, on the Platform pane at
   auth.abyme.dev.
3. Your cell's `package.json` has `abyme.id` and `version`, and a lockfile (`pnpm-lock.yaml` or
   `package-lock.json`) sits at the top of your repository.

## Your workflow

Add `.github/workflows/publish.yml` to your repository:

```yaml
name: publish
on:
  push:
    tags: ["v*"]
  workflow_dispatch:
    inputs:
      action:
        type: choice
        options: [dry-run, yank]
      version:
        description: The version to yank (x.y.z)
        type: string
        required: false
jobs:
  publish:
    permissions:
      contents: read
      id-token: write
    uses: abyme-studio/publish/.github/workflows/publish.yml@v1
    with:
      account: your-handle
      action: ${{ inputs.action || 'publish' }}
      version: ${{ inputs.version }}
```

Give `folder` when the cell is not at the top of your repository, and `cell` when its name in
your account is not its `abyme.id`.

## Publishing a version

Bump `version` in `package.json`, then push a signed tag of that version:

```sh
git tag -s v1.2.0 -m "What changed in 1.2.0"
git push origin v1.2.0
```

The tag's message becomes the version's notes. GitHub must verify its signature, so the key you sign
with is on your GitHub account. The run builds the cell, commits the files `npm pack` would ship,
and tags that commit `v1.2.0`, signed by whoever pushed the tag. A version never moves: publishing
the same one again fails.

Run the workflow by hand with `dry-run` to build and list what would ship. Nothing is pushed.

## Yanking a version

Run the workflow by hand with `yank` and the version. That version is marked yanked, and a new
range no longer picks it. Nothing is deleted, so a profile that pinned it still starts.

## What abyme refuses

A public cell of any account but abyme ships no page code (a page half, styles, a library, or a
contract's local code): its UI is records views it seeds and OpenUI it writes, which abyme draws.
abyme refuses such a publish, `403 page_code`; a private or org cell may ship page code, which only
its own account's profiles run.

A new version breaks nothing callers of the version before it send or read, unless it is a new
major: a removed function, a new required argument, a removed field. An addition needs at least a
new minor. abyme refuses any other, `409 breaking_change`, naming each change. Each version's
contract is kept on the cell's `store` branch for the next version's check.

## What runs where

Your install and build run in the job `build`, which has no OIDC token; so does
`publish-tools.mjs`, which reads what your built cell ships to a page and runs its built halves
against stand-ins for its contract. The job `push` runs none of your code. It checks the tag is
signed, trades the run's OIDC token, with the version, its page code and its contract, for a push
token to that one cell for 10 minutes, and pushes. The workflow holds no secret.

## Licence

MIT, as abyme is: [LICENSE](LICENSE).
