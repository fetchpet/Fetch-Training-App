# Fetch Training App

A working prototype of a dog training app built as a single HTML file. No build step,
no dependencies — open it in a browser and it runs.

## Running it

From this folder:

```
npx --yes serve -l 4321 .
```

Then open http://localhost:4321/rung-prototype.html

The clips have to be served over HTTP, not opened straight off disk, or the browser
blocks them.

## What's in here

| File | What it is |
|---|---|
| `rung-prototype.html` | The prototype. Everything lives in this one file. |
| `rung-prototype-cards-v1.html` | An earlier card-based version, kept for reference. |
| `clips/` | Real training footage used as worked examples in the pet library. |

Clips follow the naming convention `clips/<behaviour-id>-<rung-number>.mp4` and are
discovered automatically — dropping in `sit-6.mp4` is enough to add it.

## The idea

Training is broken into six rungs per behaviour, climbed one at a time. The app
deliberately does not do streaks, never declares a behaviour mastered, and never
claims a skill has decayed without evidence. When a dog slips back, the regression is
named and explained rather than hidden.

Films of each rung are stored on the device only, in IndexedDB. There is no feed.
Sharing happens through the phone's own share sheet, so a film only leaves the device
if the owner personally sends it.

## Privacy

This repo is private because `clips/` contains real footage of the owner and her dog.
Keep it that way.
