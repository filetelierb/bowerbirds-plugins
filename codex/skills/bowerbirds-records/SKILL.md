---
name: bowerbirds-records
description: Read, file, sign and route records in a Bowerbirds workspace through the bowerbirds MCP server: captures with numbered comments, credential-free payload URLs, the signature that marks what this agent already handled, the findings the platform's media processor filed for a recording, audio, PDF or pictures, and the router tools an external router uses. Use whenever a task mentions Bowerbirds, a bucket, a capture, a screenshot someone filed for you, or routing records.
---

# Bowerbirds records

Bowerbirds is where people file annotated screenshots (captures), notes, dictations and screen recordings for their agents. You reach a workspace through the `bowerbirds` MCP server; the first call opens an approval page in the person's browser, where they pick the workspace and the role you get. You act as your own token, never as the person.

## The records

- A **capture** is a picture with numbered pins. Its `content` is the list of comments, one per pin, in the same numbering. `get_capture` also answers `comments` parsed out of the content.
- A record's payloads are listed in `get_capture` as `{name, role, mimeType, size, url}`. For a capture: `annotated.png` (role `annotated`, the picture with the pins drawn on it), `raw.png` (role `original`), `session.json` (role `capture-session`, the editable annotations). Recordings carry video and audio files; a record whose media has been read also carries `context.json` (the findings document) and `context-chunks.json` (what the processor observed chunk by chunk).
- **Payload bytes never come through MCP.** Each `url` is short-lived (15 minutes) and self-authenticating: fetch it with a plain HTTP GET and no header. Fetch the annotated picture when the comments refer to something you need to see.
- A record whose media the platform has read carries a **`context` stamp** on `get_capture` and `list_captures`: `version` (2), `at`, `fingerprint` (the payload set the findings were computed over), `tokens` (what computing them cost) and the two counts `findings` and `chunks`. It is absent until a router cycle has read the record, and a stamp whose `fingerprint` is not the record's current payload set is stale — the media changed since.

## The findings

The platform's media processor **condenses** a record's media; it does not hand back raw extractions. It splits long media into overlapping chunks — video and audio into chunks of at most thirty minutes with two minutes of overlap (40 minutes becomes two chunks of 21, 90 minutes four of 24), a PDF into page ranges of at most fifty pages with two pages of overlap (120 pages becomes 1–41, 40–80, 79–120), pictures into groups of twenty — reads each chunk, has a second reader check that nothing was missed, and files one document: a `summary` paragraph, a `findings` array, and the `chunks` table the findings cite.

Those limits are the platform's, not this plugin's: they mirror the cloud knobs in `internal/mediactx/knobs.go` (`MaxChunkMinutes`, `OverlapMinutes`, `MaxPDFPages`, `PDFOverlapPages`, `ImageGroupSize`). If a number here ever disagrees with what a document answers, the document is right and this page is stale.

Each finding is `{id, finding, refs, quotes?}`. `finding` is a detailed description of one thing the media establishes: who, what, when in the media, what was shown or said. `refs` say where it was found — `{chunk, start, end}` in seconds for video and audio, `{chunk, pages}` for a PDF, `{chunk, scenes}` for pictures — and a finding that straddles the overlap between two chunks carries one ref in each. `quotes` are verbatim lines when the exact words matter. Ids run `f1`, `f2`, … in chunk order.

`get_record_context` (`id`, `part?`) answers that whole document, or one slice of it: `part` is `summary` (the header, the paragraph and the two counts), `findings` or `chunks`. It is free. Read the summary and the findings before deciding what a recording, an audio file, a PDF or a set of pictures is about, instead of fetching the media bytes yourself. A record with no findings answers a `no_context` error: that is data, not a failure — decide from its content and payloads.

`get_media_chunk` (`id`, `chunk`) answers what the processor **observed** in ONE chunk as it read it: `{id, ref, observations, quotes}`, each observation with its own window and the pass that added it. Free — the observations were stored when the media was read, so reading them costs no model call. Open the chunk a finding cites (`c1`, `c2`, …) when the finding's description is not enough. Router role.

`process_media` (`id`, `force?`, `videoResolution?`) computes a record's findings now through the platform's processor and answers the document. It **costs the workspace's credits**, and it is advertised to the bucket's own router only — the `router` role, and the token that bucket's router names as its brain; any other token is refused with `router_role` before a request is made. Use it only when the record has no findings. A record whose findings are current answers from cache at no cost; `force` recomputes anyway, so do not pass it by habit. A record carrying no media the processor reads is refused with `no_media` — its title, text and comments are its findings and need no processing — and one whose media is still uploading with `payload_incomplete`: try again once the upload settles.

## Reading

