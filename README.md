# Shopify Theme QA Guardrails

A CI/CD pipeline and review framework for managing AI-generated Shopify theme sections in production. This repository provides automated checks, branching strategy templates, and deployment guardrails to ensure AI-generated Liquid code meets Shopify's performance and maintainability standards before reaching a live storefront.

---

## The Problem

Shopify's GitHub integration allows a theme's codebase to be connected with a branch in a GitHub repository. When a client uses an AI assistant — such as Cursor, GitHub Copilot, or a ChatGPT-based workflow — to generate new `.liquid` section files and commit them directly to the connected branch, the AI is effectively acting as an **unsupervised developer with direct deploy access to the live storefront**.

This creates three categories of risk:

### 1. Performance Degradation

AI-generated Liquid code often works correctly but ignores Shopify's performance best practices, hurting Core Web Vitals (LCP, CLS, INP), conversion rates, and SEO.

**Unscoped collection loops:** AI frequently generates loops that iterate every product in a collection without a `limit` or `paginate` tag. On a collection with 500+ products, this forces Shopify's Liquid engine to process all 500 objects server-side before sending a single byte of HTML to the browser, delaying the server response (TTFB), which in turn pushes back when the browser can begin rendering the largest visible element on screen (LCP).

```liquid
❌ BAD: No limit, no pagination
{% for product in collections['summer-sale'].products %}
  <div class="promo-card">
    <img src="{{ product.featured_image | img_url: 'master' }}">
    <h3>{{ product.title }}</h3>
    <p>{{ product.price | money }}</p>
  </div>
{% endfor %}
```

```liquid
✅ CORRECT: Paginated, limited, responsive images
{% paginate collections['summer-sale'].products by 12 %}
  {% for product in collections['summer-sale'].products %}
    <div class="promo-card">
      <img
        srcset="{{ product.featured_image | image_url: width: 400 }} 400w,
               {{ product.featured_image | image_url: width: 800 }} 800w"
        sizes="(max-width: 768px) 400px, 800px"
        src="{{ product.featured_image | image_url: width: 800 }}"
        alt="{{ product.title }}"
        loading="lazy"
        width="800"
        height="800"
      >
      <h3>{{ product.title }}</h3>
      <p>{{ product.price | money }}</p>
    </div>
  {% endfor %}
  {{ paginate | default_pagination }}
{% endpaginate %}
```

### 2. Inline JavaScript and CSS

AI tools commonly embed `<script>` and `<style>` blocks directly inside section files instead of using Shopify's asset pipeline (`asset_url` filter). This bypasses CDN caching, introduces render-blocking resources, and duplicates the payload every time the section appears on multiple pages.

```liquid
❌ BAD: Inline script and style
<style>
  .testimonial-slider { display: flex; overflow: hidden; }
</style>
<div class="testimonial-slider" id="testimonialSlider">
  {% for block in section.blocks %}
    <div class="testimonial-slide">
      <p>"{{ block.settings.quote }}"</p>
    </div>
  {% endfor %}
</div>
<script>
  let current = 0;
  const slides = document.querySelectorAll('.testimonial-slide');
  setInterval(() => { /* ... */ }, 4000);
</script>
```

```liquid
✅ CORRECT: External assets, deferred loading
{{ 'section-testimonial-slider.css' | asset_url | stylesheet_tag }}

<div class="testimonial-slider" id="testimonialSlider">
  {% for block in section.blocks %}
    <div class="testimonial-slide" {{ block.shopify_attributes }}>
      <p>"{{ block.settings.quote }}"</p>
    </div>
  {% endfor %}
</div>

<script src="{{ 'section-testimonial-slider.js' | asset_url }}" defer></script>
```

### 3. Schema Bloat

AI-generated schemas often include far too many settings. A simple promotional banner might arrive with 30+ options (separate images for each device, individual color pickers, font sizes, padding values, etc.), overwhelming the merchant in the theme editor and creating untested edge-case combinations that break layouts.

```json
❌ BAD: 16+ settings for a simple banner
{
  "name": "Promo Banner",
  "settings": [
    { "type": "image_picker", "id": "image_desktop", "label": "Desktop Image" },
    { "type": "image_picker", "id": "image_tablet", "label": "Tablet Image" },
    { "type": "image_picker", "id": "image_mobile", "label": "Mobile Image" },
    { "type": "range", "id": "heading_size", "min": 12, "max": 72, "step": 1 },
    { "type": "color", "id": "heading_color", "label": "Heading Color" },
    { "type": "color", "id": "overlay_color", "label": "Overlay Color" },
    { "type": "range", "id": "overlay_opacity", "min": 0, "max": 100 },
    { "type": "range", "id": "button_radius", "min": 0, "max": 40 },
    { "type": "range", "id": "padding_top", "min": 0, "max": 100 },
    { "type": "range", "id": "padding_bottom", "min": 0, "max": 100 }
  ]
}
```

