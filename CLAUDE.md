# Image Hider (Chrome extension, Manifest V3)

Toolbar-click toggle that covers every image on the current tab with white, keeping layout intact.
`docs/SPEC.md` is the source of truth for behaviour. If code and SPEC disagree, stop and ask.

## Commands
- Load locally: chrome://extensions > Developer mode > Load unpacked > select `extension/`
- Unit tests: `npm test` (node:test, no browser)
- E2E tests: `npm run test:e2e` (Playwright, Chromium with the unpacked extension)
- Lint: `npm run lint`
- Package for Chrome Web Store: `npm run package` -> `dist/image-hider-<version>.zip`
- Serve fixtures manually: `npm run fixtures` (http://localhost:8080)

## Layout
- `extension/` shipped as-is, nothing outside it goes into the zip
  - `manifest.json`, `background.js` (service worker, ES module)
  - `content/hider.js`, `content/hider.css`
  - `lib/` pure logic shared by SW and tests (state keys, URL checks, media-iframe allowlist)
  - `icons/`
- `tests/unit`, `tests/e2e`, `tests/fixtures/*.html`
- `scripts/` dev-only tooling (packaging, icon generation, fixture server)
- Docs and their audiences (never duplicate content between them, link instead):
  - `README.md` humans installing or cloning. Structure fixed by SPEC section 13.
  - `docs/SPEC.md` behaviour contract. `docs/PLAN.md` implementation plan.
  - `CLAUDE.md` this file, for the agent only.

## Hard rules
- Vanilla JS (ES2022), no bundler, no runtime dependencies. devDependencies only for tests/lint/packaging.
- No remote code, no eval, no network requests from the extension (CWS policy).
- Service worker is ephemeral: no module-level state. Persist in `chrome.storage.session`.
- Register all SW event listeners synchronously at top level.
- Permissions are fixed by SPEC "Permissions". Adding any permission needs my approval first.
- Hiding must never change an element's box size or position. Never modify `src`, `srcset`, `poster` or inline styles; use `data-ih-*` attributes + injected CSS.
- Showing must restore the page exactly (remove every attribute/style/listener the extension added).

## Gotchas
- `chrome.permissions.request` must run inside the `action.onClicked` gesture, before any `await`.
- Extension reload in dev unregisters dynamic content scripts: reconcile on `runtime.onInstalled` and `runtime.onStartup` from `chrome.permissions.getAll()`.
- chrome://, edge://, the Chrome Web Store and view-source: pages cannot be scripted. Handle gracefully.
- Content scripts do not know their tab id. Ask the SW; it reads `sender.tab.id` and `sender.url`.

## Definition of done (every task)
1. `npm run lint && npm test && npm run test:e2e` all pass. Show the output.
2. Update SPEC "Known limitations" if behaviour deviates from SPEC.
3. List which manual checks from SPEC "Manual test checklist" you could not automate.
4. If the change touches npm scripts, layout, permissions, install steps or user-visible behaviour,
   update README.md in the same commit. Never edit the owner intro block in README.md.
5. Small, focused commits with descriptive messages.
