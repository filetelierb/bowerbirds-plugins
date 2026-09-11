# Bowerbirds plugins

Bowerbirds for coding agents, packaged for the two plugin ecosystems. Both bundle the same thing: the **remote MCP door** (`https://bowerbirds-cloud-y7wboabaoa-uc.a.run.app/mcp`), which the agent's client authorizes in your browser through OAuth — you pick the workspace and the role, and you can revoke the connection at any time from Bowerbirds' Developer Tools — plus a skill that teaches the agent how records, payload URLs, signatures and routing work.

The same repository is also a **Bowerbirds workspace plugin**: `bowerbirds.plugin.json` at the root is the manifest a workspace installs (`POST /plugins/install {"source": "filetelierb/bowerbirds-plugins"}`, or the Plugins page in the apps). Installing brings the `bowerbirds-records` skill into the workspace, the route template **Bugs to Linear** (agentic: bug captures become Linear issues, with `search_issues`, `get_issue`, `create_issue` and `create_comment` enabled and the delete tools off), and a declaration of Linear's remote MCP server (`https://mcp.linear.app/mcp`, OAuth) that lands in "needs connecting" until someone connects it. Removing the plugin takes back only what it brought.

## Claude Code

```sh
claude plugin marketplace add filetelierb/bowerbirds-plugins
claude plugin install bowerbirds@bowerbirds
```

Or, without a marketplace, straight from a checkout:

```sh
claude --plugin-dir ./claude
```

The plugin adds the `bowerbirds` MCP server, the `bowerbirds-records` skill and two commands:

- `/bowerbirds-triage <bucket>` sweeps a bucket's unsigned records, acts on each in the repository you are in, and signs it.
- `/bowerbirds-route <bucket>` runs one routing cycle as the bucket's **external router**: it claims the cycle, reads the routes, fetches the unsigned records, decides per record per route (derive, create, append, link or none), looks at the destination with the route's read tools before acting, acts only through `derive_record` and the route's enabled connector tools, signs each record with its status and links, and releases the cycle. It never moves or trashes a record, and a clamp from the gate stops that record and is reported verbatim.

## Codex and ChatGPT

The `codex/` directory is an Agent Plugin: `plugin.json`, `mcp.json` naming the remote server, the `bowerbirds-records` and `bowerbirds-route` skills, `prompts/bowerbirds-route.md` (the same routing loop as a custom prompt: `/prompts:bowerbirds-route <bucket>`), and the `.codex-plugin/` compatibility manifest. Install it from this path in the Codex CLI, or register the MCP endpoint in ChatGPT's developer mode; publishing to a workspace or to the OpenAI directory goes through the ChatGPT Plugins menu.

## What the agent can do

Fetch records with filters (kind, path, changed-since, signed or not), read one whole with fetchable payload URLs, file captures and notes, move and trash records under the same rules a person has, and sign records to mark them handled. See [Bowerbirds for coding agents](https://github.com/filetelierb/bowerbirds/blob/main/docs/mcp.md).

## Routing a bucket from your own agent

A bucket has at most one router. Make it an external one whose brain is your agent: mint a workspace token with the `router` role (`bowerbirds cloud tokens mint --role router`, or Developer Tools in the app), attach a router of kind `external` to the bucket naming that token, and give it routes — destinations, linked connectors with per-tool grants, a description and rules; the **Bugs to Linear** template is one tap. Then `/bowerbirds-route <bucket>` from your terminal is the cycle: your model, your tools, zero platform credits, and the records end up with the same route marks, router status and destination links a native router writes. The router role can add and sign only; the move and trash doors refuse it.

The five routing tools, advertised to a router token beside the others:

| Tool | What it does |
| --- | --- |
| `list_routes` | The router's routes with destinations, connectors and tool grants, description, goal, rules and trust; whether this token is the brain; the running cycle. |
| `claim_cycle` / `renew_cycle` / `release_cycle` | One run at a time on the bucket, under a lease; the cycle id the marks carry. |
| `derive_record` | A new record from a source (split or augmented), with provenance and the source's bytes copied; judged by the gate; the source is never written. |
| `sign_items` (`items` shape) | The record's route marks and router mark, with a status (`kept`, `processed`, `routed`, `pending`) and one link per route (`derive`, `create`, `append`, `link`, `none`) naming the destination item. |
