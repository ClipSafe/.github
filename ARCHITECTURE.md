# How ClipSafe is built

This is the technical description of the ClipSafe products: what each app is made of, how the pieces talk to each other, how releases go out, and where the architecture is less than ideal. It is written for anyone who wants the detail, and it doubles as a portfolio reference.

## Products at a glance

| Product | What it is | Stack | Ships on |
| --- | --- | --- | --- |
| ClipSafe | Clipboard manager for Windows | .NET 10, WPF, EF Core 10, SQLite, MSIX | Microsoft Store |
| ClipSafe Notes | Markdown notes app for Windows | .NET 10, WPF, Markdig, NativeMarkdown.Wpf, SQLite FTS5, MSIX | Microsoft Store |
| ClipSafe for VS Code | Editor extension that sends code to ClipSafe | TypeScript, esbuild, VS Code API | VS Code Marketplace |
| clipsafe.app | Website, docs and small APIs | Static HTML/CSS/JS, Cloudflare Pages Functions, R2 | Cloudflare Pages |

## Decisions that shape everything

**Local first.** The apps keep their data on disk in the user's profile and never upload it. No accounts, no sync service, no telemetry inside the desktop apps.

**Explicit sharing.** The only network feature in the desktop apps is ClipSafe Drop, and it runs only when a user shares one item on purpose. The payload is encrypted before it leaves the machine.

**Integrate over local pipes, not plugins.** Other tools push snippets into ClipSafe through a named pipe. That keeps the clipboard database closed to readers while still letting editors and apps contribute items.

**Boring technology.** WPF, EF Core, SQLite, static HTML, Cloudflare Pages. Nothing here needs a preview runtime, a build farm or a cluster, which matters when one person maintains all of it.

## ClipSafe for Windows

### Stack

- .NET 10, C#, WPF (`net10.0-windows10.0.19041.0`, x64)
- MVVM with CommunityToolkit.Mvvm, DI with Microsoft.Extensions.DependencyInjection and Configuration
- EF Core 10 with the SQLite provider, code-first migrations and a pragma interceptor for WAL settings
- Win32 interop for clipboard notifications, global hotkeys, window and process lookups
- MSIX packaging through a Windows Application Packaging (WAP) project

### How the solution is split

| Project | Responsibility |
| --- | --- |
| `ClipSafe` | Views, view models, controls, theme resources, app startup, services (clipboard, theme, versioning, deep links) |
| `Core` | Plain models and interfaces, tag and item types, the Drop crypto format, changelog types |
| `Infrastructure` | EF Core storage, clipboard type handlers, Windows services (pipes, hotkeys, settings, storage paths, image processing), Win32 helpers |
| `ClipSafe.Package` | The MSIX project: manifest, assets, store identity |
| `ClipSafe.Tests` | xUnit tests for the pipe server, Drop crypto and links, receipts, forwarding and storage |

`Core` has no WPF reference, so the domain and the Drop format can be tested without a window. `Infrastructure` holds everything that touches Windows or the database. The app project stays mostly UI.

### Clipboard capture

A hidden window listens for `WM_CLIPBOARDUPDATE`. When a change arrives, the clipboard processor inspects the formats and hands the data to one of the type handlers: text, image, dropped files, audio or a custom payload. The source app is resolved from the foreground window and process, which is what produces the `Source` label on each item (for example `Visual Studio` or `VSCode/my-repo`).

Snipping Tool has its own path. It writes an unpublished temp file while the user is still editing, so the app watches for that specific window and updates the existing item instead of creating a new one on every edit.

Images are copied to disk and referenced from the database, with an animated GIF path that reuses the same file. Link previews are fetched with HtmlAgilityPack and cached.

### Storage

SQLite through EF Core. The schema is small: clipboard items, tags, and the join table between them. Migrations cover the initial schema, source labels, indexing, pins and tags.

Details that matter in practice:

- A connection interceptor applies WAL journaling, normal synchronous mode, an in-memory temp store, a 256 MB memory map, a 20 MB cache, a 5 second busy timeout and `PRAGMA optimize` on every connection. Clipboard writers and the UI both hit the same file, so the lock behaviour has to be predictable.
- Value converters turn the strongly typed ids and the source struct into SQLite-friendly values.
- Indexes exist on timestamp, type, source, content, pinned flag and pinned timestamp, because every list view filters or sorts on one of them.
- Paged reads use a cursor (timestamp or pinned timestamp) plus a search predicate built from the active filters, so the UI can keep loading pages without loading the table.
- Tag creation is normalised and case-insensitive, and the storage layer attaches tags to items without inserting duplicates.

### UI and shell integration

One window with two modes: a full history window and a compact floating window, both sharing the same view models. The list uses a virtualising wrap panel, because a history with thousands of items must scroll without allocating everything. Themes are resource dictionaries selected at runtime, and every screen follows the same design tokens.

Shell integration covers a tray icon, optional auto-start, Windows toast notifications, and configurable global hotkeys (`Ctrl + Shift + 1-9`) that can copy or copy and paste the last items without opening the window.

