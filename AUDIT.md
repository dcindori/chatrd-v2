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

## Deep code review

A multi-agent code review was run separately; its verified findings are appended below
when available.
