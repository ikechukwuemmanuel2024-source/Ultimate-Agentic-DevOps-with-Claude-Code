# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Static HTML/CSS portfolio website for the **DevOps Micro Internship (DMI)** teaching program. The site itself is plain HTML5 + CSS3 with no build step. This repo also carries an agentic DevOps toolkit (Claude Code skills) for provisioning AWS hosting via Terraform and deploying via GitHub Actions — the automation is as much the point of this repo as the site it ships.

## Site Content

`index.html` is a **single page** whose sections are reached by in-page anchor navigation, not separate routes:

- `#home` (hero) · `#about` · `#services` · `#courses` · `#book` · `#community` · `#contact` — see the `<section id="…">` blocks in `index.html`
- **style.css** — all styling for `index.html` (~17 KB), mobile-first responsive. `index.html` has no inline styles; put site styling here.
- **privacy.html / terms.html** — standalone pages that use **inline `<style>`**, not `style.css`. Edit their styling inside each file, not in `style.css`.
- **images/** — static assets referenced by relative path from `index.html` (`logo.png`, `profile.jpg`, `signature.png`, and course/topic thumbnails). Keep filenames stable when replacing an image, or update every reference.

To preview, open `index.html` directly in a browser — there is no build or dev server.

### Known gap: the page expects JavaScript that isn't in the repo

`index.html` calls `goToSection(...)` and `toggleMenu()` from `onclick` handlers (nav buttons, logo, hamburger) and has an empty `<span id="year">` in the footer, but **no `<script>` tag or `.js` file exists** — so section navigation, the mobile menu, and the copyright year are currently inert. If you're asked to "make the nav work" or add interactivity, this is the missing piece: define those functions (inline `<script>` before `</body>`, or a new JS file). Note the deploy excludes only cover `*.md` and tooling dirs (see below), so **a new `.js` file at the repo root WILL be synced live** — that's expected, just be deliberate about what lands in the site root.

### DMI ownership footer — do not strip attribution

The footer (`index.html:604`) reads `Crafted with <span>cloud</span> excellence by Pravin Mishra`. Per the DMI exercise (see README.md), students add a `Deployed by:` proof line to the footer rather than replacing this attribution. When editing the footer, preserve the existing credit line unless the user explicitly asks to change it.

## Deployment Architecture

The live deployment (see `.github/workflows/deploy.yml`) is **AWS S3 + CloudFront**, not the Nginx-on-a-VM flow described in README.md. The README documents a separate manual student exercise and is out of sync with the automated pipeline — trust the workflow file for how deploys actually happen.

CI/CD flow (triggers on push to `main`):
1. Authenticate to AWS via GitHub OIDC (assumes IAM role `github-actions-deploy`, no stored keys)
2. `aws s3 sync` the site to the bucket (with `--delete`), excluding `.git`, `.github`, `.claude`, `terraform`, `.mcp.json`, and all `*.md`
3. Invalidate the CloudFront distribution (`/*`)

Concrete values currently hardcoded in the workflow: region `eu-north-1`, bucket `pravinmishradmi-site-production`, CloudFront distribution `E3V6O6MRE2E21P`, AWS account `533267262133`. Update these in `.github/workflows/deploy.yml` if the target infrastructure changes.

Practical implications of the `--sync --delete` + exclude list:
- Anything at the repo root that is **not** a `*.md` file or one of the excluded tooling dirs gets published — new `.html`, `.js`, `.css`, and image files ship automatically. Keep the site root free of stray files.
- `--delete` means removing a file locally removes it from the bucket on the next push. Don't rename/move site assets casually.
- Docs (`README.md`, `CLAUDE.md`, and any `*.md`) never reach the site, so they can't break it — but they also can't be served.

## Infrastructure (Terraform)

There is **no `terraform/` directory checked in** — it is generated on demand. The `scaffold-terraform` skill reads `.claude/skills/scaffold-terraform/template-spec.md` (the source of truth for the infra spec) and writes Terraform files into a new `terraform/` directory for S3 + CloudFront static hosting with OAC-based private-bucket access.

**Defaults vs. live infra mismatch:** `scaffold-terraform` defaults to region `ap-south-1` and name `portfolio-site`, but the live pipeline deploys to `eu-north-1` / bucket `pravinmishradmi-site-production`. If you scaffold infra intending to match the existing deployment, pass matching arguments (`/scaffold-terraform eu-north-1 pravinmishradmi-site`) rather than accepting the defaults — otherwise the generated infra targets a different region/bucket than the workflow expects.

## Skills (`.claude/skills/`)

Infrastructure and deployment tasks are driven by skills, not written by hand. All four are **manual-only** (`disable-model-invocation: true`) — invoke them explicitly; Claude will not auto-trigger them. Each declares a restricted `allowed-tools` set.

```
/scaffold-terraform [region] [name]  → Generate terraform/ files from template-spec.md
                                        (defaults: region ap-south-1, name portfolio-site)
/tf-plan                             → cd terraform && terraform plan -no-color, then risk analysis
/tf-apply                            → cd terraform && terraform apply -auto-approve; does NOT auto-retry on failure
/deploy                              → S3 sync (reads bucket/dist from terraform output) + CloudFront invalidation
```

Typical order for standing up fresh infra: `/scaffold-terraform` → review the generated files → `/tf-plan` → `/tf-apply` → `/deploy`.

Skill conventions to preserve when editing them:
- Each skill stops and reports on error rather than continuing to the next step.
- `tf-apply` never retries automatically — it shows the error and waits for instructions.
- `deploy` derives the bucket and distribution ID from `terraform output -json` rather than hardcoding them (unlike the GitHub workflow). If you change what the Terraform outputs are named, update the `deploy` skill to match.
- `scaffold-terraform` treats `template-spec.md` as the source of truth — change the spec, not the generated files, when the infra shape needs to change. Generated `terraform/` files are disposable.

## Conventions

- **Editing the site:** match the existing HTML structure and class naming in `index.html`; put all `index.html` styling in `style.css` (never inline). `privacy.html`/`terms.html` are the exception — their styles are inline. No linter, test suite, or build step exists; verify changes by opening the page in a browser.
- **Infrastructure:** all changes go through Terraform generated/managed by the skills — never modify AWS resources manually, and never hand-write Terraform outside the `scaffold-terraform` flow.
- **Auth:** GitHub Actions authenticates via OIDC — no long-lived AWS access keys anywhere in the repo. Don't introduce stored credentials.
- **Deploys:** site content changes deploy automatically on push to `main`. There is no staging environment — a merge to `main` is a production release.

## Repository Hygiene

There is a nested duplicate of the entire project at `Ultimate-Agentic-DevOps-with-Claude-Code/` (it has its own `.git`). It appears to be an accidental copy and is untracked. Do not edit files there; work in the repository root.
