# ChatRD Fork Audit

Audit of the local edits made on top of upstream [vortisrd/chatrd](https://github.com/vortisrd/chatrd),
diffed against the fork point (`12831fd`). Roughly 9.8k added lines: the new
`skin-chatrd.css` default skin, two stream skins, preview pages, toast/live-bar/density
systems in `js/chatrd.js`, and platform-module tweaks.

Date: 2026-08-18 · Reviewed by Claude (Fable 5)

## What's solid (no action needed)

- OBS-mode gating: input/keydown/click listeners and the autocomplete DOM are only wired
  when `chatField` (or `chatModeration` for the submit path) is enabled.
- Animation fallback: every `window.chatrdAnimate` call site falls back to the legacy
  setTimeout animation when the Motion CDN fails to load — the overlay degrades gracefully
  offline.
- Reduced-motion: `skin-chatrd.css` has two `prefers-reduced-motion` blocks covering the
  keyframe animations, including the infinite emote-float and live-chip pulse.
- Toast system: overflow capping at 3, exit-guard via `data-exiting`, cloned root id
  removed, moderation buttons stripped from clones.
- Twitch reply username is HTML-escaped (`escapeHTML`) before injection.

## Should fix

### 1. External skin URL param is a CSS injection vector
`js/chatrd.js` (skin selection): any `http(s)://` URL in the `chatrdSkin` query param is
loaded as a stylesheet. Upstream deliberately used an explicit whitelist ("✅ Explicit
whitelist" comment). Malicious CSS can exfiltrate rendered content (attribute selectors +
`url()` beacons) and fully spoof the overlay. Low practical risk while only you build the
OBS URL, but a hole if links are ever shared or the settings page generates URLs from user
input.

**Fix:** drop `http://` (mixed content fails in OBS anyway); ideally pin allowed hosts to
a domain you control.

### 2. Twitch badge matching double-fires on substrings
`js/modules/twitch/module.js` — `hasBadge` uses `b.includes(n)`, so a `lead_moderator`
badge also matches `'moderator'` and the row gets both `role-mod` and `role-lead-mod`.
Whichever CSS rule is more specific/later silently wins. Same trap for any future badge
containing `vip`/`sub` as a substring.

**Fix:** exact match (`b === n`) — all name variants for lead-mod are already passed
explicitly.

### 3. `package.json` is misleading dead weight
Declares `framer-motion@^12.38.0` but nothing imports from `node_modules` (no build step);
the runtime uses `motion@12.23.12` from jsDelivr in `chat.html`. The declared package,
its version, and the CDN pin all disagree.

**Fix:** delete `package.json` + `package-lock.json`, or align them and document why they
exist.

### 4. `streamelements-chat-test/` doesn't belong in this repo
~4k lines of a separate StreamElements widget experiment inside a ChatRD fork. Inflates
every diff and confuses readers.

**Fix:** move to its own repo, or document in CLAUDE.md why it lives here.

### 5. Live-bar counts round-trip through formatted text
Modules write formatted counts ("1.2K") into the hidden `#statistics` DOM and
`parseCount` in `js/chatrd.js` parses them back — 1,249 viewers become 1,200, and the
delta flash can show phantom ±jumps around format boundaries (999 → "1.0K").

**Fix (if exact totals ever matter):** have modules also write `data-count="1249"` and
read that instead of the display text.

## Minor nits

- `TOAST_EXIT_MS` in `js/chatrd.js` is defined but never used.
- `js/chatrd-scramble.js`: the initial `render(' '.repeat(len))` uses regular spaces,
  which collapse in HTML — phase 1 correctly uses ` `; the initial render should too,
  or the element briefly shrinks.
- Announcements are both toasted and shown inline as full clones (badge/avatar images
  re-requested unless cached). Fine in practice; just intentional duplication to be aware of.
- Cache-busting by hand-bumping `?nocache=25` across 20+ tags is error-prone — missing one
  tag ships mixed versions.

## Inherited from upstream (not regressions)

- Kick message/reply content is interpolated into `innerHTML` unescaped —
  `parseKickEmojis` only does emote regex replacement. DOMPurify is loaded in `chat.html`
  but this path doesn't use it. If hardening, wrap `messageHTML`/`replyHTML` in
  `DOMPurify.sanitize(...)`.
- The fork point is 193 commits behind upstream `main`; rebasing/merging upstream would be
  a substantial (and probably not worthwhile) effort given how far the skin has diverged.

## Deep code review findings (verified)

A second, deeper review pass (8 angles, every finding verified against the actual code).
Dominant theme: **new features are wired into the shared `addEventItem`/module pipeline
but only designed for the default skin** — the top findings are cross-skin or
fallback-path regressions.

### High severity

1. **Toasts break every non-default skin.** `addEventItem` (`js/chatrd.js:357`)
   unconditionally dual-mounts every event into `#chat-toasts`, but only
   `skin-chatrd.css` styles that container. On `stream`, `stream-bubbles`, `nutting`,
   `kimballs`, `bubbles`, or an external skin URL, the clone renders as an unstyled
   static block at the top of `.wrapper` — every sub/raid/donation appears **twice** for
   5 s and pushes `#chat` down instead of overlaying it.
   *Fix:* gate `showEventToast` on the active skin, or move the `#chat-toasts`
   positioning rules into `css/chatrd.css` (structural base).

2. **Toasts render 0px tall when the Motion CDN is unreachable.** In the no-CDN fallback
   path the inline item still has `height: 0px` inline style when `showEventToast`
   clones it synchronously (`js/chatrd.js:364`; the fallback expands the item 10 ms
   later). The clone keeps `height:0px` forever — toasts collapse and overlap. Applies
   on any OBS machine without access to cdn.jsdelivr.net.
   *Fix:* clear the dimension inline style on the clone in `showEventToast`.

3. **Live bar shows "LIVE 0 watching" whenever the YouTube module is enabled.**
   `platformActive('youtube')` (`js/chatrd.js:1188`) only checks element existence and
   ignores `display`, unlike the other three platforms. The YouTube module inserts
   `<div id="youtube" style="display:none">` at load, so the bar un-hides even with
   statistics off and nothing live.
   *Fix:* apply the same `el.style.display !== 'none'` check.

4. **Username-color removal regresses the legacy skins.** Deleting the inline
   `user.style.color` assignments (twitch:206, youtube:151, tiktok:221, kick:322) kills
   per-user colors on `nutting`/`kimballs`/`bubbles` too — none of those skins define a
   `.user` color rule; they relied entirely on the inline style. The "force white" rules
   exist only in `skin-chatrd.css`.
   *Fix:* keep setting the color but let skin-chatrd override it (it already forces
   white with higher specificity), or set it as a CSS custom property the skins can opt
   into.

### Medium severity

5. **Moderation can't remove toast clones.** Deleted/banned users' announcement content
   stays on screen up to 5 s after moderation removes the inline copy — deletion works
   by element id (stripped from the clone) and ban queries only `#chat`
   (`js/chatrd.js:361`, `twitch/module.js:871`).
   *Fix:* on delete/ban, also query `#chat-toasts` by `data-user`/message data attribute.

6. **Dead `createRandomColor` awaits on the hot path.** TikTok (`module.js:219`) and
   YouTube (`module.js:149`) still await `createRandomColor` per message even though the
   result is unused after the color removal — plus the `userColors` Map grows one entry
   per unique chatter, never pruned. *Fix:* delete the calls (or see finding 4).

### Low severity / cleanup

7. **Unused `framer-motion` npm dependency** (`package.json`) — duplicate of "Should
   fix #3" above; the CDN pin (12.23.12) and declared range (^12.38.0) can silently
   diverge.
8. **`--glow-stroke` is dead** (`css/skin-chatrd.css:84`) — never consumed via `var()`;
   the real hairline stroke knob is `--chatrd-text-stroke` (line 85). CLAUDE.md's
   "Readability system" section documents the dead variable, so future edits following
   those instructions will tweak a no-op. *Fix:* remove the variable and correct
   CLAUDE.md, or wire it up.
9. **`TOAST_EXIT_MS` unused** (`js/chatrd.js:44`) — `dismissToast` hardcodes 0.22/0.14 s;
   the named constant next to the two used ones is the obvious (non-functional) knob.
10. **Scramble first frame collapses** (`js/chatrd-scramble.js`) — initial render uses
    collapsible spaces; the shrink-wrapped event card visibly jumps narrower for a frame
    (up to the full per-node delay, e.g. 300 ms for `.actual-message`). Use ` `
    like phase 1 does.
