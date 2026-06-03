# Shoulder — Claude Code plugin

The security trust layer for AI-generated code, in Claude Code. Shoulder answers one question —
**"is this safe to ship?"** — with a decision-first verdict, a blast-radius statement, and the
source→sink data flows that prove it. Every reported attack path is a chain of real taint-graph
edges, not a regex match.

It ships:

- **`/shoulder:trust`** — a skill that runs trust assessments on code, diffs, and packages.
- **The Shoulder MCP server** — `trust`, `trust_diff`, and `trust_deps` tools that Claude calls
  proactively before installs, merges, and deploys.

Powered by [shoulder.dev](https://shoulder.dev).

## Install

```shell
/plugin marketplace add shoulderdev/plugins
/plugin install shoulder@shoulderdev
/reload-plugins
```

That's it. The MCP server runs the Shoulder CLI via `npx @shoulderdev/cli`, so there is **no
separate install step** — npm fetches the right platform binary on first launch and reuses it
after. This works identically on your machine and in Claude Code on the web.

> Want the CLI on your PATH for terminal use too? `npm install -g @shoulderdev/cli`. The plugin
> does not require it.

## Enable in cloud sessions and shared repositories

User-scoped plugins do not carry into Claude Code on the web — those sessions run on Anthropic
infrastructure, not your machine. To enable Shoulder there, or to turn it on for everyone who
clones a repository, declare it in the project's checked-in `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "shoulderdev": {
      "source": { "source": "github", "repo": "shoulderdev/plugins" }
    }
  },
  "enabledPlugins": {
    "shoulder@shoulderdev": true
  }
}
```

Commit that file. When a teammate trusts the folder — or when a cloud session opens the repo —
Claude Code registers the marketplace and enables the plugin automatically. The MCP server
self-provisions via `npx`, so nothing else is needed on the host.

## Usage

Run it explicitly:

```shell
/shoulder:trust
```

Or just ask naturally — *"is this repo safe?"*, *"what did I just change?"*, *"should I install
`left-pad@1.3.0`?"* — and Claude picks the right tool. Shoulder is also invoked **proactively**
before risky actions (installs, merges, deploys).

| Tool | Answers |
|---|---|
| `trust(project_root)` | "Is this codebase safe to ship / merge / deploy?" — covers code analysis **and** dependency CVE / malware checks. |
| `trust_diff(project_root, ref?)` | "What did this branch/PR change about risk?" |
| `trust_deps(packages[])` | "Is `<package>@<version>` safe to install?" |

## Coverage

- **Supply-chain & dependency risk** — checked across **every package ecosystem** (npm, PyPI, Go
  modules, Cargo, RubyGems, Composer, and more) against Shoulder's trust index.
- **Taint-flow code analysis** — deepest in **JavaScript / TypeScript**, solid in **Python** and
  **Go**. Other languages aren't analyzed for source→sink paths yet; the verdict says so when it
  matters.

## Structure

```
.claude-plugin/plugin.json       plugin manifest
.claude-plugin/marketplace.json  marketplace catalog (this repo is both)
skills/trust/SKILL.md            /shoulder:trust
.mcp.json                        MCP server (npx @shoulderdev/cli serve mcp)
hooks/hooks.json                 forwards all events to Shoulder's background watcher
.claude/settings.json            checked-in marketplace + enablement
```

## Develop locally

```bash
claude --plugin-dir .          # load this plugin without installing
claude plugin validate .       # validate marketplace.json + plugin manifest
/reload-plugins                # pick up edits without restarting
```

Verify it's active inside a session:

| Check | How |
|---|---|
| Skill appears | `/help` → look for `/shoulder:trust` |
| MCP server loaded | run `/plugin`, check the **Installed** tab, or `claude --debug` |
| Smoke test | `/shoulder:trust` — or ask *"is this repo safe?"* |

## Troubleshooting

**MCP server won't start** — confirm `npx -y @shoulderdev/cli --version` works in your shell. In
restricted environments without npm, install the CLI another way and ensure `shoulder` is on PATH.

**Skill doesn't appear in `/help`** — confirm the frontmatter `name:` in `skills/trust/SKILL.md`
matches the folder name, then `/reload-plugins`.

**Hooks don't show in `/hooks` after install/update** — hook subscriptions are wired at session
start, so `/reload-plugins` alone may not attach them. **Restart Claude Code** (or start a fresh
session) after installing or version-bumping. Cloud / Claude Code web sessions start fresh, so
hooks load there automatically. Note: `/hooks` aggregates hooks from *all* sources (your settings
plus every plugin), so its counts won't map 1:1 to this plugin — check the plugin's own detail
pane in `/plugin` to confirm Shoulder's hooks registered.

**Exit code 1 from a trust run** — expected when findings are present. Claude reads the output
regardless; it's not a command failure.
