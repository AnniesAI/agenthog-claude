---
name: agenthog-integrate
description: Install AgentHog analytics into a web app, a React Native / Expo app, a Capacitor hybrid app, or a Unity game, send server-side events from a backend, name events the way the AgentHog contract requires, and verify data is arriving. Use when adding analytics to a project, instrumenting events or conversions, relaying webhooks or backfilling history into analytics, or when AgentHog, ah.js, @brightmotion/agenthog-react-native, @brightmotion/agenthog-capacitor, or com.brightmotion.agenthog is mentioned.
when_to_use: The user asks to "add analytics", "set up AgentHog", "track signups", "instrument this app/game", "track subscriptions/revenue", "send events from our backend", "why is AgentHog not receiving events", or names any AgentHog package (including a Capacitor/Ionic app).
---

# Integrate AgentHog

AgentHog is web + mobile analytics with an agent-readable CLI. Integrating it means:
picking the right SDK for the project, wiring it in once, and confirming events land.

Most of the value is in **autocapture** — you do not hand-instrument pageviews, clicks,
forms, or scroll depth. Add the SDK, then add custom events only for things autocapture
cannot see (server-side outcomes, purchases, feature usage).

## 1. Get a project key

Every install needs a project key shaped `ah_xxxxxxxx`. If the user has not given you one:

- Ask them to create a site at `https://hog.brightmotion.io/sites/new` and paste the key, or
- If the `ah` CLI is installed and authenticated, run `ah projects list` and use an existing key.

AgentHog is in early access: new sites are approved by hand, so `/sites/new` may hand the
user a request form instead of a key. If so, stop and let them come back once approved —
there is nothing to integrate without a key.

Do not invent a key and do not proceed without one — the tracker silently disables itself
when `data-project` is missing.

## 2. Pick the platform

Inspect the repo before choosing:

- `ProjectSettings/ProjectVersion.txt` or a `Packages/manifest.json` with `com.unity.*`
  dependencies → it is a **Unity game** (§5)
- `package.json` with `react-native` or `expo` → **React Native install** (§4)
- `capacitor.config.ts` / `capacitor.config.json` (or `@capacitor/core` in package.json)
  → a **Capacitor hybrid app** (§6). Check this BEFORE the web rule — a Capacitor repo
  serves HTML too, but the web tag inside its WebView misclassifies every session.
  A `config.xml` with Cordova and NO Capacitor config is a **legacy Cordova app** the SDK
  does not target; if it is mid-migration to Capacitor, instrument the Capacitor version.
- anything else that serves HTML (Next.js, Astro, Remix/React Router, Rails, plain HTML) → **web install** (§3)

A repo can need both (marketing site + mobile app/game). They are separate projects in
AgentHog — use a different project key for each, so their stats stay clean.

Events that no client can witness — a billing webhook, a queue worker, a backfill of
existing history — are not an SDK install at all; they go straight to the ingest endpoint
from the backend (§7).

## 3. Web install

One script tag, before `</head>`, on every page:

```html
<script src="https://hog.brightmotion.io/ah.js" data-project="ah_xxxxxxxx" defer></script>
```

Put it wherever that framework renders `<head>` site-wide — `app/layout.tsx` (Next.js App
Router), `pages/_document.tsx` (Pages Router), `app/root.tsx` (React Router / Remix),
`src/layouts/*.astro` (Astro), the shared layout template otherwise. Do not add it per-page.

Optional attributes:

| attribute | when you need it |
|---|---|
| `data-api="https://your-agenthog.example"` | self-hosted backend on a different origin than the script |
| `data-cookie-domain=".example.com"` | share one visitor id across subdomains (`www.` and `app.`) |

What you get with no further work: pageviews (including SPA route changes — `pushState`,
`replaceState`, and `popstate` are all hooked), clicks, form submits, input focus, scroll
depth at 25/50/75/90/100%, and a `leave` event carrying time-on-page and max scroll.

The global API for custom work is `window.agenthog`:

```js
window.agenthog.capture('checkout_completed', { plan: 'pro', cents: 4900 })
window.agenthog.identify('user@example.com', { plan: 'pro' })  // email = cross-device stitch key
window.agenthog.tag('beta_cohort', true)
window.agenthog.register({ app_version: '2.1.0' })             // merged into every later event
```

Guard these behind a `typeof window !== 'undefined' && window.agenthog` check in SSR
frameworks — the object only exists in the browser, after the script runs.

## 4. React Native / Expo install

```bash
npm i @brightmotion/agenthog-react-native
npx expo install @react-native-async-storage/async-storage   # for persistent ids
```

