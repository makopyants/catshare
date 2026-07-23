# Catshare Agent Guide

This is the canonical technical guide for coding agents working in this
repository. Read it before changing code, content, media, or deployment
settings.

## Project purpose

Catshare is a small public adoption landing page for three rescued kittens in
Belgrade. The production site is:

`https://makopyants.github.io/catshare/`

The page is intentionally simple:

- static HTML, CSS, JavaScript, fonts, and images;
- no backend, build step, package manager, or framework;
- one external runtime service: the privacy-focused Umami Cloud analytics
  beacon;
- Russian and Serbian Cyrillic content;
- responsive kitten galleries, a lightbox, contact links, and social sharing;
- deployment from the repository's `main` branch through GitHub Pages.

Keep the site usable as a plain static site unless a task explicitly calls for
an architectural change.

## Repository map

| Path | Role |
| --- | --- |
| `index.html` | The application. Markup, styles, configuration, translations, and all runtime logic are inline. |
| `board.html` | Legacy URL redirect to the site root. |
| `README.md` | Russian instructions for the human maintainer, mainly media and contact updates. |
| `media/kroshka/` | Gallery assets for kitten `k1`, "Kroshka". |
| `media/blondinka/` | Gallery assets for kitten `k2`, "Blondinka". |
| `media/pacan/` | Gallery assets for kitten `k3`, "Pacan". |
| `media_originals/` | Ignored local source media such as HEIC and MOV files. Never commit it. |
| `sharing/` | Prepared social-network artwork, selected photos, and posting copy. It is not loaded by the site. |
| `qr/` | PNG and SVG QR codes pointing to the production URL. |
| `wood.jpg`, `parchment.jpg` | Repeating visual textures used by the CSS. |
| `medallion.jpg`, `favicon.png`, `apple-touch-icon.png`, `og-cover.jpg` | Header, browser, device, and social-preview assets. |
| `fonts/monomakh.otf` | Local display font. Its license is in `fonts/OFL.txt`. |
| `.nojekyll` | Tells GitHub Pages to serve the repository as static files without Jekyll processing. |

`.claude/settings.local.json` is ignored by the user's global Git
configuration. It is machine-specific and must not be treated as shared
project configuration.

## Runtime architecture

`index.html` contains four layers in one file:

1. `<head>` metadata and links to static assets.
2. Inline CSS for the wooden board/parchment design, responsive cards,
   carousels, share controls, and lightbox.
3. A small HTML shell: fixed header, empty card container, footer sheet, and
   lightbox.
4. Inline JavaScript that supplies data and renders the interactive UI.

The `<head>` also loads the deferred Umami Cloud tracker. It is restricted to
the production `makopyants.github.io` hostname, honors Do Not Track, and
collects Core Web Vitals. The public website ID is configuration, not a secret.

The important JavaScript configuration blocks are:

- `CONTACTS`: public Telegram, phone, WhatsApp, and Viber values.
- `SHARE_URL`: canonical URL used by share actions.
- `MEDIA_DIR`: root gallery directory.
- `KITTENS`: stable kitten keys, media directories, and fallback filenames.
- `OWNER` and `REPO`: repository queried through the public GitHub API.
- `IMG_EXT` and `VID_EXT`: accepted browser-ready file extensions.
- `I18N`: all Russian (`ru`) and Serbian (`sr`) UI and profile text.
- `ICONS` and `PAW`: inline SVG path/template data.

### Gallery loading sequence

The gallery has a deliberate two-stage loading strategy:

1. `buildCards(null)` immediately renders `KITTENS[*].fallback`, avoiding an
   empty page and layout jump.
2. `loadFileLists()` requests the recursive tree for the remote `main` branch:
   `https://api.github.com/repos/<OWNER>/<REPO>/git/trees/main?recursive=1`.
3. Supported files directly below `media/<kitten-dir>/` are grouped by
   directory.
4. If the remote lists differ from the fallbacks, all cards are rebuilt from
   the API result. A failed or rate-limited request leaves the fallback UI
   intact.

Files are sorted with images first and videos second, then by
locale-aware/numeric filename order. Spaces and non-ASCII characters are
supported because media filenames are passed through `encodeURIComponent`.

Every configured kitten directory must contain at least one supported file.
Carousel and lightbox navigation assume a non-empty list.

### Language behavior

`setLang()` updates the document language, title, visible translations, and
active switch button. The selected language is stored under the `localStorage`
key `lang`.

Language selection precedence is:

1. saved `localStorage` preference;
2. Russian if any browser language starts with `ru`;
3. Serbian for every other browser.

