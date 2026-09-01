# AgentHog

Teaches your coding agent to install [AgentHog](https://agenthog.io) analytics, run A/B
tests, and write shared reports with live charts, so you can ask in plain language instead
of reading SDK docs.

Three skills, no MCP server and no hooks — see [What this plugin does](#what-this-plugin-does)
for exactly what it touches.

## Install

```
/plugin marketplace add AnniesAI/agenthog-claude
/plugin install agenthog@agenthog
```

Skills load on demand; ask for what you want and the right one triggers.

## Skills

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

**`agenthog:agenthog-report`** — writes up the numbers as a shared report: a Markdown file
with live charts that everyone in the organization reads in the AgentHog dashboard. Learns
the house format from an existing report, decides what stays live and at what TTL, excludes
bot traffic in every live query, previews until clean, publishes, verifies by reading it
back, and revises the same report on a schedule so its history stays intact.

> "write up this week's numbers" · "make a report for the launch" · "update the weekly
> report" · "why is that widget showing an error"

## Prerequisites

Free to try — the Free plan covers 1M events a month, 1 project, 3 team members and 30 days
of history, with no credit card. Experiments, funnels, revenue and reports are all on it;
paid plans buy history, more projects and a bigger team. See
[agenthog.io/pricing](https://agenthog.io/pricing).

- **A project key** shaped `ah_xxxxxxxx`. Create a project at
  [agenthog.io/projects/new](https://agenthog.io/projects/new). The key is public — it ships
  in your client bundle.
- **The CLI**, for experiments, reports, and for the agent to read your analytics back:

  ```
  npm i -g @brightmotion/agenthog
  ah login          # add --write for experiments, reports and server-side events
  ```

Integration alone needs only the project key; the agent will ask for the CLI when a task
requires it.

## What this plugin does

The plugin itself is three `SKILL.md` files. It bundles no MCP server, no hooks, no
executables, and nothing runs at install time. What the skills then direct the agent to do,
with your approval at each step:

- **Edit your project** — add an SDK (a `<script>` tag, an npm package, or a Unity package)
  and the event calls around it.
- **Run the `ah` CLI** — reads (events, funnels, experiment results, reports) and, with a
  write-scope token, mutations (create flags, open experiments, publish reports).
- **Send data to agenthog.io** — your app's analytics events go to `https://agenthog.io/ingest`,
  and the web SDK loads `https://agenthog.io/ah.js` and reads flags from
  `https://agenthog.io/flags`. Nothing is sent anywhere else.

On secrets: the `ah_` project key is public, but the `ah_tok_` write token minted at
[agenthog.io/tokens](https://agenthog.io/tokens) is not. The skills instruct the agent to
keep it in the backend environment only — never a client bundle, an app binary, or a
committed file — and to keep tokens and PII out of event props, which are readable by
anyone with project access.

## Docs

- Dashboard — [agenthog.io](https://agenthog.io)
- CLI — [agenthog.io/docs/cli](https://agenthog.io/docs/cli)
- Shared reports — [agenthog.io/docs/reports](https://agenthog.io/docs/reports)
- Server-side ingest — [agenthog.io/docs/server](https://agenthog.io/docs/server)
- Unity SDK — [github.com/AnniesAI/agenthog-unity](https://github.com/AnniesAI/agenthog-unity)

## License

MIT — see [LICENSE](../../LICENSE).
