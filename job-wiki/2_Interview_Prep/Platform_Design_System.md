# Platform & Design System Architecture — UI Principal Engineer Interview Prep
### ConnectWise Platform Context

[← Back to Index](../README.md)

---

## 1. What Platform Team Actually Means

Get this framing right — it's the difference between a senior answer and a principal answer.

```
Feature team:   ships user-facing features, optimizes for velocity
Platform team:  ships infrastructure other teams BUILD ON
                → wrong decision = 10 teams suffer
                → right decision = 10 teams go faster

Platform team owns:
  - Shell app (routing, auth, layout chrome)
  - Design system (@cw/ui or similar)
  - Shared utilities (@cw/api-client, @cw/analytics)
  - Monorepo tooling and CI templates
  - MFE contracts and standards
  - Developer experience (DX)
```

---

## 2. Monorepo vs Polyrepo

```
POLYREPO (one repo per team / product)
  ✅ True team autonomy — own your pipeline, own your pace
  ✅ Smaller CI surface — billing's build doesn't break on manage's PR
  ❌ Sharing code is painful — publish to npm, version, wait for consumers to upgrade
  ❌ Tooling diverges — 8 teams, 8 ESLint configs, 8 different conventions
  ❌ Cross-cutting changes (React upgrade) = 8 PRs, 8 reviews, 8 merges
  Use: teams are TRULY independent (different companies, separate release cadences)

MONOREPO (all shared code in one repo)
  ✅ Atomic cross-cutting changes — React upgrade = 1 PR, affects all
  ✅ Shared tooling enforced (ESLint, TS config, test setup)
  ✅ Design system changes visible immediately — no publish/upgrade cycle
  ✅ See impact of breaking changes across all consumers before merging
  ❌ CI must be smart — don't rebuild everything on every PR
  ❌ Repo gets large — needs caching and affected-only builds
  Use: when platform + product teams share significant code

RECOMMENDATION for ConnectWise:
  Monorepo for: platform libs, design system, shared utilities
  Polyrepo for: individual product MFEs (they deploy independently)
  Hybrid: platform team owns the monorepo, product teams import from it
```

### Tooling: Nx (preferred for multi-team platform)

```bash
# Only runs tests for changed libs + their dependents
nx affected:test --base=main

# Only builds what changed
nx affected:build --base=main

# Visual dependency graph — great for architecture reviews
nx graph

# Enforce module boundaries — billing cannot import from manage
# nx.json
"@nx/enforce-module-boundaries": [{
  "sourceTag": "scope:billing",
  "onlyDependOnLibsWithTags": ["scope:shared"]
}]

# Generate a new MFE with standard setup
nx generate @cw/generators:mfe billing
# → repo structure, webpack config, CI pipeline, Dockerfile, all wired correctly
# → team is productive in 30 minutes, not 3 days
```

---

## 3. Design System Architecture

### Package structure

```
packages/
  @cw/tokens/        ← design tokens (colors, spacing, typography, shadow)
  @cw/ui/            ← component library (Button, Modal, Table, Form...)
  @cw/icons/         ← SVG icon set (tree-shakeable)
  @cw/hooks/         ← shared hooks (useDebounce, usePagination, useIntersection...)
  @cw/utils/         ← pure utilities (formatDate, formatCurrency, truncate...)
  @cw/api-client/    ← base Axios instance with auth + retry + error mapping
  @cw/analytics/     ← event tracking wrapper (Segment/Amplitude)
  @cw/generators/    ← Nx generators for scaffolding new MFEs
```

### Design Tokens — the foundation

```typescript
// @cw/tokens/src/tokens.ts — single source of truth
export const tokens = {
  color: {
    brand: {
      primary:   '#0052CC',
      secondary: '#00B8D9',
    },
    semantic: {
      success: '#36B37E',
      danger:  '#FF5630',
      warning: '#FFAB00',
      info:    '#0065FF',
    },
    neutral: {
      50:  '#F7F8F9',
      100: '#EBECF0',
      500: '#6B778C',
      900: '#091E42',
    }
  },
  spacing: {
    xs: '4px',
    sm: '8px',
    md: '16px',
    lg: '24px',
    xl: '32px',
    xxl: '48px',
  },
  typography: {
    fontFamily: {
      base: "'Inter', -apple-system, sans-serif",
      mono: "'JetBrains Mono', monospace",
    },
    fontSize:   { xs: '11px', sm: '12px', base: '14px', lg: '18px', xl: '24px', xxl: '32px' },
    fontWeight: { regular: 400, medium: 500, semibold: 600, bold: 700 },
    lineHeight: { tight: 1.2, base: 1.5, relaxed: 1.75 },
  },
  radius:  { sm: '4px', md: '8px', lg: '12px', xl: '16px', full: '9999px' },
  shadow:  {
    sm: '0 1px 2px rgba(9,30,66,.08)',
    md: '0 4px 8px rgba(9,30,66,.10)',
    lg: '0 8px 16px rgba(9,30,66,.12)',
  },
} as const;

// Generate CSS variables — shell injects these globally
export function generateCSSVariables(t = tokens): string {
  return `
    --color-brand-primary:    ${t.color.brand.primary};
    --color-brand-secondary:  ${t.color.brand.secondary};
    --color-semantic-success: ${t.color.semantic.success};
    --color-semantic-danger:  ${t.color.semantic.danger};
    --color-neutral-900:      ${t.color.neutral[900]};
    --spacing-sm:  ${t.spacing.sm};
    --spacing-md:  ${t.spacing.md};
    --spacing-lg:  ${t.spacing.lg};
    --font-base:   ${t.typography.fontSize.base};
    --radius-md:   ${t.radius.md};
    --shadow-md:   ${t.shadow.md};
  `;
}
```

