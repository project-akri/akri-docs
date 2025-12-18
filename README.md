# Akri Docs

This repo contains documentation for [Akri](https://github.com/project-akri/akri), including:

1. [Akri's user and developer docs](./docs), which are hosted Akri's docs site [docs.akri.sh](https://docs.akri.sh/) by
   GitBook
1. [Proposals](./proposals) for new Akri features
1. [Akri's logo](./art) artifacts

## Adding Documentation

To add documentation to Akri's GitBook documentation site, create a markdown file in the appropriate section folder
within `docs`.

Akri's documentation is divided into six sections:

1. :blue_book: User Guide: Documentation for Akri users.
1. :mag_right: Discovery Handlers: Documentation on how to configure Akri using Akri's currently supported Discovery Handlers
1. :rocket: Demos: End-to-End demos that demostrate how Akri can discover and use devices. Contain sample brokers and end applications.
1. :gear: Architecture: Documentation that details the design and implementation of Akri's components.
1. :computer: Development: Documentation for Akri developers or how to build, test, and extend Akri.
1. :tada: Community: Information on what's next for Akri and how to get involved!

Then, link to it in [`SUMMARY.md`](docs/SUMMARY.md) to ensure it is displayed on Akri's site.

Add associated images to the `media` folder and adhere to GitBook [Markdown syntax
requirements](https://docs.gitbook.com/editing-content/markdown).

## Creating a Proposal

To propose a design for a new Akri feature such as a new Discovery Handler, create a PR to add your document to the
[proposals](./proposals) folder. A proposal template and conventions for updating proposal statuses will be added soon.

## Other Documentation

### HackMD

If you have guides, scenarios, examples, etc. that you think would benefit from being a part of the Akri ecosystem but
do not fit on Akri's documentation site or as a proposal, they could be added to [Akri's
HackMD](https://hackmd.io/team/akri). Create an issue to request that a HackMD page be created for the content.

### Akri's Medium Blog

If you have a scenario or use-case you'd like to share more broadly, we are always looking for contributions to [Akri's
Medium Blog](https://medium.com/akri).

> The Linux Foundation has registered trademarks and uses trademarks. For a list of trademarks of The Linux Foundation, please see our [Trademark Usage page](https://www.linuxfoundation.org/legal/trademark-usage)

