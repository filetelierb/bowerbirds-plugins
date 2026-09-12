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
- `/bowerbirds-route <bucket>` runs one routing cycle as the bucket's **external router**: it claims the cycle, reads the routes, fetches the unsigned records, reads each record's findings (`get_record_context`: the summary and the findings with their chunk references, `get_media_chunk` when one finding needs the chunk behind it), searches the destination for what already exists (`search_records`, `get_record`, and the route's connector read tools), decides per record per route (merge, derive, create, append, link or none), acts only through `derive_record` and the route's enabled connector tools, signs each record with its status, links and the finding ids each rested on, and releases the cycle. It never moves or trashes a record, and a clamp from the gate stops that record and is reported verbatim.

## Codex and ChatGPT

The `codex/` directory is an Agent Plugin: `plugin.json`, `mcp.json` naming the remote server, the `bowerbirds-records` and `bowerbirds-route` skills, `prompts/bowerbirds-route.md` (the same routing loop as a custom prompt: `/prompts:bowerbirds-route <bucket>`), and the `.codex-plugin/` compatibility manifest. Install it from this path in the Codex CLI, or register the MCP endpoint in ChatGPT's developer mode; publishing to a ChatGPT workspace or to the OpenAI directory goes through the ChatGPT Plugins menu.

## What the agent can do

Fetch records with filters (kind, path, changed-since, signed or not), read one whole with fetchable payload URLs, file captures and notes, move and trash records under the same rules a person has, and sign records to mark them handled. See [Bowerbirds for coding agents](https://github.com/filetelierb/bowerbirds/blob/main/docs/mcp.md).

## Routing a bucket from your own agent

A bucket has at most one router. Make it an external one whose brain is your agent: mint a workspace token with the `router` role (`bowerbirds cloud tokens mint --role router`, `bowerbirds cloud tokens login --role router` through the browser approval page, or Developer Tools in the app), attach a router of kind `external` to the bucket naming that token (`bowerbirds cloud routers create --kind external --token <id> --name <name> --bucket <owner/slug>`), and give it routes — destinations, linked connectors with per-tool grants, a description and rules; the **Bugs to Linear** template is one tap. Then `/bowerbirds-route <bucket>` from your terminal is the cycle: your model, your tools, zero platform credits, and the records end up with the same route marks, router status and destination links a native router writes. The router role can add and sign only; the move and trash doors refuse it.

The five routing tools, advertised to a router token beside the others:

| Tool | What it does |
| --- | --- |
| `list_routes` | The router's routes with destinations, connectors and tool grants, description, goal, rules and trust; whether this token is the brain; the running cycle. |
| `claim_cycle` / `renew_cycle` / `release_cycle` | One run at a time on the bucket, under a lease; the cycle id the marks carry. |
| `derive_record` | A new record from a source (split or augmented), with provenance and the source's bytes copied; judged by the gate; the source is never written. With `mergeInto` it joins an existing Package as its next scene, with `related` it makes a new Package of the related records and itself — the **merge** decision. |
| `sign_items` (`items` shape) | The record's route marks and router mark, with a status (`kept`, `processed`, `routed`, `pending`) and one link per route (`derive`, `merge`, `create`, `append`, `link`, `none`) naming the destination item and the finding ids the decision rested on. |

Three more read and compute a record's **findings** — what the platform's media processor condensed out of a recording, an audio file, a PDF or a set of pictures. It splits long media into overlapping chunks (video and audio at most thirty minutes with two minutes of overlap, a PDF at most fifty pages, pictures in groups of twenty), reads each chunk, has a second reader check nothing was missed, and files one document: a summary paragraph, findings with chunk references, and the chunk table. The chunk limits are the platform's knobs (`internal/mediactx/knobs.go`), restated here for a reader; the [records skill](claude/skills/bowerbirds-records/SKILL.md) carries the worked numbers.

| Tool | What it does |
| --- | --- |
| `get_record_context` | The record's findings document: the `summary` paragraph, the `findings` (each a detailed description of one thing the media establishes, citing the chunk and the moment, pages or scenes it was found at, with verbatim quotes when they matter), and the `chunks` table — or one `part` (`summary`, `findings`, `chunks`). Any role; free. Read it before deciding. |
| `get_media_chunk` | What the processor observed in ONE chunk as it read it, behind the findings' descriptions: `{id, ref, observations, quotes}`. Router role; free — the observations were stored when the media was read, so reading them costs no model call. Open the chunk a finding cites when its description is not enough. |
| `process_media` | Computes a record's findings now through the platform's processor. The bucket's own router only, and it **costs the workspace's credits**: use it only when the record has no findings. Your own model is free; the platform's processor is not. A record whose findings are current answers from cache at no cost. |

And two search the bucket itself, so a route can look before it writes:

| Tool | What it does |
| --- | --- |
| `search_records` | One bucket's records — titles, text and a capture's comments — searched for what already exists there, optionally scoped to a folder. Answers id, title, path, kind and a snippet. Free. |
| `get_record` | One record whole: id, title, path, kind, tags, content and a capture's numbered comments, with the payload manifest. Free. |
