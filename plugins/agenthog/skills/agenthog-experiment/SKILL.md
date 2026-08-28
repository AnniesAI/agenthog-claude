---
name: agenthog-experiment
description: Run an A/B test end-to-end with AgentHog as the source of truth — create a feature flag, wire the variant read into the app, verify exposure is flowing, open an experiment window with the right metric and horizon, read results honestly, ship the winner, and retire the flag. Use when setting up an A/B or A/B/n test, ramping a rollout percentage, checking experiment results, or cleaning up finished flags.
when_to_use: The user asks to "set up an A/B test", "run an experiment", "roll this out to 10%", "check the experiment results", "ship the winner", "kill that feature flag", or names an AgentHog flag/experiment. For installing AgentHog itself, use the agenthog-integrate skill first.
---

# Run an experiment with AgentHog

An experiment is an analysis window on a multivariate feature flag. The flag delivers
variants (deterministic per-user hashing in the SDK — no assignment service); the
experiment compares treatments against a control on a metric and reports each arm's
chance to beat it. Everything runs through the `ah` CLI; every mutation needs a
write-scope token.

Follow the phases in order. Each ends with a check — do not proceed past a failed check.

## 0. Preconditions

- The AgentHog SDK is integrated and verified in the target app (events arrive in
  `ah events`). If not: stop and run the **agenthog-integrate** skill first.
- `ah` is authenticated with **write** scope: `ah whoami` shows the project and
  `scope: write`. If read-only, ask the user to run `ah login --write`.
- You know the **decision metric**: one event name that already exists (check
  `ah events top`) or a goal (`ah goals list`). If the metric event doesn't exist yet,
  instrument it and verify it arrives before doing anything else.

## 1. Create the flag

```
ah flags create checkout_cta --variants control:50,b:50 --desc "new checkout CTA copy"
```

- Key: lowercase `[a-z0-9_-]`, ≤64 chars. Variant names the same. One arm should be
  named `control` unless the user says otherwise.
- A/B/C: `--variants control:34,b:33,c:33`. Percentage rollout of a non-experiment
  feature: `ah flags create new_nav` (boolean) then `ah flags rollout new_nav 10`.
- Start experiments at full traffic (`--traffic 100`) unless the user wants a limited
  blast radius; you can ramp up later — ramping up never reshuffles assignments.

## 2. Wire the read into the app — with a hard fallback

The pattern is identical on every platform: read the flag, branch, and make the
fallback path the control experience. `undefined` means "no ruleset yet / not
enrolled / killed" — the app must work when the flag system says nothing.

Web (`window.agenthog`), React Native (`useAgentHog()`), Capacitor (`AgentHog`):

```js
await agenthog.flagsReady()                 // avoids the first-visit undefined window
const v = agenthog.flag('checkout_cta')     // 'control' | 'b' | undefined
if (v === 'b') { /* treatment */ } else { /* control AND fallback */ }
```

Never emit `$exposure` events or `$ff/*` props yourself — the SDK does both on the
first `flag()` read. Never branch on anything except the `flag()` return value.

**Check before continuing** (exposure must flow BEFORE the window opens):

```
ah events --name '$exposure' --since 1h            # rows appear as you exercise the app
ah events --by flag:checkout_cta --since 1h        # events split by variant
```

Use `agenthog.overrideFlag('checkout_cta', 'b')` to force each arm during this check —
overrides never pollute experiment data. Clear with `overrideFlag(key, null)`.

## 3. Open the experiment window

```
ah experiments start checkout_cta --metric checkout_completed \
  [--min-days 1|7] [--secondary e1,e2] [--guardrail app_error:up] [--hypothesis "..."]
```

- `--min-days` gates only the verdict badge. Choose by metric horizon: `1` when the
  metric fires in the same session as exposure (onboarding, activation, checkout);
  keep the default `7` for anything with weekly rhythm (retention, repeat usage).
- Guardrail direction is the REGRESSION direction: `errors:up`, `activation:down`.
- The command prints a 30-day baseline and required-users-per-arm for 5/10/20% MDEs.
  **Relay this to the user** — if the estimate says months, say so now, not after.

## 4. Read results honestly

```
ah experiments results checkout_cta [--json]
```

- `collecting (n of 100; day d of N)` — report progress, do NOT characterize which
  arm is ahead. Small-sample rates mislead.
- `readable`, no ★ — report chance-to-beat and credible intervals with the caveats
  printed. "68% likely better" is not a winner; say what would settle it (more users
  or more days).
- `★ likely winner` — safe to recommend shipping. Quote the lift and chance-to-beat.
- Repeat any `caveat:` lines verbatim (day-of-week coverage, excluded multi-variant
  users). If a guardrail shows ⚠, lead with that — a primary-metric win that spikes
  errors is not a win.
- `--json` field names are a stable contract for scripting.

Do not stop a live experiment, change its weights, or declare winners the tool did
not declare, unless the user explicitly asks (weight edits require `--force` for
exactly this reason).

## 5. Ship, iterate, or abandon

- **Ship**: `ah experiments stop checkout_cta --winner b --conclusion "..."` then
  `ah flags weights checkout_cta b:100`. The flag now serves the winner to everyone.
- **Iterate**: stop (no winner), adjust variants/copy, `ah experiments start` again —
  a fresh window on the same flag; results never mix across windows.
- **Abandon / emergency**: `ah flags disable checkout_cta` is the kill switch — every
  client falls back to code defaults within ~2 minutes.

## 6. Retire

Weeks later, once the winner is the only path anyone gets:

1. Delete the losing branch and the `flag()` read from the code; the winner's code
   becomes the only path.
2. `ah flags archive checkout_cta` — leaves the SDK payload, keeps history.
3. Confirm: `ah events --name '$exposure' --since 48h` shows nothing for the key
   after the archived deploy is live.

## You are done when

- [ ] `ah flags show <key>` matches what the user asked for (variants, traffic, state)
- [ ] `$exposure` events arrive and `ah events --by flag:<key>` splits by variant
- [ ] The experiment window is live with the right metric and `--min-days`, and the
      user knows the expected duration from the baseline hint
- [ ] You reported results with their gates and caveats intact — no early winners
- [ ] On ship: winner at 100% weight, experiment stopped with `--winner`, and (when
      retiring) the flag archived and its code path deleted

Timeline of everything you did: `ah changes list --kind experiment`. Flags and
results are also visible at https://agenthog.io/flags.