**Always pass `storage`.** It is not auto-detected, and omitting it is silent data loss,
not a crash: ids and the queued event buffer stay in memory, so every app restart looks
like a brand-new visitor and returning-user metrics are wrong. Use `asyncStorage` from the
subpath, or any `{ getItem, setItem, removeItem }` adapter returning promises (MMKV,
SQLite). The subpath import is what pulls the optional peer into the bundle — importing it
without installing AsyncStorage fails the Metro build.

Wrap the app at its root — in Expo Router that is `app/_layout.tsx`, inside any gesture-handler
root and outside the navigator:

```tsx
import { AgentHogProvider } from '@brightmotion/agenthog-react-native'
import { asyncStorage } from '@brightmotion/agenthog-react-native/async-storage'

<AgentHogProvider config={{
  host: 'https://hog.brightmotion.io',
  projectKey: process.env.EXPO_PUBLIC_AGENTHOG_KEY ?? '',
  enabled: !!process.env.EXPO_PUBLIC_AGENTHOG_KEY,   // inert no-op when the key is absent
  appName: 'myapp',
  appVersion: '1.0.0',
  storage: asyncStorage,                             // omit → in-memory, ids reset every launch
}}>
  {children}
</AgentHogProvider>
```

The `npm i` above installs the current SDK, which is what the `/async-storage` subpath
needs. Much older releases (before 0.2.1) had no such subpath and tried to locate
AsyncStorage themselves — a lazy require Metro cannot resolve statically, producing a fatal
redbox (`Requiring unknown module`) that no `try/catch` suppresses. Upgrade rather than work
around it; if a project is genuinely stuck on one of those, pass the peer straight through:
`import AsyncStorage from '@react-native-async-storage/async-storage'` → `storage: AsyncStorage`.

Put the key in `.env` as `EXPO_PUBLIC_AGENTHOG_KEY=ah_xxxxxxxx`. The `enabled` gate above is
the idiom for keeping analytics off in local dev and in forks that have no key.

Screen views are not automatic — the SDK cannot know your router. Feed it a pathname from
whatever router the app uses:

```tsx
import { usePathname } from 'expo-router'
import { useScreenTracking } from '@brightmotion/agenthog-react-native'

useScreenTracking(usePathname())   // emits `pageview: <path>` + a `leave` for the previous screen
```

React Navigation has no `usePathname`; derive it from the navigation state
(`useNavigationContainerRef().getCurrentRoute()?.name`) and pass that.

Then, anywhere inside the provider:

```tsx
const ah = useAgentHog()
ah.capture('photo_sent', { recipients: 3 })
ah.identify(email, { clerk_id: user.id })
ah.reset()                                  // sign-out: new anon id + session
```

Tap autocapture is on by default (fiber walk → `click: <label>`); pass `autocapture: false`
to disable. For scroll-depth events on a scrollable screen, spread the hook onto the primary
scroller — one per screen, not every list:

```tsx
<FlatList {...useScrollDepth()} … />
```

## 5. Unity install

Add the UPM package to `Packages/manifest.json` (or Package Manager → *Add package from
git URL*):

```json
"com.brightmotion.agenthog": "https://github.com/AnniesAI/agenthog-unity.git?path=com.brightmotion.agenthog"
```

With no `#tag`, UPM resolves the latest default-branch commit when the package is added and
records it in `Packages/packages-lock.json`, so the version stays fixed until someone updates
it deliberately. To pin to a release instead, append the newest tag from
`https://github.com/AnniesAI/agenthog-unity/releases` (e.g. `#v0.3.0`).

Unity 2021.3+; pure C#, zero dependencies, no native plugins. Two ways to configure — pick
the settings asset unless the game already has a bootstrap script:

- **Settings asset (no code):** *Assets → Create → AgentHog → Settings*, save as
  `Assets/Resources/AgentHogSettings.asset`, fill in host + project key. The SDK
  initializes itself on startup. In a public/shared repo, commit that asset **blank** (the
  SDK stays inert) and put the real key in `Assets/Resources/AgentHogSettingsLocal.asset`
  (gitignored) — the `Local` variant takes precedence.
- **Code:** `AgentHog.Init(new AgentHogConfig { Host = "https://hog.brightmotion.io", ProjectKey = "ah_xxxxxxxx" })`
  once at startup. With a blank key or `Enabled = false` every call is a safe no-op, so
  call sites never need guards.

What is automatic: sessions (30-min idle, survives restarts), scene loads as
`pageview: /scene-name`, uGUI taps as `click: <label>`, per-screen time via `leave`
events, device context on every event, offline/crash carry-over. What is NOT automatic:
single-scene games with UI panels should call `AgentHog.Screen("/shop")` on panel changes,
and gameplay (world-space objects, UI Toolkit) is instrumented with `Capture`:

