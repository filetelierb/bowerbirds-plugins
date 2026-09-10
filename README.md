# Bowerbirds plugins

Bowerbirds for coding agents, packaged for the two plugin ecosystems. Both bundle the same thing: the **remote MCP door** (`https://bowerbirds-cloud-y7wboabaoa-uc.a.run.app/mcp`), which the agent's client authorizes in your browser through OAuth — you pick the workspace and the role, and you can revoke the connection at any time from Bowerbirds' Developer Tools — plus a skill that teaches the agent how records, payload URLs and signatures work.

## Claude Code

```sh
claude plugin marketplace add filetelierb/bowerbirds-plugins
claude plugin install bowerbirds@bowerbirds
```

Or, without a marketplace, straight from a checkout:

```sh
claude --plugin-dir ./claude
```

The plugin adds the `bowerbirds` MCP server, the `bowerbirds-records` skill and a `/bowerbirds-triage` command that sweeps a bucket's unsigned records, acts on each and signs it.

## Codex and ChatGPT

The `codex/` directory is an Agent Plugin: `plugin.json`, `mcp.json` naming the remote server, the `bowerbirds-records` skill, and the `.codex-plugin/` compatibility manifest. Install it from this path in the Codex CLI, or register the MCP endpoint in ChatGPT's developer mode; publishing to a workspace or to the OpenAI directory goes through the ChatGPT Plugins menu.

## What the agent can do

Fetch records with filters (kind, path, changed-since, signed or not), read one whole with fetchable payload URLs, file captures and notes, move and trash records under the same rules a person has, and sign records to mark them handled. See [Bowerbirds for coding agents](https://github.com/filetelierb/bowerbirds/blob/main/docs/mcp.md).