```json
✅ CORRECT: Minimal schema using theme-level color schemes
{
  "name": "Promo Banner",
  "settings": [
    { "type": "image_picker", "id": "image", "label": "Banner Image" },
    { "type": "text", "id": "heading", "label": "Heading", "default": "Shop the Collection" },
    { "type": "text", "id": "button_text", "label": "Button Text", "default": "Shop Now" },
    { "type": "url", "id": "button_link", "label": "Button Link" },
    {
      "type": "select",
      "id": "color_scheme",
      "label": "Color Scheme",
      "options": [
        { "value": "scheme-1", "label": "Primary" },
        { "value": "scheme-2", "label": "Secondary" }
      ],
      "default": "scheme-1"
    }
  ],
  "presets": [{ "name": "Promo Banner" }]
}
```

---

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

The GitHub Actions workflow (`.github/workflows/theme-qa.yml`) runs on every pull request targeting `staging` or `main`:

```yaml
name: Shopify Theme QA

on:
  pull_request:
    branches: [main, staging]

jobs:
  theme-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Shopify CLI
        run: npm install -g @shopify/cli @shopify/theme

      - name: Run Theme Check (Liquid Linting)
        run: shopify theme check --fail-level error

      - name: Check for Inline Script/Style Blocks
        run: |
          VIOLATIONS=$(grep -rn '<script>' sections/ --include="*.liquid" \
            | grep -v 'asset_url' | grep -v 'src=' || true)
          if [ -n "$VIOLATIONS" ]; then
            echo "❌ Inline <script> blocks found in section files:"
            echo "$VIOLATIONS"
            echo "Move scripts to assets/ and load via asset_url with defer."
            exit 1
          fi

      - name: Check for Unscoped Collection Loops
        run: |
          VIOLATIONS=$(grep -rn "for product in collections\[" sections/ \
            --include="*.liquid" | grep -v "limit:" | grep -v "paginate" || true)
          if [ -n "$VIOLATIONS" ]; then
            echo "⚠️ Collection loops without limit or paginate found:"
            echo "$VIOLATIONS"
            exit 1
          fi

      - name: Validate Section Schema JSON
        run: |
          for file in sections/*.liquid; do
            SCHEMA=$(sed -n '/{% schema %}/,/{% endschema %}/p' "$file" \
              | sed '1d;$d')
            if [ -n "$SCHEMA" ]; then
              echo "$SCHEMA" | python3 -m json.tool > /dev/null 2>&1
              if [ $? -ne 0 ]; then
                echo "❌ Invalid JSON schema in $file"
                exit 1
              fi
            fi
          done

      - name: Schema Settings Count Check
        run: |
          for file in sections/*.liquid; do
            SCHEMA=$(sed -n '/{% schema %}/,/{% endschema %}/p' "$file" \
              | sed '1d;$d')
            if [ -n "$SCHEMA" ]; then
              COUNT=$(echo "$SCHEMA" | python3 -c \
                "import sys,json; d=json.load(sys.stdin); \
                print(len(d.get('settings',[])))" 2>/dev/null || echo "0")
              if [ "$COUNT" -gt 20 ]; then
                echo "⚠️ $file has $COUNT schema settings (recommended: ≤20)"
              fi
            fi
          done
```

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

**Two rollback paths:**

| Method | Speed | When to Use |
|---|---|---|
| **Publish backup theme** from Shopify admin | ~60 seconds | Critical break — homepage down, checkout broken |
| **`git revert <merge-sha>`** on `main` | ~10 minutes | Targeted removal of a specific section while preserving other recent changes |

---

## Repository Structure

```
shopify-theme-qa-guardrails/
├── .github/
│   ├── workflows/
│   │   └── theme-qa.yml              # CI pipeline — runs on every PR
│   └── pull_request_template.md       # Standard review checklist
├── examples/
│   ├── bad/
│   │   ├── ai-promo-banner.liquid     # ❌ Unscoped loops, inline JS, bloated schema
│   │   └── ai-testimonial-slider.liquid
│   └── fixed/
│       ├── promo-banner.liquid        # ✅ Paginated, asset pipeline, minimal schema
│       └── testimonial-slider.liquid
├── docs/
│   ├── branching-strategy.md          # Three-branch model explained
│   ├── client-workflow-guide.md       # One-pager for marketing teams
│   └── rollback-procedures.md         # Step-by-step recovery guide
└── README.md
```

---

## Key Takeaway

AI-generated Shopify sections accelerate development but introduce performance, maintainability, and deployment risks when committed directly to production. This repository treats the AI as an **untrusted contributor** — its output goes through a feature branch, an automated CI pipeline, a developer review via pull request, and visual QA on a staging theme before it ever reaches the published store.
