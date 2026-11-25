# Running this project with Claude Code

## 0. Preparation (you do this, once)

1. Node.js 20+ and Chrome stable installed. Claude Code logged in.
2. Create the repo and drop in the kit files:
   ```
   mkdir image-hider && cd image-hider && git init
   mkdir -p docs .claude
   # copy CLAUDE.md to repo root, SPEC.md to docs/SPEC.md, settings.json to .claude/settings.json
   git add -A && git commit -m "chore: project brief and spec"
   ```
3. `.claude/settings.json` (lets Claude run the test loop without asking each time, blocks risky commands):
   ```json
   {
     "permissions": {
       "allow": [
         "Bash(npm run lint)", "Bash(npm test)", "Bash(npm run test:e2e)",
         "Bash(npm run package)", "Bash(npm install -D *)", "Bash(npx playwright install chromium)",
         "Bash(git add *)", "Bash(git commit *)", "Bash(git status)", "Bash(git diff *)",
         "WebFetch(domain:developer.chrome.com)",
         "Bash(npm ci)", "Bash(git clone *)", "Bash(wc -l README.md)"
       ],
       "deny": ["Bash(git push *)", "Bash(rm -rf *)", "Bash(npm publish *)"]
     }
   }
   ```
4. Read `docs/SPEC.md` sections 4, 5, 11, 13 and change anything you disagree with before starting. The spec is the contract.
5. Create `README.md` with only your own intro (2 to 3 sentences: what it does, for whom, why). Claude fills the rest later and must keep this block verbatim:
   ```markdown
   # Image Hider

   <!-- owner-intro:start -->
   Your 2-3 sentences here.
   <!-- owner-intro:end -->
   ```
   Commit it together with the other kit files.

Workflow per phase: start in plan mode (Shift+Tab until "plan mode on"), review the plan, approve, let it implement, check the evidence it shows, then `/clear` before the next phase so context stays small.

---

## 1. Kickoff prompt (plan mode, no code)

```
Read CLAUDE.md and docs/SPEC.md fully. We are building the Chrome MV3 extension they describe.

Do not write code yet. Do this:
1. Verify against current Chrome docs (developer.chrome.com) every API the SPEC depends on:
   action.onClicked + permissions.request in that gesture, optional_host_permissions,
   scripting.registerContentScripts/updateContentScripts with persistAcrossSessions,
   storage.session, tabs.onReplaced, setBadgeText per tab. Note the minimum Chrome version for each.
   Cite the doc URL for each finding.
2. List anything in the SPEC that is contradictory, under-specified, or not implementable as written,
   with a proposed resolution for each. Do not resolve them silently.
3. Write docs/PLAN.md: phases matching section 2 of PROMPT.md, files touched per phase,
   which acceptance criteria (SPEC section 9) each phase makes pass, and the test strategy
   (what is unit, what is Playwright E2E, what stays manual and why).
4. Stop and wait for my review.
```

Answer its questions, edit `docs/PLAN.md` directly if needed (Ctrl+G opens the plan in your editor), then reply: `Address all my notes in PLAN.md. Do not implement yet.` Repeat until no open questions. Commit PLAN.md.

---

## 2. Phase prompts (one session each, `/clear` between)

### Phase 1: scaffold and test harness
```
Implement Phase 1 of docs/PLAN.md: package.json (devDependencies only), ESLint flat config,
manifest.json per SPEC section 6, empty background.js and content script, placeholder icons via
scripts/make-icons.mjs (no deps), fixture server script, tests/fixtures/basic.html covering every
row of SPEC section 4 plus a gradient div and a Google Maps iframe, Playwright config that launches
Chromium with the unpacked extension, and one smoke E2E test that asserts the service worker is running.
Run lint, unit and e2e. Show the output. Commit.
```

### Phase 2: flash spike (decision, not product code)
```
Do the spike in SPEC section 5. Build approach A on a throwaway branch, write an E2E that reloads a
hidden fixture page with large images and captures screenshots as early as possible. Report whether
any image pixels are visible before hiding, with evidence. If A flashes, describe B and C trade-offs
measured the same way. Do NOT add webNavigation or any permission. Recommend one approach and stop.
```

### Phase 3: state, permissions, badge (service worker)
```
Implement Phase 3: state model and lifecycle (SPEC 2, 3.1, 3.2, 3.4), permission request and deny
fallback, dynamic content script registration and reconciliation, badge/title (SPEC 7).
Put pure logic (state keys, scriptable URL check, origin pattern) in extension/lib with unit tests
written first. E2E: acceptance criteria 3, 4, 5, 8. The permission prompt cannot be clicked in
Playwright: propose how tests get host access (for example a test-only launch flag or pre-granted
host_permissions in a test copy of the manifest) and make sure nothing test-only ships in the zip.
Show test output. Commit.
```

### Phase 4: hiding engine (content script)
```
Implement Phase 4: content/hider.js and hider.css per SPEC section 4 and the flash approach chosen
in Phase 2. Write E2E for acceptance criteria 1, 2, 6, 7 before implementing. Criterion 1 must compare
getBoundingClientRect of every element before and after. Then run the performance check in SPEC 8
on a generated 5,000-element fixture and report numbers. Commit.
```

### Phase 5: packaging and store assets
```
Implement Phase 5: scripts/package.mjs (zip of extension/ only, fails if test-only files or
manifest version not bumped), PRIVACY.md, docs/STORE_LISTING.md with name, summary <=132 chars,
description, and one justification sentence per permission for the CWS privacy practices tab.
Generate a 1280x800 screenshot from the basic fixture (before/after side by side) and a 440x280
promo tile using Playwright. Verify acceptance criterion 9. Commit.
```

### Phase 6: README
```
Write README.md per docs/SPEC.md section 13. Keep the owner-intro block verbatim.
Before writing, read manifest.json, package.json and the repo tree; every fact in the README must
match them. Reuse the 1280x800 screenshot from Phase 5 as docs/img/before-after.png.
If the license is not decided, write "TODO: license" and ask me at the end.
Then verify: git clone the repo into a temp folder outside the project, run npm ci and every command
the README lists, report the result of each. Check line count is under 150.
Verify acceptance criterion 10. Commit.
```

### Phase 7: independent review
```
Use a subagent that has not seen this implementation to review the repo against docs/SPEC.md and
CLAUDE.md "Hard rules". It should report: SPEC requirements not met, restore leaks (anything left
on the page after OFF), MV3 service worker state bugs, permission over-reach, CWS policy risks,
and README statements that do not match the code or duplicate SPEC content.
Then fix confirmed issues only, and list the ones you disagree with and why.
```

---

## 3. After Claude Code finishes (you do this)

- Run SPEC section 10 manual checklist yourself in Chrome stable.
- Read README.md as a stranger: follow the Install section on another machine or a fresh Chrome profile.
- After publishing, replace `TODO: store link` in README.md with the real store URL.
- Replace placeholder icons and promo art.
- Publish `PRIVACY.md` at a public URL (GitHub Pages or a gist).
- Chrome Web Store Developer Dashboard: upload zip, fill listing from `docs/STORE_LISTING.md`, privacy practices, submit.
