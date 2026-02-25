<!-- .github/pull_request_template.md -->
## AI-Generated Section Review

### Section Name
<!-- e.g., Campaign Hero Banner -->

### Business Requirement
<!-- What did the client/marketing team request? Link to task if available. -->

### Checklist
- [ ] `shopify theme check` passes with zero errors
- [ ] No inline `<script>` or `<style>` blocks — all assets use the asset pipeline
- [ ] Images use `image_url` filter with explicit width parameters
- [ ] Collection loops include `limit` or `paginate`
- [ ] Section schema has ≤ 20 settings
- [ ] Schema uses theme color schemes instead of individual color pickers
- [ ] No logic duplicated from existing snippets (checked `snippets/` directory)
- [ ] Section tested in theme editor on staging theme — settings render correctly
- [ ] Visually reviewed on mobile (375px), tablet (768px), and desktop (1440px)
- [ ] PageSpeed Insights mobile score before: ___ | after: ___

### Screenshots
<!-- Before/after from staging theme -->
