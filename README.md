## espresso.cam
espresso.cam is an easy, in-browser way to set up simple live pet cams. It works on any device with a webcam and a modern browser—no apps to install, no complicated setup, and no ongoing subscriptions. Start a camera on one device and watch from another; everything runs right in the browser.

It’s powered by Agora’s real‑time video, so it’s smooth and low-latency. Agora includes 10,000 free viewer minutes each month, which means you can realistically watch your furry friends for roughly 22% of the month with 1 camera, or about 11% of the month with 2 cameras—at zero cost—assuming a single viewer.

I'd say check out the demo but this is meant to be free and you would drive my view minute costs up. So feel free to see the deployed demo, that has an expired token at [espresso.cam](https://espresso.cam)

Static, browser‑only app hosted on GitHub Pages. Media and signaling are provided by Agora (RTC + RTM). No custom backend.



### Tech stack
- **UI**: Plain HTML + Tailwind via CDN.
- **Media transport**: Agora RTC Web SDK 4.x (`AgoraRTC_N-4.20.2.js`) in live mode with H.264.
- **Signaling/control**: Agora RTM JS SDK (`AgoraRTM-1.5.1.js`) for lightweight peer messaging.
- **Animations**: Motion One (small UMD) for subtle loading/offline cat motion.
- **PWA**: `manifest.webmanifest` + `sw.js` for installability and offline caching.
- **Hosting**: Static files on GitHub Pages (optional custom domain via `CNAME`).

### Pages and roles
- `index.html` (Viewer): joins an Agora channel as `audience`, auto‑subscribes to remote publishers, and implements an active/preview video model with dual‑stream selection.
- `streamer.html` (Broadcaster): joins as `host`, captures mic/camera, and publishes both tracks to the same channel.

### Media pipeline (what’s happening)
- Viewer creates an Agora RTC client in `live` mode, `codec: h264`, sets role `audience`, then `join(appId, channel, token: null)`.
- Broadcaster creates an RTC client, sets role `host`, `join(appId, channel, token?)`, and publishes tracks from `createMicrophoneAndCameraTracks()`.
- Viewer listens for `user-published` events, subscribes, and renders into two surfaces:
  - **Main**: the currently active publisher; requests "high" stream.
  - **Preview**: next publisher (if any); requests "low" stream.
- Active/preview can be swapped at runtime. On `user-unpublished`/`user-left`, the preview is promoted or an offline overlay is shown.
- This is WebRTC‑based over Agora’s SD‑RTN (low‑latency relay). It behaves P2P‑like for clients, but media typically transits Agora’s edge for reliability and NAT traversal.

### Signaling/control path (Agora RTM)
- Both viewer and broadcaster log into RTM with the same `appId` and join the same RTM channel name as RTC.
- Viewer keeps track of the active publisher’s `uid` and sends peer messages directly:
  - `{ type: "playSound", key }` → instructs the active broadcaster to play a local audio asset.
  - `{ type: "memo", format, data }` → short voice memo (base64) to be decoded and played once by the broadcaster.
- Broadcaster listens to `MessageFromPeer` and executes actions (play an `Audio` element, decode memo blob and play once). No persistence.

### PWA and caching behavior
- `sw.js` pre‑caches core shell (`/`, `/index.html`, `/streamer.html`, `/manifest.webmanifest`).
- Cache‑first on GET: serve cached response if available; otherwise fetch, then cache the fresh copy. Simplifies offline start and speeds repeat visits.
- Updates require a refresh to activate the new service worker; hard refresh forces it.

### Configuration defaults (in code)
- Default `appId` and `channel` are set in both pages via URL param fallback:
  - Viewer: `defaultAppId = "c36cae78aea74b38ae8706fcd9557387"`, `defaultChannel = "cat-cam"`.
  - Broadcaster: same defaults; token optional.
- Viewer connects tokenless as audience; broadcaster may supply a token if your Agora project requires it.

### Runtime order of operations
1. Browser loads static shell and registers the service worker.
2. Viewer immediately joins RTC as `audience` and RTM for control.
3. When a broadcaster joins and publishes, viewer subscribes, renders, and manages active/preview tracks with dual‑stream hints.
4. Control messages (RTM) flow from viewer → active broadcaster to trigger local actions (sounds, memo playback).

### Deployment and hosting model
- Static site served from the repository root via GitHub Pages. Optional `CNAME` binds a custom domain. Enforce HTTPS for device permissions.
- No server, database, or build pipeline. All logic is client‑side; only Agora services are external.

### Cost profile and effectiveness
- **Hosting**: $0 on GitHub Pages for public repos.
- **Primary cost**: Agora RTC/RTM usage (publish/subscribe minutes and egress). Scales with broadcaster uptime and viewer concurrency.
- **Efficiency**: Dual‑stream selection (high for main, low for preview) reduces viewer bandwidth. No backend operating cost.

### File map
- `index.html`: viewer logic (RTC audience, RTM control sender, UI/preview system, PWA hooks).
- `streamer.html`: broadcaster logic (RTC host, RTM control receiver, local audio playback, memo decoding).
- `sw.js`: service worker (cache‑first shell; versioned cache key).
- `manifest.webmanifest`: PWA metadata.
- `assets/`: local audio effects for control messages.

### Operational notes
- Requires HTTPS (or `http://localhost`) for camera/microphone and service worker.
- Viewers do not need tokens when the project allows tokenless subscribe; broadcasters may require a token depending on project security.
- 404 on subpages: ensure Pages serves from repo root.