### The local pipe

ClipSafe runs a named pipe server on `\\.\pipe\ClipSafePipe`. The protocol is deliberately simple: one JSON line in, one JSON line out, one connection per request.

- Request: `{ "type": "add_snippet", "requestId": "...", "payload": { ... } }`
- Reply: `{ "RequestId": "...", "Success": true, "Message": "Ok", "Data": null }`
- Commands: `add_snippet`, `import_drop`, `ping`, `get_version`
- Protocol version: `1.0.0`, with a version handshake so clients can warn about mismatches

There is no read command, no list and no search. That is a security decision: a program can add a snippet, it cannot read the user's history. The server also guards against oversized payloads, applies a short client timeout, and strips the UTF-8 BOM that .NET writes before its first reply (a detail that trips up every client at least once).

### ClipSafe Drop

Drop shares one item through a temporary link:

1. The item is serialised into a `CSD1` envelope and encrypted with AES-256-GCM. An optional password derives the key with PBKDF2-HMAC-SHA256.
2. The key stays in the URL fragment, which browsers never send to a server.
3. The app posts the ciphertext to the website API, which stores it in R2 and returns an id.
4. The resulting link, `clipsafe://drop/<id>#k=<key>`, opens the app, which fetches the item and decrypts it locally.
5. Link receipts (id, expiry, whether a password was used) are stored locally so Settings can list and delete them.

### Backup, updates and release

Backup exports the database, settings and images into one compressed file and restores them on another machine. The in-app "What's new" panel reads a bundled `changelog.json` that is versioned with the app.

Release flow:

1. Bump the version in `ClipSafe.csproj` and in the package manifest.
2. Add the user-facing entry to `changelog.json`.
3. Run `publish.py`, which restores with the same properties the WAP publish uses (self-contained, x64, `win-x64`) and then builds the MSIX. Mismatched restore properties here produce a package that fails at startup, so the script exists to make that repeatable.
4. Submit the MSIX through Partner Center.

GitHub Actions builds the solution on every push and pull request (`dotnet-desktop.yml`, Debug and Release on `windows-latest`).

## ClipSafe Notes

### Stack

