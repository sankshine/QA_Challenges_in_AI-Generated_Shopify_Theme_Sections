# Shopify Theme QA Guardrails

A CI/CD pipeline and review framework for managing AI-generated Shopify theme sections in production. This repository provides automated checks, branching strategy templates, and deployment guardrails to ensure AI-generated Liquid code meets Shopify's performance and maintainability standards before reaching a live storefront.

---

## The Problem

Shopify's GitHub integration allows a theme's codebase to be connected with a branch in a GitHub repository. When a client uses an AI assistant — such as Cursor, GitHub Copilot, or a ChatGPT-based workflow — to generate new `.liquid` section files and commit them directly to the connected branch, the AI is effectively acting as an **unsupervised developer with direct deploy access to the live storefront**.

This creates three categories of risk:

### 1. Performance Degradation

AI-generated Liquid code often works correctly but ignores Shopify's performance best practices, hurting Core Web Vitals (LCP, CLS, INP), conversion rates, and SEO.

**Unscoped collection loops:** AI frequently generates loops that iterate every product in a collection without a `limit` or `paginate` tag. On a collection with 500+ products, this forces Shopify's Liquid engine to process all 500 objects server-side before sending a single byte of HTML to the browser, delaying the server response (TTFB), which in turn pushes back when the browser can begin rendering the largest visible element on screen (LCP).


### 2. Inline JavaScript and CSS

AI tools commonly embed `<script>` and `<style>` blocks directly inside section files instead of using Shopify's asset pipeline (`asset_url` filter). This bypasses CDN caching, introduces render-blocking resources, and duplicates the payload every time the section appears on multiple pages.

### 3. Schema Bloat

AI-generated schemas often include far too many settings. A simple promotional banner might arrive with 30+ options (separate images for each device, individual color pickers, font sizes, padding values, etc.), overwhelming the merchant in the theme editor and creating untested edge-case combinations that break layouts.



## Version Control Challenges

When clients use AI to modify sections through GitHub-connected production branches, four specific problems arise:

| Challenge | What Happens | Impact |
|---|---|---|
| **Direct commits skip review** | AI commits straight to `main`, which auto-deploys to the live store | A syntax error can break the theme editor or crash the homepage |
| **Sync conflicts with theme editor** | Marketing manager saves in the customizer while AI commits via GitHub | Shopify forces a choose-one resolution; design work gets silently discarded |
| **Messy commit history** | AI generates vague messages like "Add new section" | Developers cannot trace which commit introduced a regression |
| **No staging buffer** | Default setup has no unpublished preview theme | Every commit is a production deployment |

---

## The Solution: QA Guardrails

### Branching Strategy

This repository enforces a three-branch model that separates AI-generated code from the live store:

```
feature/ai-campaign-hero  →  staging  →  main
        ↓                       ↓          ↓
   AI commits here       Preview theme   Live store
                         (unpublished)   (published)
```

- **`main`** → Connected to the published live theme. Branch protection requires pull requests and at least one approval. No direct commits.
- **`staging`** → Connected to an unpublished preview theme. New sections are tested in the actual theme editor here.
- **`feature/*`** → Where all AI-generated work is committed (e.g., `feature/ai-testimonial-slider`).

**Workflow:** AI commits to feature branch → PR to `staging` → automated checks run → developer reviews → merged to `staging` → marketing previews in theme editor → PR to `main` → merged → live.

---

### Automated CI Pipeline

**What the pipeline catches:**

| Check | What It Detects | Why It Matters |
|---|---|---|
| `shopify theme check` | Liquid syntax errors, deprecated filters, missing `asset_url`, schema validation | Official Shopify linter — catches the broadest range of issues |
| Inline script detection | `<script>` tags inside section files | Bypasses CDN caching, blocks rendering |
| Unscoped loop detection | Collection loops without `limit` or `paginate` | Inflates TTFB and LCP on large collections |
| Schema JSON validation | Malformed JSON in `{% schema %}` blocks | Broken schema crashes the theme editor for that section |
| Settings count check | Sections with >20 settings | Overwhelms merchants, creates untested edge cases |

---

### Pull Request Template

Every AI-generated section is reviewed against a standard checklist (`.github/pull_request_template.md`):

```markdown
## AI-Generated Section Review

### Section Name
<!-- e.g., Campaign Hero Banner -->

### Business Requirement
<!-- What did the client/marketing team request? -->

### Checklist
- [ ] `shopify theme check` passes with zero errors
- [ ] No inline `<script>` or `<style>` blocks
- [ ] Images use `image_url` filter with explicit width parameters
- [ ] Collection loops include `limit` or `paginate`
- [ ] Section schema has ≤ 20 settings
- [ ] Schema uses theme color schemes (not individual color pickers)
- [ ] No logic duplicated from existing snippets
- [ ] Tested in theme editor on staging theme
- [ ] Visually reviewed on mobile (375px), tablet (768px), desktop (1440px)
- [ ] PageSpeed Insights mobile score — before: ___ | after: ___

### Screenshots
<!-- Before/after from staging theme -->
```

---

### Rollback Strategy

Every merge to `main` is tagged with a version and date for instant recovery:

```bash
git tag -a v1.4.2-2026.02.25 -m "Pre-merge: adding campaign hero section"
git push origin v1.4.2-2026.02.25
```



## Key Takeaway

AI-generated Shopify sections accelerate development but introduce performance, maintainability, and deployment risks when committed directly to production. This repository treats the AI as an **untrusted contributor** — its output goes through a feature branch, an automated CI pipeline, a developer review via pull request, and visual QA on a staging theme before it ever reaches the published store.