Both language objects must retain the same keys. Kitten text keys follow the
stable pattern `<key>title` and `<key>text`, for example `k1title` and `k1text`.
The Open Graph metadata in `<head>` is static and is not changed by the
language switch.

### Navigation and sharing

- Kitten IDs `k1`, `k2`, and `k3` are public URL anchors.
- A matching hash scrolls to the card and runs a highlight animation.
- Card sharing appends the relevant anchor to `SHARE_URL`.
- General sharing uses the root `SHARE_URL`.
- Telegram, WhatsApp, and Threads use web intent URLs.
- Contact controls use Telegram/WhatsApp web URLs, a `viber:` URI, and a
  `tel:` URI.
- The lightbox supports click controls, Escape, and left/right arrow keys.

## Source-of-truth rules

Use the following source for each kind of change:

| Change | Source of truth |
| --- | --- |
| Page and kitten copy | Both locales in `I18N` in `index.html` |
| Initial document/social metadata | `<title>` and `<meta>` elements in `index.html` |
| Public contacts | `CONTACTS` in `index.html` |
| Analytics website ID, domain, DNT, and performance settings | Umami `<script>` attributes in `index.html` |
| Analytics event names and properties | `trackEvent()` call sites in `index.html` |
| Gallery membership and resilient fallback | Files under `media/<dir>/` **and** matching `KITTENS[*].fallback` entries |
| Kitten-to-directory mapping | `KITTENS` in `index.html` |
| Production/repository identity | `SHARE_URL`, `OWNER`, `REPO`, Open Graph URL, `board.html`, `README.md`, `sharing/TEXTS.md`, and QR assets as applicable |
| Visual styling and responsive behavior | Inline `<style>` in `index.html` |
| Campaign copy and prebuilt posts | `sharing/TEXTS.md` |

Do not assume README prose precisely describes current behavior. In
particular:

- the GitHub API normally discovers media automatically, but fallback arrays
  still need to be kept synchronized for first render and API failures;
- empty `CONTACTS` values are **not** currently filtered out by
  `renderContacts()`. Implement conditional rendering if that behavior is
  required instead of merely blanking a value.

## Common change workflows

### Change text or add a translation

1. Edit the corresponding entries in both `I18N.ru` and `I18N.sr`.
2. Preserve all keys referenced by `data-i18n` attributes and kitten keys.
3. Check the fixed header at mobile width; long titles are clamped to two
   lines.
4. Update static `<head>` metadata separately when the public preview text
   should change.
5. Update `sharing/TEXTS.md` only when the campaign copy must stay aligned.

Keep Russian and Serbian text in their existing scripts unless the requested
content change says otherwise.

### Change contacts

Edit `CONTACTS` near the beginning of the script. Values are public in the
delivered source, so never put tokens, private credentials, or sensitive
configuration there.

Check service-specific formatting:

- `telegram`: username without `@`;
- `phone`: international number, currently including `+`;
- `whatsapp`: international digits without `+`;
- `viber`: international number accepted by the `viber:` link.

The displayed phone formatting regex is currently specialized for the Serbian
number shape. Adjust it if the country or number structure changes.

### Add, remove, rename, or reorder gallery media

1. Put browser-ready files in the correct `media/<kitten-dir>/` directory.
2. Use only supported image extensions (`jpg`, `jpeg`, `png`, `webp`, `gif`,
   `avif`) or video extensions (`mp4`, `webm`, `m4v`).
3. Update the matching `KITTENS[*].fallback` array to exactly reflect the
   directory.
4. Use numeric filename prefixes such as `01-` and `02-` when an explicit
   order is needed. Images will always precede videos.
5. Test the fallback path as well as the GitHub API path.

Raw HEIC and MOV files are not reliable cross-browser delivery formats.
Retain raw files only in ignored `media_originals/`, then create optimized
browser-ready copies. The conversion examples in `README.md` use macOS `sips`
for HEIC and `ffmpeg` for MOV. Do not overwrite or delete originals unless the
user explicitly requests it.

GitHub rejects individual files above 100 MB, but practical web media should
be much smaller. Preserve aspect ratio and avoid increasing resolution during
conversion.

Local testing has one non-obvious behavior: the page still queries the
production repository's `main` tree. If local media differs from remote media,
the asynchronous response can rebuild the local cards with the remote list.
When testing unpublished media, block the GitHub API request in browser
developer tools or temporarily disable that request in an uncommitted local
change. Never commit a test-only API bypass.

### Add or remove a kitten

This is more than a directory change. Update all of the following together:

- `KITTENS`, preserving unique `key` and `dir` values;
- matching title/text entries in every `I18N` locale;
- the hash validation in `scrollToHashCard()` (currently `/^#k[123]$/`);
- fallback media and the new media directory;
- any public copy, Open Graph artwork, QR/campaign material, and sharing text
  affected by the number of kittens.

