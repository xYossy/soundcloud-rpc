# Fork notes — xYossy/soundcloud-rpc

Fork of [richardhbtz/soundcloud-rpc](https://github.com/richardhbtz/soundcloud-rpc), Windows-first.

## What differs from upstream

- **Header spoof only on navigations** (`src/main.ts`): upstream rewrote `Sec-Fetch-Mode/Dest/Site/User`, `Accept` and
  the UA on **every** request. Every API call (likes, follows, uploads) therefore looked like a top-level document
  navigation to SoundCloud's bot protection (DataDome), which answers with `403` + a captcha. Now only
  `mainFrame`/`subFrame` navigations get the UA/Accept-Language spoof; XHR/fetch keep Chromium's real `Sec-Fetch-*`.
- **UA claims the real engine version** (`process.versions.chrome`) instead of a hardcoded `Chrome/120` next to a
  Chromium 146 engine.
- **Reliable like state** (`src/services/audioMonitorService.ts`): the like button's `class` is watched with a
  `MutationObserver` instead of a single 50 ms read. Works while paused, catches slow API responses and reverts. Click
  fallbacks at 300 ms and 1000 ms.
- **Selector fallbacks**: `.playbackSoundBadge__like` → `.playbackSoundBadge .sc-button-like` → `.sc-button-like` in
  both the monitor script and the Windows thumbar.
- **Thumbar errors are logged** instead of swallowed; the click reports whether the button was found.
- **Thumbar play/pause tooltip** updates `buttons[2]` (was `buttons[1]`, i.e. the previous button).
- `workflow_dispatch` trigger so releases can be started from a tag in a fork.

## Why likes fail (measured 2026-11-03)

Clicking the player heart sends `POST https://api-v2.soundcloud.com/users/<uid>/track_likes/<trackid>`. With the old
build that request carried `Sec-Fetch-Mode: navigate`, `Sec-Fetch-Dest: document`, `Sec-Fetch-Site: none`,
`Accept: text/html` and a `Chrome/120` UA. DataDome (SoundCloud's bot protection) answered:

```
403 {"url":"https://geo.captcha-delivery.com/captcha/?..."}
```

The frontend then logs `Could not toggle value` and the heart never flips. Same 403 appears in a normal Chrome once the
IP/session is flagged, so part of this is network reputation, not the app.

Before/after, captured from the running app with a local header sink:

| header         | v0.2.1           | v0.2.2           |
| -------------- | ---------------- | ---------------- |
| Sec-Fetch-Mode | navigate         | cors             |
| Sec-Fetch-Dest | document         | empty            |
| Sec-Fetch-Site | none             | cross-site       |
| Accept         | text/html,...    | _/_              |
| User-Agent     | Chrome/120.0.0.0 | Chrome/146.0.0.0 |

When DataDome challenges, the only way through is solving its captcha in the same profile that makes the request (the
app's own session, not an external browser).

## Why likes still reverted after 0.2.2 (measured 2026-11-04)

0.2.2 stopped the navigation headers on XHR, but still overrode the UA (`app.userAgentFallback`, `setUserAgent`) and
spoofed `sec-ch-ua` on navigations. Symptom in the app: liking from the player bar shows the track in /you/likes for
about half a second, then it disappears. Unliking works. Liking on the track page goes through but only shows after a
reload.

That asymmetry (writes rejected, deletes fine, playback fine) is SoundCloud's device verification, not DataDome.
Upstream issue #125 and open PR #132 name the cause: every header interceptor and UA override makes the wrapper's
request metadata contradict the engine, and authenticated writes get rejected.

0.2.3 removes all of it:

- both `session.webRequest.onBeforeSendHeaders` interceptors
- `app.userAgentFallback` and every `setUserAgent` call
- manual `sec-ch-ua*`, `Accept`, `Sec-Fetch-*`

Chromium now builds its own identity. Verified on the Linux build: the page still renders with the native Electron UA
(`... soundcloud-rpc/0.2.3 Chrome/146.0.7680.166 Electron/41.1.1 Safari/537.36`) and the player bar is present.

A regression test (`src/browserNetworkingPolicy.test.ts`) fails if any interceptor, UA override or manual client hint
comes back.

Diagnostics: `logLikeRequests()` writes every `track_likes` request with method, status and error into
`<userData>/like-debug.log`, so the next failure has a status code instead of a guess.

## Why the like is 403 and the unlike is 200 (measured 2026-11-04)

Reproduced in a normal logged-in Chrome on the same machine, two seconds apart:

```
DELETE .../track_likes/1865238375  -> 200  "OK"
PUT    .../track_likes/1865238375  -> 403  {"url":"https://geo.captcha-delivery.com/captcha/?..."}
```

Same session, same IP, same page. So DataDome challenges the **like action** and lets the **unlike** through. Nothing in
the app causes it, and it is not a like limit.

0.2.4 handles it: the injected monitor script wraps XHR and `fetch`, and when a `track_likes` request answers 403 with a
captcha URL, it hands that URL to the main process over the `soundcloud:captcha` bridge. The main process opens a
480x640 window that shares the app's session, so solving the challenge sets the `datadome` cookie for the app itself.
The next like goes through.

Diagnostics stay: `<userData>/like-debug.log` keeps the status per request.

## Architecture in one screen

```
BrowserView (soundcloud.com)
  audioMonitorScript (injected)  ──sendTrackUpdate──▶  preload.ts (contextBridge)
                                                          │ IPC soundcloud:track-update
                                                          ▼
                                                       main.ts (validation + fan-out)
                                                         ├ presenceService (Discord RPC)
                                                         ├ lastFmService (scrobbling)
                                                         ├ webhookService
                                                         ├ pluginService
                                                         └ thumbarService (Windows taskbar)
```

The renderer has no Node access; every state flows through that single IPC channel and is validated in
`src/validation.ts` (`isTrustedSoundCloudSender` restricts it to `*.soundcloud.com`).

## Build and release

```bash
pnpm install
npx tsc --noEmit && npx vitest run && npx prettier --check .
```

Release (GitHub Actions builds mac/win/linux):

```bash
# bump "version" in package.json first
git tag -fa vX.Y.Z -m "..." && git push -f origin vX.Y.Z
gh workflow run main.yml --repo xYossy/soundcloud-rpc --ref vX.Y.Z
```

`--ref <tag>` matters: the release step only runs when `github.ref` is `refs/tags/v*`. Push events did not trigger
workflows in this fork; `workflow_dispatch` does.

## Widevine / VMP signing

The app uses `castlabs/electron-releases` (ECS) because SoundCloud serves Widevine DRM. Production license servers
expect a VMP-signed package. Forks have no EVS credentials, so builds are unsigned (`STRICT_VMP_SIGNING=false` in the
workflow) and DRM-protected tracks may refuse to play.

To sign:

1. `python3 -m pip install castlabs_evs`
2. `python3 -m castlabs_evs.account signup` (free, e-mail confirmation)
3. `gh secret set EVS_USERNAME --repo xYossy/soundcloud-rpc` and `EVS_PASSWORD`
4. Re-run the tag build; `scripts/sign-vmp.js` signs the package.

## Syncing with upstream

```bash
git fetch upstream && git log --oneline HEAD..upstream/main
git rebase upstream/main
```

Expect conflicts in `audioMonitorService.ts` and `thumbarService.ts`.
