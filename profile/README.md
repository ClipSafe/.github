# ClipSafe

ClipSafe is a small software company building privacy-first tools for Windows. Everything we ship works locally by default: your clipboard, your notes and anything you choose to share stay on your PC, with no account and no cloud in the middle.

![ClipSafe OG Image](https://clipsafe.app/assets/og.png)

## What we build

| Product | What it is | Ships on |
| --- | --- | --- |
| **ClipSafe** | Clipboard manager: full local history, search, tags, pins and backups | Microsoft Store |
| **ClipSafe Notes** | Markdown notes app: plain `.md` files, local search, folders, tags and preview | Microsoft Store |
| **ClipSafe for VS Code** | Sends the selected code from the editor into ClipSafe | VS Code Marketplace |
| **ClipSafe Drop** | Shares one clipboard item as a temporary encrypted link | Built into ClipSafe |
| **clipsafe.app** | Website, developer docs, pipe API reference and browser tools | Cloudflare Pages |

## ClipSafe

ClipSafe is a privacy-first clipboard manager for Windows. It keeps your entire clipboard history saved locally and searchable, so your data stays private.

### Download

Get ClipSafe free on the Microsoft Store:

https://apps.microsoft.com/detail/9MZF3B649DZV

Windows 10/11 • Free Forever • No Registration Required

### Highlights

- Local-only clipboard history for privacy
- Lightning-fast search across all clipboard items
- Organize with colored tags, or pin what matters
- Background monitoring with zero cloud dependencies
- Quick access to last 9 items with `Ctrl + Shift + 1-9`
- Back up and restore everything in one compressed file
- Works with Windows Snipping Tool
- Lightweight, runs quietly in the background
- Clean, modern interface that matches clipsafe.app

### Features

**Clipboard History**

- Text with source tracking
- Files and file paths
- Images with metadata
- Audio clips
- Animated GIFs
- Link previews

**Search Everything**

- Global search
- Content search
- Source search
- Tag-based search
- Keyboard-first: `Ctrl + F` opens search, `Esc` closes it, `Enter` copies the top result

**Tag System**

- Create your own tags with the built-in color picker or any hex code
- Assign several tags to a single item
- Filter the history by tag instantly
- Manage, rename and delete tags from the Tag Manager

**Pin System**

- Pin any item so it stays at the top
- Browse all pins or only one content type
- Pinned items never expire and survive history clearing

**Keyboard Shortcuts**

- `Ctrl + Shift + 1-9` for quick access
- Choose shortcut behavior (None, Copy, Copy & Paste)
- Works with number row or numpad
- No need to open the app

**Backup & Restore**

- Export history, tags, settings and images into one compressed `.clipsafe` file
- Import it on another PC and pick up exactly where you left off

**Customizable**

- Auto-start with Windows
- Follow system theme
- Pause/resume tracking
- Set history limits
- Minimalistic mode: hide item footers and tighten spacing

**Light & Dark Themes**

- Matches your Windows theme automatically

### Screenshots

![ClipSafe Screenshot 1](https://clipsafe.app/assets/previews/dark/1.png)
![ClipSafe Screenshot 2](https://clipsafe.app/assets/previews/dark/2.png)
![ClipSafe Screenshot 3](https://clipsafe.app/assets/previews/dark/3.png)

## ClipSafe Notes

ClipSafe Notes is a Markdown notes app for Windows. Every note is a plain `.md` file in a folder you choose, so the vault survives uninstalls, sync tools and other editors. Search runs locally with SQLite FTS5, folders and tags filter the same list, and the split view renders Markdown while you write.

Notes and ClipSafe talk over the same local pipe. Sending a note labels it with its tags and the note name, and clicking that label in ClipSafe opens the note again.

### Download

Get ClipSafe Notes free on the Microsoft Store:

https://apps.microsoft.com/detail/9PH5NWCB0KR1

## ClipSafe for VS Code

Select code, press `Ctrl + Shift + Alt + S`, and the snippet lands in ClipSafe with its language, file name, workspace and a `code` tag. The extension talks to the desktop app over a local named pipe, and tells you when the app is not running.

### Install

https://marketplace.visualstudio.com/items?itemName=ClipSafe.clipsafe

## ClipSafe Drop

Drop shares one clipboard item as a temporary link. The item is encrypted on your PC with AES-256-GCM before it leaves, the key stays in the URL fragment and never reaches a server, and the bucket stores ciphertext only. Links expire, and you can add a password.

## How it is built

The Windows apps are .NET 10 WPF with SQLite storage, packaged as MSIX for the Microsoft Store. They talk to each other over a local named pipe, and deep links (`clipsafe://`, `clipsafenotes://`) move between them. The website is a static Cloudflare Pages site with a few Pages Functions for downloads, contact and Drop, backed by R2.

The longer write-up, project by project and with the trade-offs, is in [ARCHITECTURE.md](../ARCHITECTURE.md).

## Principles

- Local first. The apps do not sync, do not require an account and do not send usage data.
- Sharing is explicit and encrypted, never automatic.
- The website counts page views and anonymous download clicks. That is the only analytics in the stack.
- Boring technology that we can still maintain in two years.
- Installs come from the Store or the Marketplace, so updates stay boring too.

## Links

- Website: https://clipsafe.app
- Developer docs: https://clipsafe.app/developers
- Changelog: https://clipsafe.app/changelog
- Microsoft Store: [ClipSafe](https://apps.microsoft.com/detail/9MZF3B649DZV), [ClipSafe Notes](https://apps.microsoft.com/detail/9PH5NWCB0KR1)
- VS Code Marketplace: https://marketplace.visualstudio.com/items?itemName=ClipSafe.clipsafe
- Discord: https://discord.gg/4vfRhCZnHJ
- Reddit: https://www.reddit.com/r/ClipSafe
- Instagram: https://www.instagram.com/clipsafe.app

## Support

If ClipSafe saves you time, consider supporting development:

[![ko-fi](https://ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/J3J11TD0UT)

## Contact

- Email: support@clipsafe.app

## Logo

![ClipSafe Logo](https://clipsafe.app/assets/logo.png)
