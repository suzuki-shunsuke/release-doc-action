# release-doc-action

[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/suzuki-shunsuke/release-doc-action)

Reusable workflows to test and release documents built by `npm run build` (e.g. [Docusaurus](https://docusaurus.io/)) to GitHub Pages.

- [test.yaml](.github/workflows/test.yaml): Build the document to test it in pull requests
- [release.yaml](.github/workflows/release.yaml): Build the document and push it to the GitHub Pages branch

> [!WARNING]
> The GitHub Action [action.yaml](action.yaml) is deprecated. Please use the reusable workflows instead.
> The action installs npm packages, builds the document, and pushes it in one job, so the code of the repository and its dependencies runs with `contents: write` and `id-token: write`.

## Security

The workflows separate jobs by permissions so that the code of the repository and npm packages doesn't run with write permissions.

1. npm (`contents: read`, `id-token: write`): Install npm packages through [Takumi Guard](https://github.com/flatt-security/setup-takumi-guard-npm) without install scripts using [download-npm-package-action](https://github.com/suzuki-shunsuke/download-npm-package-action)
1. build (`contents: read`): Restore `node_modules`, run install scripts, and run `npm run build`
1. publish (`contents: write`, release.yaml only): Push the built files to the GitHub Pages branch. This job doesn't run the code of the repository and npm packages. Files such as `.git` and symbolic links in the built files are rejected
1. notify (`issues: write`, release.yaml only): Reopen or close the issue to notify the result

## Usage

### test.yaml

```yaml
jobs:
  test-doc:
    uses: suzuki-shunsuke/release-doc-action/.github/workflows/test.yaml@<sha> # <version>
    permissions:
      pull-requests: read
      contents: read
      id-token: write
    with:
      # Optional. The document is built only if the pull request changes these files
      paths: |
        website/**
        .github/workflows/workflow_call_test.yaml
    secrets:
      TAKUMI_GUARD_BOT_ID: ${{secrets.TAKUMI_GUARD_BOT_ID}}
```

#### Inputs

- `repository`: The repository (owner/repo) where the document exists. The default is the repository where the workflow runs. Only public repositories are supported
- `ref`: The git ref to build. By default, the default ref of actions/checkout is used, which is the merge commit on `pull_request` events
- `working_directory`: The directory where package.json and package-lock.json exist. The default is `website`
- `node-version-file`: The default is `website/.node-version`
- `paths`: Newline-separated glob patterns of files. On `pull_request` events, the document is built only if the pull request changes any of these files. The default is `website/**`. If empty, the document is always built. Set this to empty if the document exists in another repository

### release.yaml

```yaml
on:
  push:
    branches: [main]
    paths:
      - website/**
      - .github/workflows/deploy_doc.yaml
jobs:
  release-doc:
    uses: suzuki-shunsuke/release-doc-action/.github/workflows/release.yaml@<sha> # <version>
    permissions:
      contents: write
      issues: write
      id-token: write
    with:
      issue_number: 1724 # Optional
    secrets:
      TAKUMI_GUARD_BOT_ID: ${{secrets.TAKUMI_GUARD_BOT_ID}}
```

#### Inputs

- `repository`, `ref`, `working_directory`, `node-version-file`: Same as test.yaml
- `build_dir`: The directory where the built files are output. The default is `website/build`
- `publish_branch`: The branch where the built files are pushed. The default is `gh-pages`
- `destination_dir`: The directory in `publish_branch` where the built files are pushed. The default is `docs`
- `issue_number`: The issue number to notify the result. The issue is reopened if the workflow fails, and is closed if it succeeds
- `dry_run`: If this is true, the built files aren't pushed and the result isn't notified. This is used to test the workflow

### Release a document in another repository

If the document exists in another repository, set `repository`.
For example, the document of [aquaproj/aqua](https://github.com/aquaproj/aqua) is published from [aquaproj.github.io](https://github.com/aquaproj/aquaproj.github.io):

```yaml
jobs:
  test-doc:
    if: github.event_name == 'pull_request'
    uses: suzuki-shunsuke/release-doc-action/.github/workflows/test.yaml@<sha> # <version>
    permissions:
      pull-requests: read
      contents: read
      id-token: write
    with:
      repository: aquaproj/aqua
      paths: "" # Changed files of the pull request are irrelevant
    secrets:
      TAKUMI_GUARD_BOT_ID: ${{secrets.TAKUMI_GUARD_BOT_ID}}

  release-doc:
    if: github.event_name != 'pull_request'
    uses: suzuki-shunsuke/release-doc-action/.github/workflows/release.yaml@<sha> # <version>
    permissions:
      contents: write
      issues: write
      id-token: write
    with:
      repository: aquaproj/aqua
      issue_number: 1661
    secrets:
      TAKUMI_GUARD_BOT_ID: ${{secrets.TAKUMI_GUARD_BOT_ID}}
```
