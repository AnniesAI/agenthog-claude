# AgentHog for Claude Code

Teaches your coding agent to install [AgentHog](https://agenthog.io) analytics, run A/B
tests, and write shared reports with live charts, so you can ask in plain language instead
of reading SDK docs.

## Install

On Claude Code, install the plugin. It carries all three skills, and updates arrive with
`/plugin marketplace update agenthog`:

```
/plugin marketplace add AnniesAI/agenthog-claude
/plugin install agenthog@agenthog
```

On any other agent — Codex, Cursor, Copilot, Gemini, Zed, Windsurf and around seventy more —
the same three skills install in one command, into whichever directory that agent reads.
It asks which agents and whether to scope them to this repo or your whole machine, and
`npx skills update` refreshes them later:

```
npx skills add AnniesAI/agenthog-claude
```

Run one or the other, not both: in Claude Code the two paths install the same three skills
side by side, and the agent loads each of them twice.

Pointing an agent at a **self-hosted** AgentHog? Take the markdown from that deployment
instead — the copies here are pinned to `https://agenthog.io`, while
[agenthog.io/docs/skill](https://agenthog.io/docs/skill) serves each file with the host
rewritten to whatever origin served it.

You need an AgentHog project to point it at — the Free plan is enough for everything the
skills do, no credit card. See [Prerequisites](#prerequisites).

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

**`agenthog:agenthog-report`** — writes up the numbers as a shared report: a Markdown file
with live charts that everyone in the organization reads in the AgentHog dashboard. Learns
the house format from an existing report, decides what stays live and at what TTL, excludes
bot traffic in every live query, previews until clean, publishes, verifies by reading it
back, and revises the same report on a schedule so its history stays intact.

> "write up this week's numbers" · "make a report for the launch" · "update the weekly
> report" · "why is that widget showing an error"

## Prerequisites

Free to try. The Free plan covers 1M events a month, 30 days of history, 1 project and 3
team members, with no credit card — and every feature the skills use is on it: autocapture,
funnels, revenue, experiments, shared reports, full CLI and API. Paid plans buy history,
more projects and a bigger team ([pricing](https://agenthog.io/pricing)).

- An AgentHog project key (`ah_xxxxxxxx`). Create a project at
  [agenthog.io/projects/new](https://agenthog.io/projects/new).
- For experiments, reports, and for the agent to read your analytics back, the CLI:

  ```
  npm i -g @brightmotion/agenthog
  ah login          # add --write for experiments, reports and server-side events
  ```

## Docs

- Dashboard — [agenthog.io](https://agenthog.io)
- CLI — [agenthog.io/docs/cli](https://agenthog.io/docs/cli)
- Shared reports — [agenthog.io/docs/reports](https://agenthog.io/docs/reports)
- Server-side ingest — [agenthog.io/docs/server](https://agenthog.io/docs/server)
- Unity SDK — [github.com/AnniesAI/agenthog-unity](https://github.com/AnniesAI/agenthog-unity)

## License

MIT — see [LICENSE](LICENSE).
