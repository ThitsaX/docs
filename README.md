# ThitsaWorks Technical Documentation

This repository contains the source documentation for the ThitsaWorks technical documentation site.

The published documentation is intended to be read from:

https://docs.thitsaworks.com

This README is for maintainers who update the documentation source. It is not the main user-facing documentation page. The user-facing home page is maintained in [docs/index.md](docs/index.md).

## Purpose

The documentation explains how Mojaloop-based systems and PM4ML deployments are implemented, secured, deployed, and operated.

The current focus is:

- Platform architecture
- Hub and PM4ML security architecture
- Mojaloop and PM4ML deployment guides
- Pre-deployment readiness checks
- Operational and reliability documentation as the site grows

## Repository Layout

| Path | Purpose |
| ---- | ------- |
| `mkdocs.yml` | MkDocs site configuration and navigation |
| `docs/index.md` | Published site home page |
| `docs/platform-architecture-overview.md` | Platform architecture overview |
| `docs/hub-pm4ml-security-architecture.md` | Hub and PM4ML security architecture |
| `docs/deploy-*.md` | Deployment guides |
| `docs/pre-deployment-checklist.md` | Deployment readiness checklist |
| `docs/assets/` | Site images, logo, and favicon |

## Published Site

The canonical reading experience is the MkDocs site:

```text
https://docs.thitsaworks.com
```

Use this repository to edit the source files. Use the published site to share documentation with engineers, integration teams, DFSPs, and platform operators.

## Updating Documentation

When adding or changing a page:

1. Add or update the Markdown file under `docs/`.
2. Update `mkdocs.yml` so the page appears in the navigation.
3. Update `docs/index.md` when the page should be visible from the home page.
4. Keep examples practical and environment-aware.
5. Do not commit real secrets, private keys, customer credentials, or sensitive endpoint details.

Good documentation in this repo should explain both:

- How to perform the task.
- Why the step matters operationally.

This is especially important for deployment, security, certificate, backup, monitoring, and incident-response material.

## Local Preview

If MkDocs is installed locally, preview the site with:

```bash
mkdocs serve
```

Then open:

```text
http://127.0.0.1:8000
```

To build the static site:

```bash
mkdocs build --strict
```

If MkDocs is not installed, install the project documentation toolchain used by the team before running the preview or build commands.

## Documentation Standards

Use clear, implementation-driven writing.

Preferred style:

- Practical steps over theory-only descriptions
- Tables for checks, ownership, ports, and environment requirements
- Commands that operators can run directly
- Short rationale sections that explain why a check matters
- Explicit assumptions, scope, and out-of-scope notes

Avoid:

- Unverified production claims
- Placeholder values that look like real environment data
- Sensitive customer-specific details
- Secrets or private keys in examples
- Navigation entries that point to missing pages

## Current Main Sections

- Home
- Architecture
- Security
- Deployment

The Operations and Reliability area is being expanded with practical runbooks, checklists, and validation procedures.

## Maintainer Checklist

Before finishing a documentation change:

```bash
git diff --check
mkdocs build --strict
```

If the full MkDocs build cannot be run locally, at minimum confirm:

- The Markdown file exists under `docs/`.
- The page is included in `mkdocs.yml` when it should be visible in navigation.
- Links use valid relative paths.
- No real secrets or private environment values were added.
