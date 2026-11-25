# SPEC: Image Hider

Status: v1 draft. Owner: Tu. Changes to this file need owner approval.

## 1. Goal
One click on the toolbar icon toggles "hide images" for the current tab and current hostname.
Hidden media is painted solid white in place. Layout (size, position, flow) does not change.

Non-goals (v1): popup UI, options page, per-element reveal, sync across devices, Firefox/Safari,
blocking network downloads of images (images still load, they are only covered).

## 2. Terms
- **Hostname**: `new URL(tab.url).hostname`, exact match. `a.example.com` and `example.com` are different.
- **State key**: `${tabId}|${hostname}`. Value `true` = hidden. Absent = shown.
- **Scriptable URL**: `http:` or `https:`, excluding the Chrome Web Store domains. Everything else is non-scriptable.

## 3. User flows
### 3.1 Toggle ON (icon click, state currently shown)
1. URL not scriptable: no-op, badge text `-` for 2 s, title "Image Hider is not available on this page".
2. Check `chrome.permissions.contains({origins: ["*://<hostname>/*"]})`.
3. If not granted: call `chrome.permissions.request` for that origin (first await in the handler).
   - Granted: continue at step 4.
   - Denied: apply hiding to the current page only via `activeTab` + `scripting.executeScript`.
     Do not persist state. Badge `ON`, title "Hidden for this page only (no site access, will not survive reload)".
4. Persist state key. Ensure the dynamic content script registration includes this origin.
5. Inject content script into the current tab now (it was not loaded before the grant) and apply hiding.
6. Badge `ON` for this tab, title "Images hidden on <hostname>. Click to show".

### 3.2 Toggle OFF
Remove state key, tell the content script to restore, clear badge, title "Click to hide images".
Do not revoke the host permission.

### 3.3 Reload / navigation in the same tab
- Reload or same-hostname navigation: state persists, hiding re-applied automatically.
- Navigate to a different hostname: that hostname's own key decides (default shown).
  Returning to a previously hidden hostname in the same tab hides again.
- SPA route changes (pushState): no state change, new DOM nodes handled by the observer.
- Back/forward cache restore (`pageshow` with `persisted`): re-query state and re-apply/restore.

### 3.4 Tab lifecycle
- Tab closed: delete all keys for that tabId (`tabs.onRemoved`).
- Tab replaced (prerender/discard, `tabs.onReplaced`): move keys to the new tabId.
- Browser restart: `storage.session` is cleared, everything starts shown. This is intended.

## 4. What gets hidden and how
Content script sets `data-image-hider="on"` on `<html>`. One injected stylesheet does the visual work.

| Target | Selector / detection | Technique |
|---|---|---|
| `<img>`, `<picture>`, `<input type=image>`, SVG `<image>` | CSS selector | `filter: brightness(0) invert(1) !important; background-color:#fff !important` (transparent areas also become white) |
| `<video>` incl. `poster` | CSS selector | Same filter. Plus: pause playing videos, mark `data-ih-paused`; on show resume only marked ones (`play()` rejection ignored). While hidden, any `play` event (capture phase on document) on a video pauses it and marks it. |
| CSS `background-image` with `url()` or `image-set()` containing url, on elements and `::before`/`::after` | JS scan via `getComputedStyle`; mark `data-ih-bg`, `data-ih-bg-before`, `data-ih-bg-after` | `background-image: none !important; background-color:#fff !important` for the marked element/pseudo. Pure gradients are not touched. |
| `<iframe>` from media hosts | `src` hostname in allowlist in `extension/lib/media-hosts.js` (YouTube incl. nocookie, Vimeo player, Dailymotion, Facebook video plugin, TikTok embed, Twitch player/clips, Spotify embed, Instagram embed) | Same filter on the iframe element. Non-media iframes (maps, payments, forms, chat) untouched. |

Dynamic content: one `MutationObserver` on `childList`, `subtree`, and `attributes` (`class`, `style`, `src`).
Background scans are batched with `requestIdleCallback` (fallback `setTimeout`) and only re-scan changed subtrees.
Open shadow roots: discovered during scan, same stylesheet attached via `adoptedStyleSheets`.

Restore (OFF) must remove: the html attribute, all `data-ih-*` marks, adopted stylesheets, listeners, observer,
and resume extension-paused videos. Page must be pixel-identical to a never-hidden load (except video playback position).

