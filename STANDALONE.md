# Standalone overlay model: pristine upstream core + your changes on top

Goal: keep `vortisrd/chatrd` in this repo **byte-for-byte identical** to upstream, put all
of your visual tweaks and features in files upstream will never create, and make
`git pull` from upstream conflict-free forever.

Short answer: yes, this works — with **one deliberate exception**: a single `<script>`
line added to `chat.html` (and optionally `preview-chatrd.html`). Everything else lives
in a new `custom/` directory. One added line at the end of `<body>` essentially never
conflicts with upstream edits, and if it ever does, the resolution is "keep both lines".

---

## 1. The architecture

```
chatrd/                     ← everything here stays 1:1 with upstream
├── chat.html               ← +1 line: <script src="custom/loader.js"></script>
├── css/ …                  ← untouched
├── js/ …                   ← untouched
└── custom/                 ← 100% yours; upstream has no "custom/" dir
    ├── loader.js           ← bootstraps everything below
    ├── custom.css          ← all your visual overrides
    └── features/*.js       ← your added features, one file each
```

Why this works in *this* codebase specifically:

- **No build step.** Files are loaded as-is by the browser; adding files changes nothing
  for upstream code.
- **CSS cascade = override mechanism.** `loader.js` appends your `<link>` **after** every
  module `module.css`, so at equal specificity your rules always win. Most of the skin is
  driven by CSS variables in `:root` (`--chatrd-user-col`, `--p-twitch`, `--glow-text`, …)
  — redefining a variable in `custom.css` restyles the whole skin without touching
  `skin-chatrd.css`.
- **All JS is global and late-bound.** `addMessageItem`, `addEventItem`, and every
  `<Platform>MessageHandlers` object are plain globals. Messages only arrive later over
  the Streamer.bot WebSocket, so a script that runs after the core scripts (which
  `loader.js` does, being the last `<script>` in `<body>`) can wrap/replace any of them
  before the first event is ever processed.
- **Templates are mutable at runtime.** `<template id="chat-message">` /
  `<template id="event-message">` are cloned per message. `loader.js` can edit
  `template.content` once at startup and every future message uses your modified markup —
  no edit to `chat.html`'s template section needed.

## 2. Git setup

```bash
# one-time
git remote add upstream https://github.com/vortisrd/chatrd
git fetch upstream
```

Branch discipline:

| Branch | Contents | Rule |
|---|---|---|
| `upstream-main` (optional) | exact mirror of upstream | fast-forward only, never commit here |
| `main` | upstream + overlay commits | only ever add/edit files under `custom/`, plus the loader line |

Routine update (the whole point):

```bash
git fetch upstream
git merge upstream/main
```

Because your commits only add new files (`custom/**`, `STANDALONE.md`) and one line in
`chat.html`, git's merge machinery has nothing to conflict on. Upstream can rewrite
`skin-chatrd.css` top to bottom and the merge is still automatic.

### The one exception, and why it's safe

`chat.html` gets exactly one added line, as the last script in `<body>`:

```html
<script src="custom/loader.js?nocache=1"></script>
```

A conflict only occurs if upstream edits the *adjacent* lines in a way git can't
reconcile — rare, and trivially resolved (keep upstream's change + keep your line).
Do the same in `preview-chatrd.html` so the preview shows your overlay too.

Never edit any other upstream file. If you catch yourself needing to, see §5.

## 3. What `custom/loader.js` does

```js
(() => {
  // 1. CSS overlay — appended last, so it wins the cascade
  const css = document.createElement("link");
  css.rel = "stylesheet";
  css.href = "custom/custom.css?nocache=1";
  document.head.appendChild(css);

  // 2. Template surgery (example: add a wrapper span inside .actual-message)
  //    Runs before any WebSocket message renders, so it's race-free.
  const tpl = document.getElementById("chat-message");
  // tpl.content.querySelector(".info").append(...);

  // 3. Wrap core functions (monkey-patch instead of editing js/chatrd.js)
  const _addMessageItem = window.addMessageItem;
  window.addMessageItem = function (...args) {
    // pre-hook: filter, rewrite, tag…
    const result = _addMessageItem.apply(this, args);
    // post-hook: decorate the row that was just inserted
    return result;
  };

  // 4. Extend platform handlers (add events without touching module.js)
  //    e.g. TwitchMessageHandlers["Twitch.SomeEvent"] = (data) => { … };

  // 5. Your own URL params
  const params = new URLSearchParams(location.search);
  const myFeature = params.get("myFeature") === "true";
  if (myFeature) { /* load custom/features/my-feature.js */ }
})();
```

Keep each feature in its own file under `custom/features/` and have `loader.js` inject
them; that keeps every commit isolated and portable.

## 4. What `custom/custom.css` does

Three tiers of override, all conflict-free:

1. **Variables** — cheapest and most robust:
   ```css
   :root {
     --chatrd-user-col: 180px;
     --p-twitch: #a970ff;
   }
   ```
2. **Rule overrides** — same selector upstream uses; you win because you load last:
   ```css
   .info .user { color: #fff !important; }   /* !important only if upstream inlines styles */
   ```
3. **New rules** — for markup your loader adds.

Note on the skin system: `chatrdSkin` also accepts a full `http(s)://` URL, so if you
ever host `custom.css` somewhere, you can theme a **stock, zero-edit** checkout purely
from the OBS URL — but that path is CSS-only and needs http(s), which is why the loader
line is the more general mechanism.

## 5. Limits — what an overlay *can't* cleanly do

- **Deleting upstream behavior** that has no hook: you have to neutralize it (wrap the
  function to no-op, `display:none` the element) rather than remove the code.
- **Deep rewrites of upstream logic**: if a feature requires rewriting the middle of
  `js/sb.js`, an overlay turns into a fragile monkey-patch. For those cases, either
  (a) reproduce the whole function in `custom/` and replace the global, accepting you
  must re-check it when upstream changes that file, or (b) contribute the change
  upstream as a PR so it stops being your problem.
- **Renames upstream makes** (a global gets renamed, a template id changes) will silently
  break your patches. Mitigation: `loader.js` should assert its expectations and
  `console.warn("[custom] hook missing: addMessageItem")` so breakage is loud after a merge.

## 6. One-time migration from the current fork state

You currently have edits baked into upstream files across many commits. Migration:

1. `git fetch upstream` and create the clean base:
   `git checkout -b overlay upstream/main`
2. Diff your old `main` against upstream to see everything you changed:
   `git diff upstream/main main -- . > /tmp/my-changes.diff`
3. Re-express each change as overlay material:
   - CSS edits → variables/rules in `custom/custom.css`
   - JS edits → wrappers or handler additions in `custom/features/*.js`
   - New files you authored (e.g. `AUDIT.md`) → keep as-is, they're additive already
4. Add the loader line to `chat.html` / `preview-chatrd.html`, commit.
5. Verify in `preview-chatrd.html` (and the live overlay) that the look matches.
6. Point `main` at the new history: `git branch -m main main-old && git branch -m overlay main`
   (keep `main-old` around until you're confident nothing was missed), then
   `git push -f origin main`.

## 7. Day-to-day rules

- New work goes in `custom/` — always ask "can this be a variable override, a wrapper,
  or a template mutation?" before touching an upstream file.
- Update with `git fetch upstream && git merge upstream/main`; after every merge, open
  `preview-chatrd.html` and check the console for `[custom]` warnings.
- If upstream ships a feature that duplicates one of yours, delete your overlay file —
  that's the whole payoff of never having forked their code in place.
