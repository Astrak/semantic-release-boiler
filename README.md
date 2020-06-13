## semantic-release-boiler

This repo illustrates how to implement the [semantic-release](https://github.com/semantic-release/semantic-release) tools with a Github Action CI as of June 2020. The setup used has `master` as release branch, generates a `CHANGELOG.md` and may publish on NPM (not verified).

Here are the steps I followed:

### Steps
1. [Implement `commitizen`](#implement-commitizen)
1. [Hook it in `git commit`](#hook-commitizen-into-git-commit)
1. [Vet the commits format with `commitlint`](#vet-the-commit-messages-with-commitlint)
1. [Automate commitlint on CI (Github Action)](#have-the-commit-name-verification-in-a-github-action)
1. [Install `semantic-release`](#set-up-semantic-release)
1. [Run `semantic-release` on CI (Github Action)](#automate-release-on-ci)

#### Implement commitizen
[commitizen](https://github.com/commitizen/cz-cli) is a CLI tool triggered on git's `prepare-commit-msg` hook. It helps a team share a common commit format that will be used to identify when to release and to write a changelog.
`npm i -D commitizen cz-conventional-changelog`
Then setup a default configuration with
`commitizen init cz-conventional-changelog --save-dev --save-exact`

#### Hook commitizen into `git commit`
Now, although all the executables are there for us to run the commitizen tools on our latest changes, we want to force every contributor to use and prevent git softwares to miss it too. We can do so by calling commitizen in the `prepare-commit-msg` hook. But this config will not be stored in the repo though. We'll add an additional package to store git configs in a `/hooks` folder:
`npm i -D shared-git-hooks`
Now create a file `/hooks/prepare-commit-msg` that reads:
```bash
#!/bin/bash
exec < /dev/tty && node_modules/.bin/git-cz --hook || true
```
And make it executable: `chmod +x ./hooks/prepare-commit-msg`
Now `git commit` opens the commitizen CLI that formats the messages.

#### Vet the commit messages with commitlint
While commitizen is good, we want to vet that everyone uses it and not let non formatted commit messages through. Otherwise we might miss a release or lose track of changes in the changelog. We can do so with [commitlint](https://github.com/conventional-changelog/commitlint).
`npm i -D @commitlint/cli @commitlint/config-conventional`
Now the last commit message can be verified with
`git log -1 --pretty=format:"%s" | npx commitlint`

#### Have the commit name verification in a Github Action
We can automate the verification by running the script above in a Github Action running on PRs. Problem though, PRs can have multiple commits. One could be marked as failed and the next as successful, while we want to check all the commit messages of a PR. Details cut: if your target branch is `master` and if you never push commits directly to it, then this action is for you (create it in `./.github/workflows/action.yml`)
```yml
name: commitlint

on:
  - pull_request

jobs:
  pr-commitlint:
    runs-on: ubuntu-latest
    if: github.ref != 'refs/heads/master'
    steps:
      - uses: actions/checkout@v2
        with:
          ref: ${{ github.event.pull_request.head.sha }}
      - uses: actions/setup-node@v1
        with:
          node-version: "12.13.0"
      - run: git fetch
      - run: npm i @commitlint/{cli,config-conventional}
      - name: Debug PR commit messages passed to the linter
        run: git log origin/${{ github.event.pull_request.base.ref }}.. --pretty=format:"%s"
      - name: Vet PR commit messages
        run: git log origin/${{ github.event.pull_request.base.ref }}.. --pretty=format:"%s" | npx commitlint
```
#### Set up `semantic-release`
We're all set! Let's finally add the `semantic-release` packages on top of this all. The following setup will later handle versioning on Github, publish a zip file and generate a `CHANGELOG.md`:
`npm i -D semantic-release @semantic-release/git @semantic-release/changelog @semantic-release/release-notes-generator`
Add a `release` key in the `package.json`:
```json
{
  "release": {
    "branches": [
      "master"
    ],
    "plugins": [
      "@semantic-release/commit-analyzer",
      "@semantic-release/release-notes-generator",
      [
        "@semantic-release/changelog",
        {
          "changelogFile": "CHANGELOG.md"
        }
      ],
      [
        "@semantic-release/git",
        {
          "assets": [
            "CHANGELOG.md"
          ]
        }
      ]
    ]
  }
}
```
#### Automate release on CI
Now we want to run `semantic-release` in a Github Action, happening on `push` to our release branch (here `master`):
```yml
name: release

on:
  - push

jobs:
  release:
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/master'
    steps:
      - uses: actions/checkout@v1
      - uses: actions/setup-node@v1
        with:
          node-version: "12.13.0"
      - run: npm i -g yarn
      - run: yarn
      - name: Release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
        run: npx semantic-release
```