### Theming — per-tenant brand colors via CSS variables

```javascript
// Components use CSS variables, never hardcoded values
const Button = styled.button`
  background:    var(--color-brand-primary);
  padding:       var(--spacing-sm) var(--spacing-md);
  border-radius: var(--radius-md);
  font-size:     var(--font-base);
`;

// Shell injects tenant theme at startup — every MFE's Button adapts automatically
async function applyTenantTheme(orgId: string) {
  const theme = await fetchTenantTheme(orgId);
  const root = document.documentElement;
  root.style.setProperty('--color-brand-primary', theme.primaryColor);
  root.style.setProperty('--color-brand-secondary', theme.secondaryColor);
  // Zero changes needed in any MFE ✅
}

// Dark mode — same mechanism
function applyDarkMode() {
  document.documentElement.setAttribute('data-theme', 'dark');
  // CSS: [data-theme="dark"] { --color-neutral-50: #1A1A2E; ... }
}
```

---

## 4. Versioning Strategy — The Hardest Platform Problem

This is where Principals get separated from seniors.

```
Semantic versioning:
  MAJOR (2.0.0) — breaking change (rename prop, remove component, change behavior)
  MINOR (1.3.0) — additive (new component, new optional prop)
  PATCH (1.2.1) — bug fix, no API change

The real problem:
  Breaking change in @cw/ui = ALL teams must migrate simultaneously
  Teams resist it → design system falls behind → teams work around it → chaos
```

### Deprecation cycle — avoid big-bang migrations

```javascript
// @cw/ui v1.3.0 — add new API alongside old
function Button({ type, variant, ...props }) {
  // Support both during migration window
  const resolvedVariant = variant ?? type;

  if (type !== undefined && variant === undefined) {
    console.warn(
      '[CW] Button: `type` prop is deprecated and will be removed in v2.0. Use `variant` instead.'
    );
  }

  return <button className={`btn btn-${resolvedVariant}`} {...props} />;
}

// Storybook shows deprecation badge automatically via JSDoc:
/**
 * @deprecated Use `variant` instead. Will be removed in v2.0.
 */
type?: 'primary' | 'secondary' | 'danger';
```

**Migration timeline:**
```
v1.3.0 → announce: `type` deprecated, `variant` is the new API
          console.warn in dev, Storybook badge
Sprint 1-2 → teams migrate at their own pace (no blocking)
v2.0.0 → `type` removed — teams are already migrated
          v2.0.0 has ZERO actual migration work for teams ✅
```

### Codemods — make migration cost zero

```bash
# @cw/codemod — platform team ships this alongside the deprecation
npx @cw/codemod button-type-to-variant ./src

# Automatically rewrites:
# <Button type="primary"> → <Button variant="primary">
# across an entire codebase in seconds

# Team migration time: 5 minutes, not 2 days
```

### Canary releases — test with one team before shipping to all

```bash
# Platform team publishes a canary
npm publish --tag canary    # → @cw/ui@2.0.0-canary.1

# One brave team installs it
npm install @cw/ui@canary

# Feedback loop → platform fixes issues
npm publish                  # → @cw/ui@2.0.0 stable — ships to all teams
```

---

## 5. Storybook — Non-Negotiable

Every principal-level interviewer expects this to come up.

```
Storybook serves as:
  1. Living documentation — "does this component exist?" → Storybook first
  2. Development environment — build components without the full app running
  3. Visual regression testing — Chromatic catches unintended UI changes in CI
  4. Accessibility testing — @storybook/addon-a11y flags violations per story
  5. Design handoff — designers review real components, not Figma estimates
```

```typescript
// @cw/ui/src/Button/Button.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import { Button } from './Button';

const meta: Meta<typeof Button> = {
  title: 'Components/Button',
  component: Button,
  tags: ['autodocs'],
};
export default meta;

type Story = StoryObj<typeof Button>;

export const Primary:  Story = { args: { variant: 'primary',   children: 'Save' } };
export const Danger:   Story = { args: { variant: 'danger',    children: 'Delete' } };
export const Loading:  Story = { args: { loading: true,        children: 'Saving...' } };
export const Disabled: Story = { args: { disabled: true,       children: 'Unavailable' } };

export const AllVariants: Story = {
  render: () => (
    <div style={{ display: 'flex', gap: 8 }}>
      {(['primary','secondary','ghost','danger'] as const).map(v => (
        <Button key={v} variant={v}>{v}</Button>
      ))}
    </div>
  )
};
```