```csharp
AgentHog.Capture("level_complete", new Dictionary<string, object> { ["level"] = 12 });
AgentHog.Identify(traits: new Dictionary<string, object> { ["user_id"] = playerId });  // games rarely have emails — a stable user_id trait still stitches identity
AgentHog.Reset();   // sign-out: device becomes a new anonymous person
AgentHog.SetLandingParams(new Dictionary<string, string> { ["utm_source"] = "playstore" });  // install attribution — call before the first flush
```

Full docs: `https://github.com/AnniesAI/agenthog-unity`.

## 6. Capacitor install (hybrid apps)

```bash
npm i @brightmotion/agenthog-capacitor
npm i @capacitor/app @capacitor/device @capacitor/preferences   # required peers, all official plugins
npx cap sync
```

Requires Capacitor ≥5. Call `init` once at app bootstrap, before first render — vanilla
TS, no framework wrapper; Ionic React/Vue/Angular all consume it the same way:

```ts
import { AgentHog } from '@brightmotion/agenthog-capacitor'

await AgentHog.init({
  host: 'https://hog.brightmotion.io',              // https — iOS ATS applies to native requests too
  projectKey: import.meta.env.VITE_AGENTHOG_KEY ?? '',
  enabled: !!import.meta.env.VITE_AGENTHOG_KEY,     // inert no-op without a key (dev builds)
  appName: 'myapp',
  appVersion: '1.0.0',                              // ideally from App.getInfo()
})
```

Calls made before `init` resolves are buffered and replayed — nothing drops. Unlike RN
there is **no `storage` to configure**: ids and the offline queue persist through
`@capacitor/preferences` (native storage, survives WebView data eviction).

What is automatic: screen views from the router (history AND hash routing are hooked;
`trackScreens: false` + `AgentHog.screen('/path')` to drive manually), DOM autocapture
matching the web tracker (clicks with shadow-DOM-aware labels — Ionic components
included — form submits, input focus with field names only, scroll depth), 30-min-idle
sessions across app background/foreground, and a persisted offline queue with
crash/kill carry-over.

```ts
AgentHog.capture('run_logged', { miles: 3.1 })
AgentHog.identify(email, { user_id })          // sign-in / sign-up
AgentHog.reset()                               // sign-out: new anonymous person
AgentHog.onAttribution((a) => { /* install attribution result */ })
```

The transport detail that matters: on a device the SDK posts through CapacitorHttp
(native), so batches carry an app User-Agent (`AgentHogCap/…` — the session classifies
as mobile app traffic) and no Origin header (the project's domain allowlist never blocks
a native build). A **web/PWA build** of the same bundle falls back to plain `fetch` and
is treated as ordinary web traffic — correct, but it means a browser preview needs the
preview host in the project's domains.

## 7. Server-side events (no SDK)

Some outcomes have no client running when they happen: a subscription renews, a provider
webhook lands, a nightly job decides someone churned, or the user wants existing history
backfilled. Those go to the same `/ingest` endpoint from the backend, authenticated with a
**write-scope token** — an org admin mints it at `https://hog.brightmotion.io/tokens`.

```bash
curl -s https://hog.brightmotion.io/ingest \
  -H "Authorization: Bearer $AGENTHOG_INGEST_TOKEN" \
  -H 'content-type: application/json' \
  -d '{"project":"ah_xxxxxxxx","anonId":"server:42","sessionId":"<uuid>",
       "identify":{"traits":{"user_id":"42"}},
       "events":[{"ts":1754870400000,"type":"custom","name":"subscription_started",
                  "props":{"tier":"plus","price_usd":9.99}}]}'
```

What matters when you write the relay:

- The token is a **secret** — backend environment only, never a client bundle, an app
  binary, or a committed file. The `ah_` project key is public; the `ah_tok_` token is not.
- With the token, bot scoring is skipped (the session classifies as `server`) and event
  timestamps are trusted, which is what lets a backfill land on the right days. **Omitting
  the header still returns 204** — the batch just takes the anonymous path, gets bot-scored
  from your server's datacenter IP, and has its timestamps clamped to ±10 minutes of now.
- Conventions that make identity work: `anonId: "server:<user_id>"`, a fresh `sessionId`
  per delivery, and `identify` carrying the same email or `user_id` the app's
  `identify()` sends. That is what merges the server timeline with the device one.
- There is no idempotency key — a webhook provider that retries produces a duplicate event.
  Put the provider's own event id in props if dedupe matters.
- Make it fire-and-forget. Analytics must never fail the request that triggered it.
- Server sessions are excluded from `ah traffic` by design (a webhook is not a visit) and
  visible by default in `ah events`.

Full reference, including the subscription-lifecycle event names the mobile dashboard reads:
`https://hog.brightmotion.io/docs/server`.

## 8. Event naming — this is a contract, not a style preference

The CLI, funnels, and dashboard parse these exact shapes. Autocapture already emits:

```
pageview: /pricing        click: Join the waitlist      form_submit: waitlist
input: email              scroll: 75%                   leave: /pricing
```

