# Joomla Internal Linking — Always Author Non-SEF URLs

**The rule, in one line:**

> **Never hard-code a SEF path in Joomla content. Author every internal link as a non-SEF `index.php?option=…` URL and let Joomla's router generate the pretty URL at render time.**

This applies to links written *inside Joomla* — article bodies, `mod_custom` modules, footers, category descriptions, custom HTML anywhere. It is the difference between a link that survives a site reorganisation and one that silently 404s.

---

## Why

A SEF path like `/entertainment/clean-jokes/` is **derived output**, not an address. Joomla builds it at render time from the article alias, the category path, the menu structure, and the active SEF/router settings. Hard-coding it freezes a snapshot of all four.

Any of these silently breaks every hard-coded SEF link pointing at that content:

| Change | Effect on a hard-coded SEF link |
|---|---|
| Article or category **alias** edited | 404 |
| Article **moved** to another category | 404 (path segment changes) |
| **Menu item** renamed, moved, or its alias changed | 404 |
| Menu item **deleted** and recreated | 404 |
| **SEF URLs** toggled off, or *Remove IDs from URLs* toggled | every link breaks at once |
| **Language prefix** added (multilingual go-live) | `/en/…` prefix missing |
| A **URL-manager extension** (4SEO, sh404SEF) re-generates routes | stale alias cached, 404 |

The non-SEF form encodes **identity** (`id=3918`), not a path. IDs do not change when content is renamed or moved. Joomla resolves identity → current SEF path on every page render, so the link is correct by construction.

There is no SEO downside: the visitor and the crawler both receive the fully-formed SEF URL in the HTML. The non-SEF form exists only in the stored content.

---

## How it works

`plg_system_sef` (**System - SEF**) post-processes the rendered page and rewrites any `href` beginning with `index.php` into the site's current SEF form. **This plugin must be enabled** — it is by default.

You author:

```html
<a href="index.php?option=com_content&amp;view=article&amp;id=3918&amp;catid=160">Privacy Policy</a>
```

The browser receives:

```html
<a href="/policies/privacy-policy">Privacy Policy</a>
```

Rename the article to `privacy-and-your-data` and the output becomes `/policies/privacy-and-your-data` on the next render. The stored link never changed.

---

## The formats

Use `&amp;` (not bare `&`) — you are writing HTML.

### Article

```
index.php?option=com_content&amp;view=article&amp;id=<articleId>&amp;catid=<catId>
```

Include `catid` — it lets the router build the category-path segments. Without it Joomla still resolves the article but may route it through a less specific menu item.

### Article that has its own menu item

```
index.php?option=com_content&amp;view=article&amp;id=<articleId>&amp;Itemid=<itemId>
```

`Itemid` pins the link to a specific menu item. This controls **which menu branch highlights as active**, **which template style applies**, and **which module assignments fire**. Add it whenever a menu item exists for the destination.

### Category — list or blog layout

```
index.php?option=com_content&amp;view=category&amp;id=<catId>&amp;Itemid=<itemId>
index.php?option=com_content&amp;view=category&amp;layout=blog&amp;id=<catId>&amp;Itemid=<itemId>
```

**Match `layout=blog` to the menu item's own layout.** If the menu item is a Category Blog and you omit `layout=blog`, you can land on a differently-rendered page than the menu produces.

### A menu item, whatever it points at

```
index.php?Itemid=<itemId>
```

Terse and robust. Prefer the fuller `option=…` form when you know the destination — it still resolves if the menu item is later deleted.

### Non-com_content components

Use that component's own non-SEF route, plus `Itemid`:

```
index.php?option=com_monthlyarchive&amp;view=archive&amp;Itemid=106408
index.php?option=com_contact&amp;view=contact&amp;id=4&amp;Itemid=101
```

### Home page

Use `index.php` — **not** `/`. Joomla rewrites it to the correct root, including any language prefix.

---

## Finding the values

