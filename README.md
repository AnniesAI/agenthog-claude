# AgentHog for Claude Code

Teaches your coding agent to install [AgentHog](https://agenthog.io) analytics and
run A/B tests, so you can ask in plain language instead of reading SDK docs.

## Install

```
/plugin marketplace add AnniesAI/agenthog-claude
/plugin install agenthog@agenthog
```

## What you get

**`agenthog:agenthog-integrate`** — installs the right SDK for the project and confirms
events actually arrive. Covers web (one script tag), React Native / Expo, Capacitor hybrid
apps, Unity games, and server-side events with no SDK at all. Also carries the event-naming
contract the CLI and dashboard parse, and a troubleshooting table for when events go missing.

> "add analytics to this app" · "track signups" · "send events from our backend" ·
> "why is AgentHog not receiving events"

**`agenthog:agenthog-experiment`** — runs an A/B test end to end: create the flag, wire the
variant read with a hard fallback, verify exposure is flowing *before* the window opens,
pick the metric and horizon, read results without calling early winners, ship, and retire
the flag.

> "set up an A/B test" · "roll this out to 10%" · "check the experiment results" ·
> "ship the winner"

## Prerequisites

- An AgentHog project key (`ah_xxxxxxxx`). Create a project at
  [agenthog.io/projects/new](https://agenthog.io/projects/new). Project count
  is capped by plan (Free: 1).
- For experiments and for the agent to read your analytics back, the CLI:

  ```
  npm i -g @brightmotion/agenthog
  ah login          # add --write for experiments and server-side events
  ```

## Docs

- Dashboard — [agenthog.io](https://agenthog.io)
- CLI — [agenthog.io/docs/cli](https://agenthog.io/docs/cli)
- Server-side ingest — [agenthog.io/docs/server](https://agenthog.io/docs/server)
- Unity SDK — [github.com/AnniesAI/agenthog-unity](https://github.com/AnniesAI/agenthog-unity)

## License

MIT — see [LICENSE](LICENSE).
