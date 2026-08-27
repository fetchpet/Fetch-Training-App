# Handover: black-screen video bug in the Artifact preview

## Status: UNRESOLVED

Every video in the prototype (both the main coaching-screen clips and Goldie's
film/timeline library) renders as a black box in the published Claude Artifact
preview, for the actual user viewing it in a real desktop Chrome browser
(confirmed via the browser's own right-click context menu — see below).

## How the preview is built

There's no live server for this prototype — Claude can't host anything the
user can reach. So every time a change needs to be shown, a script rebuilds a
**self-contained copy** of `rung-prototype.html`:

1. Every training clip (`clips/sit-1.mp4` … `sit-6.mp4`) and every owner/film
   clip (`clips/owner/sit-1.mp4` … `sit-4.mp4`) gets base64-encoded and
   embedded as `<script type="text/plain" data-vid="...">` /
   `data-lib="...">` blocks.
2. An IIFE appended at the very end of the page decodes each block back into
   a `Blob`, turns it into a `blob:` URL via `URL.createObjectURL()`, and
   assigns it into the app's `VID` map (training clips) or onto the matching
   `LIB` record's `.blob` (film clips) before calling `boot()`.
3. The real `rung-prototype.html` in this repo is untouched by any of this —
   it just references `clips/...` paths normally. The rebuild only happens in
   a scratch copy that gets published as a Claude Artifact.

The rebuild script isn't committed anywhere (it lives in the assistant's
scratchpad, not the repo) — whoever picks this up will need to recreate it
following the description above, or ask Claude to.

## What's been ruled out (all confirmed, not guessed)

- **Bad data**: every embedded clip was byte-verified (md5/exact match)
  against the source file after base64 round-trip. Not corrupted.
- **Storage/IndexedDB failures**: `loadLib()`/`seedLib()` were made to
  degrade gracefully instead of aborting on a storage error (then reverted
  back to the original simpler version once this was confirmed not to be the
  cause — see git history if useful, but the current code is the *original*
  simple version, not the defensive one).
- **Video codec**: clips were re-encoded from H.264/MP4 to VP9/WebM as an
  experiment. Made no difference — still black.
- **The general Artifact platform**: published a bare-bones test page with
  nothing but `<video>` tags (no app code at all) — direct blob assignment,
  innerHTML-based assignment (exactly how the real app wires up video src),
  and blob-URL reuse across multiple elements. **All of these worked
  perfectly** in the user's real browser — real frames rendered, "OK"
  status, no errors. Test artifacts (still live, may be useful):
  - `https://claude.ai/code/artifact/7e7fc4a0-16ad-4ce2-bf57-1c99c6806069`
  - `https://claude.ai/code/artifact/1576da81-5662-4073-a8bf-1bb98855bbeb`

So: blob-based video works fine in the user's browser, in isolation, using
the exact same mechanism the real app uses. But the same technique fails
inside the full app. **The bug is specific to something in the full app's
runtime**, not the embedding approach, not the codec, and not the platform.

## Diagnostic evidence collected

- Right-click on a black video box in the real app shows a genuine `<video>`
  context menu (Loop, Copy Video Address, Cast… all present/enabled) but
  everything requiring an actually-decoded frame is greyed out (Save Video
  Frame As, Save Video As, Open Video in New Tab, Picture-in-Picture). This
  means: **the element exists and has a src, but never successfully decodes
  a frame.**
- Browser console shows only expected noise: dozens of 404s from
  `loadClips()`'s opportunistic probing of `clips/{behaviour}-{n}.mp4` for
  every behaviour (door, recall, etc.), which always 404 in this
  no-real-server preview context regardless of whether video works. One
  "Unrecognized Content-Security-Policy directive 'webrtc'" warning, which
  looks like Anthropic platform boilerplate, not an app-caused issue.
  **Never got confirmation of what's below the fold in the console** — the
  user was asked twice to scroll to the very bottom for anything that isn't
  a 404 (e.g. an "Uncaught TypeError") and never got back a clear answer.
- On Claude's own machine, headless Chromium (Playwright) has **no H.264
  decoder at all** (confirmed independently — even a plain, non-blob,
  non-app `<video src="file.mp4">` fails there) but **does** decode VP9/WebM
  fine. This made Claude's own local testing partially blind for a while —
  worth knowing if you go down the same path. It does NOT explain the user's
  issue, since the user is on real desktop Chrome (context menu showed a
  "Cast…" option, which is Chrome-only) where both H.264 and VP9 work — as
  proven by the isolated test pages above.

## Best next steps for whoever picks this up

1. **Get the browser console scrolled to the very bottom**, past the 404
   noise — there may be a real JS error being missed.
2. **Get a full, non-cropped screenshot of the running app** (not just the
   video area) — never actually confirmed whether it's *only* the video
   that's black or something more is off (layout, other elements).
3. Consider testing with a **smaller, less complex slice of the real app**
   (e.g. just the film/timeline screen, stripped of the CSS animations and
   the multi-behaviour deck) to bisect what specifically differs from the
   isolated test pages that worked.
4. Consider whether `buildAll()` being called multiple times in quick
   succession during boot (once in `boot()`, again inside `loadLib()`, again
   at the end of `loadClips()`) could be tearing down/rebuilding video
   elements in a way that never lets one settle — not confirmed, but never
   properly tested in isolation either.
5. If reproducing locally: be aware Playwright's bundled Chromium here has
   no H.264 decoder, so a "no errors, looks fine" local result doesn't
   guarantee it plays in a real browser. Test VP9/WebM locally, or find a
   Chromium build with proprietary codecs if H.264 support needs verifying.

## What's currently in this commit

- Removed the WhatsApp/Instagram/TikTok share buttons and the "mock
  pretend" share popup — replaced with a single "Save Goldie's Video"
  option, per explicit request.
- Reworded the journey-sharing line under the film section.
- "Getting it" video: 5 weeks ago → now the oldest at 24 days ago (still
  reads as "3 weeks ago" via the app's existing date-label logic).
- Goldie's film library clips (`clips/owner/sit-1.mp4` … `sit-4.mp4`)
  replaced with fresh footage the user re-uploaded, ordered oldest → newest:
  Getting it → Adding the word → Losing the food → Fluent at home (the
  "now" clip).
- `SEEDKEY` bumped to `fetch-seeded-v5` so any browser with old cached
  library data re-seeds with the new clips.
