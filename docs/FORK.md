# Fork notes — xYossy/soundcloud-rpc

Fork of [richardhbtz/soundcloud-rpc](https://github.com/richardhbtz/soundcloud-rpc), Windows-first.

## What differs from upstream

- **Reliable like state** (`src/services/audioMonitorService.ts`): the like button's `class` is watched with a
  `MutationObserver` instead of a single 50 ms read. Works while paused, catches slow API responses and reverts. Click
  fallbacks at 300 ms and 1000 ms.
- **Selector fallbacks**: `.playbackSoundBadge__like` → `.playbackSoundBadge .sc-button-like` → `.sc-button-like` in
  both the monitor script and the Windows thumbar.
- **Thumbar errors are logged** instead of swallowed; the click reports whether the button was found.
- **Thumbar play/pause tooltip** updates `buttons[2]` (was `buttons[1]`, i.e. the previous button).
- `workflow_dispatch` trigger so releases can be started from a tag in a fork.

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