## 5. Avoiding image flash on reload
Requirement: on reload of a hidden page, images should not be visibly painted before being covered.
Implementation is decided by a spike (see PROMPT phase 2):
- A. Dynamic content script at `document_start` asks the SW for state, then applies. Zero extra permissions. May flash.
- B. SW applies CSS via `scripting.insertCSS` on `webNavigation.onCommitted`. Needs `webNavigation` permission (extra warning). Requires owner approval.
- C. Content script at `document_start` applies hide CSS immediately, removes it if SW says shown. Inverts the flash onto non-hidden pages of granted hosts.
Default: A. Measure with E2E (screenshot at first paint/`DOMContentLoaded`) on a fixture with large images and report before choosing.

## 6. Permissions
- `permissions`: `storage`, `scripting`, `activeTab`
- `optional_host_permissions`: `http://*/*`, `https://*/*` (requested per hostname at click time)
- No `content_scripts` in manifest. Use `chrome.scripting.registerContentScripts` / `updateContentScripts`, one registration id `image-hider`, matches = granted origins, `runAt: document_start`, `allFrames: false`, `persistAcrossSessions: true`.
- Reconcile registration with `chrome.permissions.getAll()` on `onInstalled`, `onStartup`, `permissions.onAdded`, `permissions.onRemoved`.
- Set `minimum_chrome_version` to the lowest version that supports every API used (verify in Chrome docs, cite it in the commit).

## 7. Badge and icon
- Badge per tab (`setBadgeText({tabId})`): `ON` when hidden, empty when shown. Neutral badge color.
- Update badge on `tabs.onUpdated` (url/status change) from stored state.
- Icons 16, 32, 48, 128 PNG. Placeholder icons generated by `scripts/make-icons.mjs` (no deps); owner replaces before publishing.

## 8. Performance
- Applying on a 5,000-element fixture: main thread blocking under 50 ms per chunk, full apply under 500 ms.
- No continuous polling. Observer callbacks debounced.
- Content script must not throw on any page; wrap entry points and log with a `[image-hider]` prefix only in dev.

## 9. Acceptance criteria (automated where possible)
1. Fixture `basic.html` (img, picture/srcset, transparent PNG, video with poster, background-image div, `::before` background, gradient div, YouTube iframe, Google Maps iframe): after ON, every listed media target is white; gradient and Maps iframe unchanged; `getBoundingClientRect()` of every element identical before/after.
2. OFF restores: no `data-ih-*` attributes left, no extension styles, screenshot equal to pre-ON screenshot.
3. Reload keeps hidden; navigating to another hostname in same tab is shown; navigating back is hidden.
4. Two tabs on the same hostname are independent.
5. Closing a tab removes its keys from `storage.session`.
6. Dynamic: images added by JS 1 s after load, lazy-loaded images, and class-toggled backgrounds get hidden.
7. Video playing before ON is paused; resumes after OFF. Video that was paused before ON stays paused after OFF.
8. Non-scriptable page click does not throw and shows the `-` badge.
9. Packaging zip contains only `extension/` contents, manifest valid, no source maps or test files.

## 10. Manual test checklist (cannot be fully automated)
- Permission prompt appears on first ON for a new hostname; deny path works and shows the "this page only" title.
- Real sites: news site, YouTube watch page, Instagram, Google Images, a site with dark theme.
- Service worker killed (chrome://serviceworker-internals > Stop) then click icon: still works.
- Extension reloaded in chrome://extensions: previously granted hosts still auto-hide after reload.

## 11. Known limitations (v1, accepted)
- Cross-origin iframe players keep playing audio (cannot pause without the player's API).
- Text placed over a removed background image may become unreadable (e.g. white text on hero image).
- Not covered: `<canvas>`, inline SVG drawings, icon fonts, emoji, CSS `mask`/`border-image`/`list-style-image`, closed shadow roots, images inside non-media iframes, favicons.
- First click on each new hostname shows a permission prompt (consequence of optional host permissions).
- Images are still downloaded; this is visual hiding, not bandwidth saving.

## 12. Chrome Web Store release
- Developer account registered, one-time fee paid.
- Zip from `npm run package`. Version bumped in `manifest.json`.
- Listing: name, summary (<= 132 chars), description, category, language, 128x128 store icon, at least one 1280x800 screenshot, small promo tile 440x280 (check if the dashboard currently requires it).
- Privacy practices tab: single purpose statement, justification for each permission (`storage`, `scripting`, `activeTab`, optional host access), certify no user data collected.
- Privacy policy: `PRIVACY.md` published at a public URL (states: no data collected, no network requests, state kept locally in session storage).