`list_items` fetches a bucket newest first, in pages: `slug` (required with a workspace token), `kind` (one or several of `capture`, `note`, `recording`, `dictation`, `file`, `package`), `path` (a directory path, the subtree unless `exact` is true), `since` (RFC3339), `signed` (see below), `limit` (default 50, at most 200), `cursor` (from the previous page's `nextCursor`). `list_directories` names the bucket's directory paths.

## Signing

`sign_items` puts this token's mark on records; `unsign_items` takes it off. Every record you fetch answers `signed: true|false` for your token, and `list_items` with `signed: false` is the queue of what you have not handled. Sign a record when you are done with it, and only then. Another agent's marks are invisible to you; a person sees none.

## Writing

- `add_capture` files a record: `title`, `content`, `kind` (`capture` by default; `note` or `file`), `directoryPath`, `tags`, and a `payloads` list of `{name, mimeType, size, role}`. It answers one upload URL per payload: PUT the bytes there with exactly the `Content-Type` and `Content-Length` it lists, within 15 minutes. A mismatch is refused before anything is stored.
- `move_items` moves records to a `toDirectoryPath`, in the same bucket or another bucket of the workspace (`toBucketSlug`). The rules are the app's: a record in the Trash refuses, a deleted or intake bucket refuses, a move to where the record already is succeeds and changes nothing.
- `trash_items` moves records to the Trash; a person can restore them. Nothing here deletes for good.
- `add_finding` files a short note at the bucket root.

## Routing

A bucket can have one router, and a router of kind `external` names an agent token as its brain: a token minted with the `router` role, which may add and sign and nothing else. Holding that token, you run the bucket's routing cycle yourself, on your own model, with more tools (`/bowerbirds-route <bucket>` runs the whole loop):

- `list_routes` answers the bucket's router and its routes: each route's `destinations` (workspace `paths` and `buckets` a derivation may land in; `external` targets inside a linked connector), its `connectors` with the tools it `enabled` and `disabled` by name, its `description`, `goal`, `rules`, and `trustedExternalCreate`. It says whether your token is the router's brain and which cycle is running. This is the allowlist the gate holds you to.
- `claim_cycle` takes the bucket's cycle marker under a lease (default 10 minutes, at most 60) and answers the `cycleId` your marks carry; it is refused with the holder while another run is on the bucket. `renew_cycle` extends it, `release_cycle` lets it go — always release when done.
- `get_record_context` before deciding: the summary and the findings say what a recording or a document is about, and `get_media_chunk` opens the chunk a finding cites when its description is not enough. `process_media` computes missing findings at the workspace's expense — only for a record that has none; a router's own model is free, the platform's processor is platform credits.
- `search_records` (`q`, `path?`, `limit?`) searches ONE bucket's records — titles, text and a capture's comments — for what already exists there, and answers id, title, path, kind and a snippet of the matching text. `get_record` (`id`) reads one hit whole, its numbered comments included. Both free. Search the destination BEFORE you decide: that step is what turns a create into a merge or an update.
- `derive_record` files a NEW record from a source for a route: `sourceId`, `route`, `kind` (`split` or `augment`), `title`, `content`, `path` or `bucket` inside the route's destinations, `copyPayloads` (default true: the source's bytes are copied onto it), `cycle`. With `mergeInto` (a Package's id) the derived record joins that Package as its next scene; with `related` (record ids) a NEW Package is made of those records and the derived one — and `path` must then be the Package's, or the related records', directory. It carries `derivedFrom`, the route, its depth and origin chain; landing in the same bucket it is signed `routed` at birth. The gate judges it first (destinations, depth 3, five per source per day). The source is never written.
- `sign_items` has a second shape for the router: `items: [{id, status, links: [{route, kind, destination, connectorId, target, tool, bucket, path, findings}]}]` with the `cycle`. `kind` is one of `derive | merge | create | append | link | none`; `findings` names the finding ids that decision rested on (`f1`, `f-text-2`); `status` is `kept` (every route said none), `processed` (a route derived or merged), `routed` (dispatched externally, the record stays live) or `pending` (a route still owes a report), and it must agree with the links. Every link is judged by the gate against the route's allowlist, grant and trust; the answer says per record what was signed and what was refused. A record signed with a final status leaves the router's queue until it changes, and carries your plain signature too.

### How a route decides

Per record, per route: understand the findings that route should see, **search the destination first**, and only then decide exactly one of —

- **merge** — the destination is internal and a related record already exists: a derived record carrying the findings, placed INTO the Package that holds the related record (`derive_record` with `mergeInto`), or a new Package made of the related records and the derived one (`derive_record` with `related`).
- **update** — the destination is external and a related item already exists: a comment or an append on that item, through a tool the route enables. On an internal destination an update is proposed as a merge beside the related record: routers add, they never edit a source.
- **create** — nothing related exists: a new item at an external target, or a new derived record, carrying links to the related items when the destination supports them.
- **nothing** — the findings add no value at this destination. Say why in one line; that is a decision, not a failure.

A finding may fit several routes, and each route sees only the findings that fit it. A finding that fits no route stays where it is and is reported as unrouted, never forced.

Routers only add: as a router you never move, edit or trash a source — a token with the router role is refused at `move_items` and `trash_items`. External items are created or appended through the tools the route enables on its linked connector, and never through a tool the route lists as disabled. A refusal from the gate is a rule, not a retry: report its sentence.

## What you cannot do

Read the Trash, read another workspace, mint credentials, or reach a bucket that takes submissions through a publishable web key only. With the router role: move or trash anything. Every refusal names its reason; report it rather than retrying.