- **Article id / catid** — the editor's **Article** toolbar button (or CMS Content → Articles) inserts exactly this format. Easiest path: use the button, then check the source.
- **Itemid** — Menus → the item; the ID column, or `id=` in the edit URL.
- **Via API** — `list_menu_items` returns each item's `id` and its raw `link` value. The stored `link` **is** the non-SEF URL; append `&Itemid=<id>` and you are done.

That last point is the shortcut: **a menu item's `link` column already holds the canonical non-SEF URL for its destination.** Read it rather than reconstructing it.

---

## Verifying

Author the non-SEF link, then confirm two things on the rendered page:

```bash
# 1. No raw index.php should survive into the output — if it does, plg_system_sef is off
curl -s https://example.org/ | grep -o 'href="index.php[^"]*"'

# 2. The links resolved to real pages
curl -s -o /dev/null -w "%{http_code}" https://example.org/policies/privacy-policy
```

In a headless browser, sweep every link in the region you changed:

```js
const links = [...new Set([...document.querySelectorAll('.my-footer a[href]')]
  .map(a => a.getAttribute('href')))];
const bad = [];
for (const h of links) {
  const r = await fetch(h, { redirect: 'follow' });
  if (r.status !== 200) bad.push({ h, status: r.status });
}
return { total: links.length, broken: bad,
         stillRaw: links.filter(h => h.includes('index.php')) };
```

`stillRaw` must be empty. If it is not, **System - SEF** is disabled and every one of those links is being served to visitors as an ugly `index.php?…` URL.

---

## Gotchas

1. **`&` vs `&amp;`** — in HTML content write `&amp;`. A bare `&` may be mangled by the editor or produce invalid markup. Joomla parses both, but the WYSIWYG round-trip is safer with entities.
2. **The WYSIWYG editor can rewrite links.** Some editor configurations "helpfully" convert URLs on save. After a save through the editor, re-check that the non-SEF form survived.
3. **Menu-item type "URL"** does *not* get SEF-rewritten — it is emitted verbatim. For internal destinations use an **Internal Link** menu item type instead.
4. **Absolute URLs are never rewritten.** `https://example.org/some/path` is left alone by `plg_system_sef`. Author internal links **relative and non-SEF**, so keep them starting with `index.php`.
5. **Caching.** With page caching or a CDN in front, an alias change will not visibly propagate until the cache clears. The stored link is still correct; the cache is stale.
6. **A missing `Itemid` is not an error, just less precise.** The router picks a best-match menu item, which may highlight the wrong menu branch or apply a different template style.

---

## Anti-patterns

```html
<!-- ❌ hard-coded SEF path — breaks on any alias/category/menu change -->
<a href="/entertainment/clean-jokes/">Clean Jokes</a>

<!-- ❌ absolute URL to own site — never rewritten, breaks on domain/protocol change -->
<a href="https://example.org/entertainment/clean-jokes/">Clean Jokes</a>

<!-- ❌ root-relative home link — loses language prefix on multilingual sites -->
<a href="/">Home</a>

<!-- ✅ non-SEF, router-resolved -->
<a href="index.php?option=com_content&amp;view=category&amp;layout=blog&amp;id=73&amp;Itemid=100495">Clean Jokes</a>
```

---

## Real-world confirmation

Applied on **cybersalt.org** (Joomla 6.1.2), 2026-08-05, when building a site footer and a home portal page. 27 internal links across a `mod_custom` footer and a featured article were converted from hard-coded SEF paths to non-SEF `index.php?option=…` form.

Rendered output confirmed the rewrite — `index.php?option=com_content&view=article&id=3918&catid=160` came back to the browser as `/policies/privacy-policy` — and all 27 resolved HTTP 200 with zero raw `index.php` surviving into the HTML.

The motivating problem was concrete: that same site had category pages returning HTTP 500, and separately a set of hard-coded SEF links in content whose destinations had drifted. Identity-based links would have survived both.

---

## Related

- `JOOMLA5-COMPONENT-ROUTING.md` — building a `RouterBase` so *your own* component produces good SEF URLs (the other side of this coin)
- `REGULAR-LABS-SOURCERER-NOTES.md` — embedding executable PHP in content, e.g. a dynamic copyright year
