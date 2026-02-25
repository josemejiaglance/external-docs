---
name: generate-sdk-docs
description: Generates Nextra v4 MDX documentation from any structured source files (JSON, YAML, Markdown, TypeScript definitions, etc.). Use when asked to generate, create, rebuild, or update documentation from data source files, or when importing docs into a Nextra v4 documentation portal. Source files can come from any project, SDK, API, or product — not limited to any specific format or domain. Covers the full pipeline: source discovery → content analysis → structure design → MDX generation → _meta.js creation → link and anchor validation.
---

# Generate Docs from Source Files

Produces a Nextra v4 MDX documentation tree from any structured source files. The structure, grouping, and page hierarchy are derived entirely from the source content — nothing is hardcoded.

---

## Phase 1 — Source Discovery

Scan the source directory for all data files:

```bash
find src/ -type f | sort
```

Source files can be any format: `.json`, `.yaml`, `.ts`, `.md`, `.xml`, `.csv`. Read every file before making any decisions.

For each file, extract:

| Signal | Examples |
|---|---|
| **What it represents** | product metadata, API surface, integration steps, error catalog, sample code, configuration schema |
| **Named entities** | class names, method signatures, type definitions, event codes, configuration keys |
| **Versioning** | multiple files for the same domain (e.g. `surface.v1.json`, `surface.v2.json`) |
| **External URLs** | repository links, documentation references, API endpoints, web app URLs |
| **Project identity** | product name, platform, language, package manager |

> If source URLs, repository links, or product names appear inside the data files, use those values — do not assume them.

---

## Phase 2 — Content Analysis & Grouping

After reading all source files, group them into documentation sections based on content semantics — not file names:

| Content type | Maps to section |
|---|---|
| Product metadata, requirements, install steps | `getting-started/` |
| Step-by-step implementation flows, feature walkthroughs | `guides/` |
| Class/method/type/enum reference | `api/` or `reference/` |
| Sample app, code walkthrough, key files | `sample-app/` or `examples/` |
| Symptom/cause/resolution catalog | `troubleshooting/` |
| Configuration schema, option tables | `configuration/` |
| Event catalog, webhook payloads | `events/` |

**Rules:**
- Sections emerge from the data. If no troubleshooting data exists, don't create a troubleshooting section.
- A single source file may contribute to multiple sections (e.g. a product file might feed both `getting-started/` and `api/`).
- Multiple versioned files for the same surface (e.g. `api.v1.json`, `api.v2.json`) consolidate into one section with version callouts, not parallel trees.

---

## Phase 3 — Structure Design

Before writing any file, output a proposed directory tree and get confirmation when in doubt.

### File naming conventions

- All folder and file names: lowercase kebab-case
- Spaces and underscores → hyphens (`my feature` → `my-feature`)
- Version suffixes: drop from filenames, capture in content instead
- Index files: always `index.mdx` (never `_index.mdx` or `readme.mdx`)

### Example structure shape (adapts to actual content)

```
docs/
└── <product-section>/         ← derived from product name or domain
    ├── index.mdx              (asIndexPage: true)
    ├── _meta.js
    ├── getting-started/
    │   ├── index.mdx          (asIndexPage: true)
    │   ├── _meta.js
    │   └── <one-file-per-step>.mdx
    ├── guides/
    │   ├── index.mdx          (asIndexPage: true)
    │   ├── _meta.js
    │   └── <one-file-per-feature>.mdx
    ├── api/                   ← or "reference/" depending on content
    │   ├── index.mdx          (asIndexPage: true)
    │   ├── _meta.js
    │   └── <one-file-per-class-or-group>.mdx
    └── troubleshooting/
        ├── index.mdx          (asIndexPage: true)
        ├── _meta.js
        └── <one-file-per-theme>.mdx
```

When `api/` or any reference section is placed as a **top-level sibling** of the main section (rather than nested inside it), all relative links from nested files must account for the extra `../` depth. Calculate depth before writing links.

### Depth-aware link formula

For any file at path `docs/A/B/C/file.mdx`, to link to `docs/X/Y/page.mdx`:

```
relative path = (number of segments from docs/ in source path - 1) × "../" + X/Y/page
```

| Source depth from docs/ | Prefix to reach a sibling top-level section |
|---|---|
| 1 level (`section/file.mdx`) | `../sibling/` |
| 2 levels (`section/sub/file.mdx`) | `../../sibling/` |
| 3 levels (`section/sub/sub/file.mdx`) | `../../../sibling/` |

---

## Phase 4 — MDX Generation

### Required front matter

```mdx
---
title: Exact Page Title
description: One-sentence description for search and SEO.
---

# Exact Page Title
```

- H1 text must **exactly match** `title` in front matter
- Folder index pages must also include `asIndexPage: true`
- Never mix front matter with `export const metadata`

### Standard page sections

Adapt these to whatever the source content contains:

| Section | Include when |
|---|---|
| Prerequisites / Requirements | Guides, tutorials, any sequential content |
| Overview / What it does | All pages |
| Parameters / Options table | API reference, configuration |
| Code examples | All guides and API pages |
| Version differences | Multiple versioned source files exist for same domain |
| Related links | All guides and troubleshooting pages |

### Version differences pattern

When multiple versioned source files cover the same API surface, consolidate into one page with a version callout derived from the actual version identifiers found in the source:

```mdx
import { Callout } from 'nextra/components'

<Callout type="info">
**v[CURRENT] (current):** [describe what changed — from source data]

**v[PREVIOUS]:** [describe previous behavior — from source data]
</Callout>
```

### API reference page pattern