**CI integration:**
```yaml
# .github/workflows/chromatic.yml
- name: Publish to Chromatic
  run: npx chromatic --project-token=${{ secrets.CHROMATIC_TOKEN }}
# PR gets visual diff — changed stories highlighted
# Reviewer approves or rejects the visual change before merge
```

---

## 6. Developer Experience (DX) — What Principals Are Judged On

The best platform teams are obsessed with DX. If using the design system is painful, teams work around it.

```
1. Scaffolding generators
   nx generate @cw/generators:mfe billing
   → MFE repo structure, webpack config (with Module Federation wired),
     CI pipeline, Dockerfile, ESLint + TS config
   → Team is productive in 30 minutes, not 3 days

2. TypeScript everywhere
   → Wrong prop = compile error, not a runtime bug in production
   → Autocomplete for all component props

3. VSCode snippets
   → Type cw-button → expands to full JSX with correct imports
   → Removes "what was that prop called?" friction

4. Changelog automation
   → Conventional commits → auto-generated CHANGELOG.md per package
   → Teams always know what changed between versions
   → semantic-release handles publish + git tags automatically

5. Performance budget in CI
   → Each MFE has a bundle size limit
   → PR that exceeds it fails CI with exact diff showing what got bigger
   bundlesize check ./dist/billing.js --max 250KB

6. Adoption tracking
   → Monitor which version of @cw/ui each team is on
   → If 6 of 8 teams are on v1 six months after v2 ships → DX failure
   → Platform team's job is to make upgrading easy, not to blame teams

7. Office hours + Slack support
   → Platform team runs weekly DX office hours
   → #platform-help channel with SLA
   → Build WITH teams, not for teams
```

---

## 7. Governance Model — Cross-Team Standards at Scale

```
Who makes decisions?
  Platform team: owns the design system, MFE contracts, shell
  RFC process: significant changes go through a written RFC
    → team proposes a pattern → platform reviews → all teams comment
    → builds buy-in before implementation, surfaces edge cases early

How do teams contribute to the design system?
  Option A: Platform team only
    → highest quality control
    → bottleneck — platform can't keep up with 8 teams' needs
  
  Option B: Open contribution (any team can PR)
    → fast, no bottleneck
    → quality varies, inconsistent patterns creep in
  
  Option C (RECOMMENDED): Contribution with platform review
    → Product teams build the component in their MFE first
    → When it proves useful, they PR it to @cw/ui
    → Platform team reviews API design, a11y, docs, tests
    → Component graduates to the design system
    → Teams own what they contribute

Breaking change governance:
  → Platform team reviews ALL PRs to @cw/ui
  → ESLint rule flags breaking changes automatically
  → MAJOR bumps require RFC + 2-sprint deprecation window
```

---

## 8. Interview Answers

### "How would you build and govern a design system for 8 teams?"

> "Start with design tokens — a single TypeScript source of truth for color, spacing, and typography. Tokens generate CSS variables the shell injects globally, so all MFEs inherit theming including per-tenant brand colors without any MFE code changes.
>
> The component library sits on tokens, documented in Storybook with Chromatic running visual regression in CI. Every component has stories for all variants and an accessibility audit.
>
> The hardest part is versioning. I avoid breaking changes entirely where possible by running a deprecation cycle: add the new API alongside the old with a console warning, give teams two sprints to migrate, then remove the old in the major. For any unavoidable migration I ship a codemod — teams run one command and they're done in five minutes.
>
> For governance: platform team reviews all design system PRs, but product teams contribute components that proved useful in their MFE. We track adoption — if teams aren't upgrading, that's a DX signal, not a team failure. The fix is better tooling, clearer changelogs, or a migration script."

### "How do you enforce consistency across 8 UI teams without becoming a bottleneck?"

> "Three things: tooling, templates, and trust. Tooling means Nx generators so spinning up a new MFE produces the correct webpack config, ESLint rules, and CI pipeline by default — consistency is automatic, not enforced by policing. Templates mean RFC documents for significant patterns so teams understand the 'why', not just the 'what'. And trust means product teams can contribute to the design system — platform reviews and guides the API design, but teams aren't blocked waiting for the platform team to build everything. When teams have ownership and the tools make the right path easy, you don't need a bottleneck."

### "Monorepo or polyrepo for a multi-team UI platform?"

> "Hybrid. The design system, shared utilities, and platform libs live in a monorepo owned by the platform team — that's where Nx's affected builds and atomic cross-cutting changes pay off. Product MFEs stay in their own repos for true deployment independence. Product teams import from the platform monorepo as a versioned dependency. This gives us the sharing benefits of a monorepo for shared code and the deployment independence of polyrepo for product teams."

### "How do you handle a breaking change in the design system?"

> "I try to avoid them entirely through the deprecation cycle: add the new API alongside the old in a minor version with a console warning in dev, announce it in the team Slack and Storybook changelog. Teams migrate at their own pace over two sprints. When we ship the major version removing the old API, teams are already migrated — so the 'breaking' release has zero actual migration work. For any remaining cases I ship a codemod with the deprecation announcement, so migration is a one-command operation."
