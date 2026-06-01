# Filenames

Filename conventions for Lore docs.

## Filename rules

The slug is derived from the filename. Filename rules:

- **Lowercase only.**
- Separate words with a hyphen (`-`). Never underscores, spaces, or other symbols.
- Use keywords readers would actually search. Avoid `page-2.md` or `misc.md`.
- No special characters. Convert `&` to `and`, drop punctuation.

Filename-to-slug transformation examples:

- `Resolving merge conflicts` → `resolving-merge-conflicts.md` → `/resolving-merge-conflicts/`
- `lore commit Reference` → `lore-commit-reference.md` → `/lore-commit-reference/`

## What's derived from the folder

The site build infers these from the file's path.

| Property | Derived from |
| --- | --- |
| Doc type (Tutorial / How-To / Reference / Explanation / Internals / ADR / Code-Standard / Landing) | The folder the file lives in. See `../canon/doc-types.md`. |
| Audience (user-facing / contributor) | Anything under `docs/developing/` is contributor; everything else is user-facing. |

## Navigation

A page doesn't exist for readers until it's wired into navigation. The doc-type folder structure plus a separate journey manifest drive nav generation; the site build picks both up automatically. A page reachable only by direct link is invisible to most readers.

## Search-engine basics

A handful of search-engine rules apply to every Lore page. Most are already enforced by the site build; these are the ones the writer controls.

- **One H1 per page.** The H1 must be text, must include the page's primary keyword, and must not be hidden.
- **Headings:** H2 and H3 for body structure; H4 only when needed; never H5 or H6. See `../canon/format.md#headings`.
- **Filename slug:** lowercase, hyphen-separated, keyword-bearing, no special characters.
- **Anchor text:** descriptive of the destination. Never `click here`, `here`, `read more`. See `../canon/language.md#link-text`.
- **Substantive content must not be hidden** behind collapsed admonitions or behind tabs that require interaction to reveal.