The CSS grid itself is responsive and does not require a fixed card count.

### Change analytics

Umami automatically records production pageviews. The inline
`pendingAnalyticsEvents` queue preserves early events until the deferred
tracker is ready; `trackEvent()` safely becomes a no-op from the user's
perspective if the tracker is blocked.

Current custom events are:

| Event | Properties | Meaning |
| --- | --- | --- |
| `share_click` | `platform`, `scope`, `lang` | A site- or kitten-level sharing intent was opened. |
| `contact_click` | `channel`, `lang` | A contact destination was opened. |
| `kitten_interest` | `kitten`, `media_type` | A gallery item was opened in the lightbox. |
| `engaged` | `kind`, `lang` | The visit reached 30 seconds or the footer became visible. |
| `language_change` | `from`, `to` | The visitor changed the UI language. |
| `media_api_error` | `reason` | GitHub media-tree discovery failed and fallback media remained active. |

Keep names and property values low-cardinality. Never send contact values,
filenames, full URLs containing private data, free-form user input, or other
personally identifying information.

`shareLinks()` adds attribution to destinations shared from the page:

- `utm_source`: destination platform;
- `utm_medium=share`;
- `utm_campaign=catshare_2026`;
- `utm_content`: `site` or the stable kitten key.

The printed bilingual poster QR codes still point to the untagged production
root. Such visits appear as direct traffic and cannot be identified as QR scans
with certainty. Do not relabel all direct traffic as QR traffic.

If the Umami provider, website ID, production host, campaign name, or event
schema changes, update this section and validate both analytics-disabled and
analytics-enabled behavior. Session replay, heatmaps, visitor identification,
and personal-data collection are intentionally out of scope.

### Change the repository, branch, or public URL

The current implementation hard-codes `main` in the GitHub tree request and
hard-codes the production URL in several files. Search the entire repository
before changing hosting:

```sh
rg -n 'makopyants|catshare|github\\.io|git/trees/main'
```

Update `OWNER`, `REPO`, `SHARE_URL`, Open Graph metadata, the canonical redirect
in `board.html`, documentation/campaign links, and QR codes as needed. Prefer
relative URLs for runtime assets so GitHub Pages project-path hosting continues
to work.

## Local development

No installation is required. Serve the repository root over HTTP:

```sh
python3 -m http.server 8000
```

Then open:

`http://localhost:8000/`

Do not add a package manager, lockfile, bundler, or framework only to perform a
small edit. There is currently no automated test, lint, or formatting suite.

## Validation checklist

Use checks proportional to the change. For normal UI, content, or media work:

1. Run `git diff --check`.
2. Review `git status --short` and ensure raw media, `.DS_Store`, local Claude
   settings, and unrelated files are not staged.
3. Serve the root locally instead of relying on `file://`.
4. Check desktop and a narrow mobile viewport.
5. Switch Russian and Serbian, reload, and verify language persistence.
6. Check all three cards, counters, previous/next controls, swipe/scroll, and
   image/video lightbox behavior.
7. Test `#k1`, `#k2`, and `#k3`, including highlight and post-load behavior.
8. Inspect the browser console and network panel for JavaScript errors and
   missing assets.
9. Verify contact and share destinations without sending a message or creating
   a post. Confirm shared destinations retain their kitten hash and expected
   `utm_*` parameters.
10. When media changes, verify that each fallback list exactly matches its
    directory and test with the GitHub API unavailable.
11. When metadata or hosting changes, check `board.html`, the Open Graph image,
    favicon/touch icon, production URLs, and QR destinations.
12. For analytics changes, confirm pageviews and each affected event in Umami
    realtime data, check for duplicate events, confirm localhost is excluded,
    and verify that blocking the tracker does not break the UI.

Do not perform real social posts, calls, messages, or deployments unless the
user explicitly asks for those external actions.

## Editing principles

- Keep changes narrow and preserve the no-build deployment model.
- Treat `index.html` as the application source, not generated output.
- Preserve relative asset paths and GitHub Pages subdirectory compatibility.
- Keep fallback arrays synchronized with checked-in gallery files.
- Preserve stable `k1`/`k2`/`k3` anchors unless a migration is intentional.
- Preserve the mobile header media-query ordering noted in the CSS comment;
  moving it before base language-switch rules can reintroduce overrides.
- Do not commit secrets, raw media, machine-local settings, or `.DS_Store`.
- Do not modify prebuilt sharing artwork merely because page styling changed;
  those assets are a separate campaign deliverable.
- Update this guide when architecture, data flow, deployment, or validation
  requirements materially change.