```mdx
## MethodName(param:)

Brief description.

**Signature**

```swift showLineNumbers
func methodName(param: ParamType) -> ReturnType
```

**Parameters**

| Parameter | Type | Description |
|---|---|---|
| `param` | `ParamType` | What it controls |

**When called** — [describe trigger conditions from source]
```

### Troubleshooting page pattern

Group related entries from the symptom/cause/resolution catalog. Use H2 for each symptom.

```mdx
## [Symptom as a statement or question]

**Cause** — [from source]

**Fix**

```language showLineNumbers
// resolution code from source
```

> [any additional context from source]
```

---

## Phase 5 — `_meta.js` Rules (Critical)

**Never** add an `index` key to `_meta.js` when the folder's `index.mdx` uses `asIndexPage: true`.
This causes Nextra to throw: *"The field key 'index' in _meta file refers to a page that cannot be found"*.

```javascript
// ❌ WRONG
export default {
  index: { title: 'Overview', display: 'hidden' },
  installation: 'Installation',
}

// ✅ CORRECT
export default {
  installation: 'Installation',
  setup: 'Initial Setup',
}
```

### `_meta.js` key naming

Keys must exactly match the filename (without `.mdx`) of the corresponding page:

```javascript
export default {
  'getting-started': 'Getting Started',   // → getting-started/index.mdx (folder)
  'quick-start': 'Quick Start',           // → quick-start.mdx
  'api-reference': 'API Reference',       // → api-reference/ (folder)
}
```

### Separators and section labels

Use a `type: 'separator'` entry to add visual section dividers in the sidebar:

```javascript
export default {
  overview: 'Overview',
  '---': {
    type: 'separator',
    title: 'Guides',
  },
  'start-session': 'Start a Session',
  'video-session': 'Video Sessions',
}
```

---

## Phase 6 — Anchor Fragment Validation

Nextra uses [github-slugger](https://github.com/Flet/github-slugger) for heading-to-anchor conversion.

**Algorithm:**
1. Take the full heading text (strip markdown formatting like backticks and bold)
2. Lowercase everything
3. Replace every contiguous sequence of non-alphanumeric characters with a single `-`
4. Trim leading and trailing `-`

**Key gotcha** — method signatures in backticks get their punctuation stripped:

| Heading | Anchor |
|---|---|
| `## Overview` | `#overview` |
| `## Getting Started` | `#getting-started` |
| `## GlanceSDKEventCode` | `#glancesdkeventcode` |
| `### \`methodName(param:)\`` | `#methodname-param` |
| `### \`hostedSessionDidEnd(errorMessage:)\`` | `#hostedsessiondidend-errormessage` |
| `## Content masking not working — agent sees masked content` | `#content-masking-not-working-agent-sees-masked-content` |

> Em dashes (`—`), parentheses `()`, colons `:`, backticks, and all other non-alphanumeric characters collapse into the surrounding replacement hyphen.

**Always verify** anchor links against the actual heading text of the target section before writing them.

---

## Phase 7 — External Link Tracking

Collect all external URLs from the generated docs into `missinglinks.txt` at the project root.
Extract URLs from both the source data files and any generated MDX content.

**Format:**

```text
External Links to Verify
========================

1. [Human-readable description]
   File : docs/path/to/file.mdx (line N)
   URL  : https://example.com/...
   Used : [context — what the link points to]
```

Do not assume external URLs are valid or accessible. Flag them all for manual verification.

---

## Phase 8 — Do Not Overwrite

Check before writing every file. Only create new files:

```bash
ls docs/<target-dir>/ 2>/dev/null
```

If a file exists, report it and skip — never silently overwrite.

---

## Nextra Component Patterns

### Cards (index pages)

```mdx
import { Cards } from 'nextra/components'

<Cards num={2}>
  <Cards.Card title="Section Title" href="./section" arrow>
    One-line description of what this section covers.
  </Cards.Card>
</Cards>
```

Always set `num` explicitly. Use `num={2}` for most cases; `num={3}` only when 3 equal-weight sections exist.

### Steps (sequential guides)

```mdx
import { Steps } from 'nextra/components'

<Steps>
### Step title

Step content here.

### Next step

More content.
</Steps>
```

Use `Steps` for any ordered, sequential process (installation, setup, integration flows).

### Callout

```mdx
import { Callout } from 'nextra/components'

<Callout type="info">Informational note.</Callout>
<Callout type="warning">Something to watch out for.</Callout>
<Callout type="error">Breaking or dangerous action.</Callout>
<Callout>Default callout (no type = "default").</Callout>
```

### Tabs

```mdx
import { Tabs } from 'nextra/components'

<Tabs items={['Swift', 'Objective-C']}>
  <Tabs.Tab>
  ```swift showLineNumbers
  // Swift code
  ```
  </Tabs.Tab>
  <Tabs.Tab>
  ```objectivec showLineNumbers
  // Obj-C code
  ```
  </Tabs.Tab>
</Tabs>
```

Use `Tabs` when the same concept has multiple implementations (languages, package managers, OS variants).

---

## Quick Checklist

Before finishing:

- [ ] All source files read and fully analyzed before writing any MDX
- [ ] Directory structure proposed and reviewed before creation
- [ ] Every `index.mdx` has `asIndexPage: true`
- [ ] No `index:` key in any `_meta.js`
- [ ] `_meta.js` keys match actual filenames exactly (kebab-case)
- [ ] H1 text exactly matches front matter `title`
- [ ] Relative link depths calculated correctly per file location
- [ ] Anchor fragments verified against actual headings using slugger rules
- [ ] External URLs collected in `missinglinks.txt`
- [ ] Version differences consolidated with `<Callout>` — not parallel files
- [ ] No existing `.mdx` files overwritten
- [ ] All source URLs/identifiers extracted from data, not hardcoded
