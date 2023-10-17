# Documentation for the Community Health Toolkit

[![CHT Documentation Site Build](https://github.com/medic/cht-docs/workflows/CHT%20Documentation%20Site%20Build/badge.svg)](https://github.com/medic/cht-docs/actions)

## About

This repo contains documentation for the Community Health Toolkit (CHT), and how to build digital health applications with [CHT Core](https://github.com/medic/cht-core). This repository powers the CHT documentation site at [docs.communityhealthtoolkit.org](https://docs.communityhealthtoolkit.org).

The documentation is built using Markdown pages, which can be converted into a navigable website using a static-site-generator. The Hugo static-site-generator is being used with the [Docsy theme](https://themes.gohugo.io/docsy/). To maintain portability, content should be written in plain Markdown with limited use of HTML, custom shortcodes, and modifications to the theme. Contributions should align with the [documentation style guide](https://docs.communityhealthtoolkit.org/contribute/docs/style-guide/).

## Build dependencies

The documentation site uses [Hugo](https://gohugo.io/), and specifically uses features found in the extended version of Hugo, v0.76.0 and later. See `.tool-versions` for the currently used version

[asdf](https://asdf-vm.com/guide/getting-started.html) is the recommended way to manage `hugo` and `golang` versions for local development.

After installing it run:
```shell
asdf plugin add golang
asdf plugin add hugo
```
### Installing Hugo Manually

If using `asdf` fails you can try installing Hugo manually. To install, follow the [installation instructions for your Operating System](https://gohugo.io/getting-started/installing/), and be sure to get the extended version. Most users will be able to simply install using their native package manager like `brew`, `apt` or `snap`.

## Building the Documentation

1. Clone the `cht-docs` repo
1. From the command line, navigate to the local directory where you cloned the repo
1. Type `hugo server`
1. Preview your site in your browser at: http://localhost:1313/
1. As you make changes to the site, your browser will automatically reload your local dev site so you can easily and quickly see your changes.

MacOS users getting errors when running `hugo server`, please see our [Troubleshooting guide](./troubleshooting.md).

## Link Checking [Optional]

We validate that all links on the docs site work (do not 404) using a tool called [Muffet](https://github.com/raviqqe/muffet) along with [Actions](https://github.com/features/actions). If you're creating a lot of new links, or editing a lot of existing links, you may optionally run Muffet locally before pushing your commits. Running Muffet locally can save time by exposing broken links before pushing a build since you can avoid waiting for the Action to run, finding you have a broken link, fixing it, and pushing a new change.

1. Install [Go](https://golang.org/doc/install) as a prerequisite 
2. Install Muffet: `go get -u github.com/raviqqe/muffet`
    - If using `asdf` you need to reshim (`asdf reshim golang`)
3. Ensure you've run `hugo server` so your local docs instance is reachable at http://localhost:1313/
4. Test the links with the [`muffet.sh`](https://github.com/medic/cht-docs/blob/main/.github/scripts/muffet.sh) script.  If you're in the root of this repo, that'd be: `./.github/scripts/muffet.sh` 
  
It should take about 60 seconds depending on your Internet connection. If Muffet returns no output, you have no broken links, congrats! 

_Note_: The `muffet.sh` script here is the identical script we run on GitHub. If you simply run `muffet http://localhost:1313` you will hit GitHub's rate limiting and get lots of [429's](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/429). Our script intentionally reduces concurrency and excludes some repetitive GitHub URLs.

## Continuous Deployment

All changes to `main` branch run a [GitHub action](.github/workflows/ci.yml) to first check for any broken links ([per above](#link-checking-optional)) and then deploy to the documentation site: [docs.communityhealthtoolkit.org](https://docs.communityhealthtoolkit.org)

## Upgrading dependencies

### Hugo

The version of Hugo used to deploy the site is specified in `.tool-versions`. 

_(Note there is also a minimum version of Hugo required to deploy the site specified via `module.hugoVersion` in the [`config.toml`](./config.toml) file.)_

See the [Hugo Release Notes](https://github.com/gohugoio/hugo/releases) for documentation regarding what has changed in the new versions.

### Docsy

The current version of Docsy is pinned in the [`go.mod`](./go.mod) file. To upgrade to a new version of Docsy, use the `hugo mod get` command as described [in the Docsy documentation](https://www.docsy.dev/docs/updating/updating-hugo-module/).

See the [Docsy CHANGELOG](https://github.com/google/docsy/blob/main/CHANGELOG.md) for documentation regarding what has changed in the new versions.
