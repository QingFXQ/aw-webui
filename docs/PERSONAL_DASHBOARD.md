# Personal Dashboard

This fork keeps ActivityWatch's collectors, AFK detection, storage, categorization, sync model, and query semantics intact while replacing the default landing experience with a simpler personal statistics dashboard.

## Counting rule

The personal dashboard currently reports foreground application activity with AFK time filtered out.

- Desktop: foreground window events intersected with non-AFK periods.
- Multiple devices: ActivityWatch's existing multi-device query is used; overlapping events are resolved with `union_no_overlap`, so simultaneous use is not blindly summed.
- Android / ScreenTime-only devices: app-usage events are treated as active because those sources do not provide desktop-style AFK buckets.
- Audible browser activity is not added as active time on the personal dashboard. The advanced Activity page keeps the upstream controls for that behavior.
- Stopwatch sessions are not mixed into the personal dashboard totals.

The dashboard must not silently change these rules. If the definition changes later, update both this document and the UI note.

## Dashboard queries

The landing page intentionally uses the existing aggregate queries directly instead of loading the entire Activity view state machine for every card.

Normal first-screen data:

- today aggregate
- current week aggregate
- current month aggregate
- one 7-day aggregate for the trend on desktop/multi-device sources

Single mobile sources fall back to one query per day for the 7-day trend because their aggregate result does not retain positioned active events.

Lifetime statistics are loaded after the first screen and split into 92-day chunks. This avoids a single large request that can exceed the ActivityWatch server request timeout on a long history.

## Local development against the normal ActivityWatch database

The upstream WebUI development server normally targets the testing server on port `5666`. To use the normal local ActivityWatch server on port `5600`, run the production ActivityWatch app/server first, allow the WebUI development origin in the server CORS configuration, then run the WebUI with an explicit server URL.

Typical macOS/Linux flow:

```bash
git clone https://github.com/QingFXQ/aw-webui.git
cd aw-webui
git checkout feat/steam-style-dashboard
npm ci
AW_SERVER_URL="'http://localhost:5600'" npm run serve
```

The development WebUI is served on `http://localhost:27180` by default. The ActivityWatch server may need `cors_origins = http://localhost:27180` in its configuration.

Do not expose the ActivityWatch server to the public network just to make this development setup work.

## Production-style build

```bash
npm ci
npm run lint
npm run build
```

The output is in `dist/`.

ActivityWatch supports two useful ways to try a custom WebUI build:

1. Replace the server's WebUI static assets with the `dist/` contents, keeping a backup of the original assets.
2. With `aw-server-rust`, run the server with `--webpath /path/to/aw-webui/dist` so the custom UI can be tested without copying assets into the installation.

The upstream WebUI README documents both mechanisms. Prefer `--webpath` while iterating because reverting is trivial.

## Focused GitHub Actions gate

`.github/workflows/personal-dashboard.yml` runs:

1. `npm ci`
2. `npm run lint`
3. `npm run build`
4. uploads `dist/` as the `activitywatch-personal-webui` artifact

It runs on pushes to `feat/steam-style-dashboard`, pull requests into `master`, and manual dispatch.

If the fork has GitHub Actions disabled, no workflow will start until Actions are enabled for the fork. Do not treat the absence of a run as a successful build.

## Smoke checklist

Before merging the dashboard branch:

- [ ] Dashboard loads with one desktop device.
- [ ] Dashboard renders a useful empty state with no eligible device.
- [ ] Today / week / month totals render and update on refresh.
- [ ] Weekly app ranking renders.
- [ ] Lifetime total starts in the background and eventually completes.
- [ ] Lifetime app ranking can be selected.
- [ ] 7-day trend renders.
- [ ] Category totals render when category rules exist.
- [ ] Timeline and Settings remain reachable.
- [ ] Multi-device selector can switch between one device and all devices.
- [ ] All-device total does not blindly double-count overlapping device time.
- [ ] Repeated refresh / device switching does not let an older lifetime query overwrite the new selection.
- [ ] Narrow/mobile layout remains readable.

## Known limitation

Lifetime app ranking combines the app aggregates returned by each 92-day chunk. The upstream aggregate query caps the returned app list per chunk. In an extreme history, an app could rank below that cap in every individual chunk but rank inside the all-time top list after cross-chunk summation. This is intentionally accepted for the personal fork instead of changing a shared upstream query API solely for a low-probability edge case.
