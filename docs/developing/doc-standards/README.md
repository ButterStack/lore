# Documentation standards

The house style for every Markdown doc in the Lore project. Use this folder when you're writing a new doc, reviewing one, or bringing an existing page into compliance.

## Start here

**Writing or contributing a doc?** Walk through [`writing-a-doc.md`](writing-a-doc.md) — a 10-minute on-ramp covering the rules that bind a small contribution, with depth pointers into the canon for substantial work. Items not in the on-ramp are maintainer concerns on second-pass review.

**Reviewing or self-validating a doc?** Open [`operational/review-checklist.md`](operational/review-checklist.md). Each item links back to the rule it enforces, so a failed check tells you exactly where to look.

## Reference

The walkthrough and the review checklist cite these by topic. Read them when something sends you here, or use them as lookup when you have a specific question.

### Canon

- [`canon/doc-types.md`](canon/doc-types.md). All seven Lore doc types — four user-facing (Tutorial, How-To, Reference, Explanation), three contributor (Internals, ADR, Code-Standard) — plus Landing pages.
- [`canon/language.md`](canon/language.md). Voice, mood, contractions, pronouns, word choice, branding, and link conventions.
- [`canon/format.md`](canon/format.md). Headings, lists, tables, bold/italic/code, callouts, and key combinations.

### Operational

- [`operational/filenames.md`](operational/filenames.md). Filename conventions and SEO basics.
- [`operational/accessibility-and-media.md`](operational/accessibility-and-media.md). Alt text, color, screen-reader pass, diagrams, and image conventions.
- [`operational/review-checklist.md`](operational/review-checklist.md). Pre-publish gating checklist (includes the open-source posture rules).
- [`operational/glossary-conventions.md`](operational/glossary-conventions.md). Rules for adding terms to the project glossary at `docs/glossary.md`.

### Tooling

- [`tools/README.md`](tools/README.md). The three linters — Vale (prose), markdownlint (structure), lychee (links) — with invocation, rules, and install. Tool configs live next to the README under `tools/vale/` and `tools/lychee/`.

## Authority

When two rules conflict, the higher entry wins. The [Microsoft Writing Style Guide](https://learn.microsoft.com/en-us/style-guide/welcome/) informed the foundation of these standards and is a useful reference for grammar, punctuation, and mechanics not addressed explicitly here. Lore also adds its own rules for vocabulary, product naming, and project-specific conventions. Where Lore style choices differ from the guide, they're documented at their topical homes with rationale.

1. [`canon/doc-types.md`](canon/doc-types.md). Doc-type definitions for both type families.
2. [`canon/language.md`](canon/language.md) and [`canon/format.md`](canon/format.md). Voice, language, structure, format.
3. [`operational/`](operational/). Pre-publish mechanics.

## Scope

These standards govern Lore's documentation house style.

**About Lore.** Lore is an open-source, distributed version control system built for binary-first workloads at scale.

**Audiences.** Lore docs serve developers running the `lore` CLI, integrators embedding Lore through the Rust API, the C API, or a language SDK, operators running Lore Server, and contributors working on the source. The user-facing folders (`tutorials/`, `how-to/`, `reference/`, `explanation/`) cover the first three; the contributor folders (`developing/internals/`, `developing/decisions/`, `developing/code-standards/`) cover the last.

**Repository conventions.**

- Lore docs live in `docs/` at the repository root and render through a static-site build configured at the project level.
- The top-level `README.md` is the entry point on the GitHub project page; everything else is in `docs/`.
- Lore is contributed to by a multi-organization community. A reader on the public internet must be able to follow every Tutorial and How-To end-to-end. The full open-source posture rules live in [`operational/review-checklist.md`](operational/review-checklist.md).

## Out of scope

These standards don't govern:

- **Per-crate `README.md` files** (such as `lore-server/README.md`).
- **Rust doc comments** (`///` in `.rs` files).
- **Brand- or product-specific style rules** that apply to other products outside Lore. Lore is its own product; rules tied to other products in any sponsor organization's portfolio don't apply unless a Lore doc directly references the product (rare, and discouraged by the evergreen rule in [`canon/language.md`](canon/language.md#third-party-brands)).
- **Consumer-facing content rules** (game-rating tone framings, marketing voice, in-game flavor text). Lore is a developer-and-operator product.
- **Authoring-tool-specific guidance** (proprietary excerpt macros, scheduled publishing, vendor-specific callout types, pagetree navigation, tool-managed metadata fields). Lore docs use plain Markdown idioms portable across static-site generators.
- **Sponsor-organization marketing or branding rules.** These standards cover Lore project conventions, not any sponsor organization's internal voice, marketing language, or brand guidelines.
- **Heavy media production rules** (per-page header, social, and thumbnail meta images; video recording specs; GIF authoring; design-tool templates; screenshot editing for product UI). Lore docs are text-first; code-first; the few rules that apply (lowercase filenames, no text baked into images, image on its own line beneath supporting prose) are in [`operational/accessibility-and-media.md`](operational/accessibility-and-media.md).

See [`docs/README.md`](../../README.md) for the full docs structure.