Rules that matter when you add custom events:

- Custom event names are stored **verbatim**. Pick short, human-readable, stable names —
  `checkout_completed`, not `evt_CHECKOUT_v2` or a UUID.
- Use `snake_case` and keep the name free of interpolated values. Put the varying part in
  props: `capture('plan_selected', { plan: 'pro' })`, never `capture('plan_selected_pro')`.
  Props are queryable via `--by props.plan`; names baked with values are not.
- `identify(email)` is what stitches a person across devices. Call it at sign-in and after
  sign-up. Prefer email; a bare id still works but will not merge across devices.
- Never put secrets, tokens, or full PII blobs in props — props are readable by anyone with
  dashboard access.

## 9. Verify before you report success

Do not tell the user it works because the code compiles. Confirm data arrived:

1. Load the site (or run the app) and click through two or three screens.
2. Check, in order of preference:
   - `ah events --since 24h` — if the `ah` CLI is authenticated, this is the fastest proof
   - `ah digest` — sessions, sources, and top events in one report
   - the dashboard at `https://hog.brightmotion.io` otherwise
3. Expect a `pageview:` row within a few seconds — the web tracker flushes every 5s or every
   10 queued events; React Native, Unity, and Capacitor flush every 10s or every 20 events.
   Unity editor Play mode sends real events too (registered prop `platform: editor`).

If nothing arrives, work through §10 rather than adding more instrumentation.

## 10. When events are missing

| symptom | cause |
|---|---|
| console: `[agenthog] missing data-project — tracker disabled` | no `data-project` attribute, or it is an empty template value |
| no requests to `/ingest` at all | script blocked by an ad blocker, a CSP `script-src`, or never rendered — check the built HTML, not the source |
| web events stop after the first page | the tag was added to one page instead of the shared layout |
| RN: nothing at all | `enabled` resolved false because `EXPO_PUBLIC_AGENTHOG_KEY` was unset at bundle time — restart the bundler after editing `.env` |
| RN: no `pageview:` rows, but taps appear | `useScreenTracking` was never wired to the router |
| every session is a new visitor | no `storage` adapter passed (RN — it is not auto-detected), or cookies blocked (web) |
| RN redbox: `Requiring unknown module "@react-native-async-storage/async-storage"` | an old SDK (before 0.2.1) self-detecting the peer; upgrade to the current SDK and pass `storage` explicitly |
| Unity: nothing at all | the settings asset has a blank key (committed-blank is the intended public-repo default — the real key belongs in `AgentHogSettingsLocal.asset`), or `Init` never ran. Set `debugLog = true` and watch the Console for `[AgentHog]` lines |
| Unity: clicks show GameObject names like `click: BtnStart2` | the pressed control has no `Text`/TMP child — autocapture falls back to the GameObject name. Add label text or rename the object |
| Unity: gameplay taps missing | autocapture covers uGUI (Canvas) only — world-space objects and UI Toolkit need explicit `Capture` calls |
| Capacitor: sessions classify as Chrome/Safari and miss the mobile dashboard | that was a web/PWA run of the bundle (browser preview) — only native builds take the CapacitorHttp path that sets the app UA. If it IS a device build, check `@capacitor/core` ≥5 |
| Capacitor: `403` from `/ingest` | the same web fallback hitting the domain allowlist — add the preview host to the project's domains. Native builds send no Origin and are never allowlist-blocked |
| Capacitor: taps but no `pageview:` on navigation | the router changes neither the URL path nor a `#/` hash — call `AgentHog.screen('/path')` where it navigates |
| server relay: `401` / `403` from `/ingest` | token invalid or revoked (401), or read-scope / wrong project (403). A bad token is a hard fail by design — it never silently falls back |
| server relay: 204s, but events are bot-scored or all dated today | the `Authorization` header is missing, so the batch took the anonymous path (§7) |
| traffic looks inflated | you are seeing bots; aggregates exclude them by default, `--all` includes them |
| your own testing shows up as users | run `ah test-ips add` from that network (no ip needed — the server uses the address you call from). Its traffic becomes `classification: test` — kept, but out of the default filter, and applied retroactively to what it already did. `ah test-ips rm <ip>` reverses it |
| test traffic still appears after listing the IP | the device is probably on cellular, where the carrier IP churns — a list only matches stable (wifi) IPs |

## 11. After integrating

Two follow-ups worth offering the user:

- **Define a conversion goal** so sessions get marked converted:
  `ah goals set signup "form_submit: waitlist" --project <name>`
- **Give their agent the read surface**: install the `ah` CLI (`npm i -g @brightmotion/agenthog`,
  then `ah login`) and add a short AgentHog section to `CLAUDE.md` so future sessions query
  analytics instead of guessing. Full CLI docs: `https://hog.brightmotion.io/docs/cli`.