- .NET 10, C#, WPF, MVVM with CommunityToolkit.Mvvm
- Markdig for parsing, rendered by the in-house [NativeMarkdown.Wpf](https://github.com/ClipSafe/NativeMarkdown.Wpf) library
- Microsoft.Data.Sqlite with an FTS5 virtual table for search
- MSIX packaging through its own WAP project, x86/x64/arm64 bundle

### Vault and index

The vault is a normal folder. Every note is a `.md` file with a tiny front matter block for title, tags, favourite and timestamps, plus an `attachments` folder for pasted and dropped images and a `.trash` folder for deletions.

Files are the source of truth. The SQLite index is disposable: on startup the vault is scanned, notes are upserted, and search runs against an FTS5 table that mirrors title, body and tags. Deleting the index costs a rescan, not data.

### Preview

Markdown is parsed with Markdig and rendered straight into a WPF `FlowDocument` by the in-house
[NativeMarkdown.Wpf](https://github.com/ClipSafe/NativeMarkdown.Wpf) library.
The viewer keeps the visual block for every top-level Markdown block and rebuilds only the blocks whose
source changed. Typing redraws the edited paragraph instead of reloading the document, which is what
removed the flicker and the debounce timer the earlier preview relied on. Color emoji are drawn as vector
art from the system emoji font, so they stay sharp at any size and add nothing to the install size.

Rendering inside the app also removed a compositor quirk: there is no browser surface painting above WPF
content, so the startup overlay no longer has to wait for the preview to move out of the way.

### Integration with ClipSafe

Notes sends snippets over the same ClipSafe pipe. Each item carries the note name as its workspace, a `note` tag, and the note's own tags. ClipSafe shows the source as `Notes/<note name>` and makes it clickable.

The reverse direction is a custom URI scheme, `clipsafenotes://note/<name>`. The app keeps a single instance with a mutex, and a second process hands the note name to the running window over a small named pipe instead of opening a second copy.

### Release

Same pattern as ClipSafe: version in the csproj and manifest, `changelog.json` entry, MSIX build, Partner Center submission. Tests cover the vault, front matter parsing, settings and the index (xUnit, `ClipSafe.Notes.Tests`).

## ClipSafe for VS Code

TypeScript extension built with esbuild, linted with ESLint and tested with the VS Code test CLI (mocha) against an Electron instance.

The extension is a pipe client: it validates the payload size, connects to `\\.\pipe\ClipSafePipe`, sends one `add_snippet` request and reads one reply. It retries once when the connection is refused (the app rebinds its listener), times out after four seconds, strips the BOM, and maps failures to readable messages (app closed, timeout, rejected, protocol mismatch).

It contributes one command, one context-menu entry, one keybinding (`Ctrl + Shift + Alt + S`) and two settings (size confirmation threshold and success notifications), plus a status bar item that reports whether ClipSafe is reachable. Snippets go out tagged `code` with language, file name and workspace, which is what makes them sortable in ClipSafe.

Packaging is `npm run package` (type check, lint, production esbuild bundle) followed by a `.vsix` upload to the Marketplace.

## Website and cloud

### Static site

clipsafe.app is plain HTML, CSS and JavaScript on Cloudflare Pages. No framework, no bundler, no build step. Pages is deployed automatically from `main`.

- Shared header and footer are injected at runtime from `partials/` so every page keeps one copy of the navigation.
- One global stylesheet holds the design tokens and shared components; pages add their own small stylesheet.
- Metadata is hand-written: canonical extensionless URLs, Open Graph images, JSON-LD per page, `sitemap.xml`, `robots.txt` and an `llms.txt` summary.
- Google Analytics counts page views. AOS handles scroll animations, with reduced-motion respected.

Browser tools (secret scan, HTML to Markdown, paste cleaner, line tools and others) run entirely client side.

### Pages Functions

Four small Functions, written as ES modules:

| Route | What it does |
| --- | --- |
| `/download` | Logs the click and redirects to the correct Store listing with UTM parameters |
| `/api/contact` | Verifies a Turnstile token and sends the message through Resend |
| `/api/drop` | Creates an encrypted Drop, stores ciphertext in R2 |
| `/api/drop/<id>` | Reads or deletes a Drop with its delete token |

Download tracking writes to a Workers Analytics Engine dataset (`clipsafe_download_clicks` through the `DOWNLOAD_CLICKS` binding) with the placement, country and product as blobs. Without the binding the redirect still works and logs to the console.

The contact form checks a honeypot field, enforces length limits, verifies the request origin, validates the topic against an allowlist, and verifies Turnstile before calling Resend. Secrets (`TURNSTILE_SECRET`, `RESEND_API_KEY`, sender and recipient overrides) live in Pages settings, never in the repository.

### Drop API and storage

The Drop endpoints never see plaintext. They validate the payload shape and size, enforce a short rate limit window in memory, and write the ciphertext to a private R2 bucket under `drops/<id>`. Deletion requires the hashed delete token. Expiry is part of the stored metadata, and a lifecycle rule can clean the bucket on a schedule.

`_headers` applies a strict CSP, `no-store`, `no-referrer` and `noindex` to the viewer and the API. The viewer page loads no third-party assets at all.

### Routing, redirects and email

- `_routes.json` limits Functions to `/download*` and `/api/*`, so every other request is a static asset and stays free.
- `_redirects` proxies `/d/<id>` to the viewer without consuming a Function request, and hides the Function sources.
- Redirect Rules send `notes.clipsafe.app` to the Notes page while the canonical URL stays on `clipsafe.app`.
- Cloudflare Email Routing delivers `support@clipsafe.app`. Outbound contact mail goes through Resend, because Cloudflare Email Sending to arbitrary recipients requires a paid Workers plan.

### Tests

`node:test` suites cover the Drop format (a shared test vector, so the C# and JavaScript implementations stay compatible) and the Drop API handlers with a fake R2 binding. They run with plain Node, no test framework.

## Testing and CI

| Area | Tests | Runs in CI |
| --- | --- | --- |
| ClipSafe | xUnit: pipe protocol, Drop crypto and links, receipts, forwarding, storage | Yes, via GitHub Actions |
| ClipSafe Notes | xUnit: front matter, index, settings, vault | No, local |
| VS Code extension | VS Code test CLI plus unit tests for the pipe client | No, local |
| Website | `node:test` for Drop vectors and API handlers | No, local |

## Versioning and changelogs

Each product carries its own version and its own changelog file:

- The desktop apps ship a `changelog.json` next to the executable, and the app shows the entry for the running version as "What's new".
- The website has `changelog.json` for ClipSafe and `changelog-notes.json` for Notes, rendered on one page with product tabs and deep links (`/changelog#notes`).
- The extension keeps `CHANGELOG.md` in the Marketplace format.


## Skills this covers

**Languages:** C#, TypeScript, JavaScript, SQL, HTML/CSS, Python (release automation)

**Desktop:** WPF, XAML, MVVM, .NET 10, Win32 interop, MSIX and Windows Application Packaging, Microsoft Store submission

**Data:** SQLite, EF Core (migrations, value converters, indexes, paging), FTS5 full text search

**Integration and security:** Windows named pipes, JSON line protocols with versioning, custom URI schemes and activation, single instance handoff, AES-256-GCM, PBKDF2, CSP and security headers, Turnstile, scoped secrets

**Web and edge:** Cloudflare Pages, Pages Functions on the Workers runtime, R2 object storage, Analytics Engine, Email Routing, redirect rules, caching and header policy, JSON-LD and technical SEO

**Tooling and quality:** Git and GitHub, GitHub Actions, MSBuild and Visual Studio automation, xUnit, `node:test`, VS Code Test CLI, ESLint, esbuild, Markdig, NativeMarkdown.Wpf

**Product work:** product design and copy, release notes, store listings and screenshots, support, privacy and security decisions end to end
