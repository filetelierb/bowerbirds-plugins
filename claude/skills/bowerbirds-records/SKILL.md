---
name: bowerbirds-records
description: How to read, file, move, trash, sign and route records in a Bowerbirds organization through the bowerbirds MCP server — captures with numbered comments, payload URLs that need no credential, the signature that marks what this agent already handled, the media context (get_record_context) the platform filed for a recording, audio, PDF or set of pictures, and the router tools (list_routes, claim_cycle, derive_record, sign_items with links, process_media) an external router uses. Use whenever a task mentions Bowerbirds, a bucket, a capture, a screenshot someone filed for you, or routing a bucket.
---

# Bowerbirds records

Bowerbirds is where people file annotated screenshots (captures), notes, dictations and screen recordings for their agents. You reach an organization through the `bowerbirds` MCP server; the first call opens an approval page in the person's browser, where they pick the organization and the role you get. You act as your own token, never as the person.

## The records

- A **capture** is a picture with numbered pins. Its `content` is the list of comments, one per pin, in the same numbering. `get_capture` also answers `comments` parsed out of the content.
- A record's payloads are listed in `get_capture` as `{name, role, mimeType, size, url}`. For a capture: `annotated.png` (role `annotated`, the picture with the pins drawn on it), `raw.png` (role `original`), `session.json` (role `capture-session`, the editable annotations). Recordings carry video slices and, when processed, a transcript.
- **Payload bytes never come through MCP.** Each `url` is short-lived (15 minutes) and self-authenticating: fetch it with a plain HTTP GET and no header. Fetch the annotated picture when the comments refer to something you need to see.
- A record whose media the platform has read carries a **`context` stamp** on `get_capture` and `list_captures`: a `digest` (one short paragraph), `signals` (`bug`, `feature`, `question`, `task`, `decision`, `other`, each `high | medium | low | none`), `entities`, `screens`, and `partial` with a `note` when something was left unread. It is absent until a router cycle has read the record.

## The media context

`get_record_context` (`id`, `part?`) reads the whole document the platform's media processor filed for a record — the digest, the signals, the entities and screens, the quotes, and the segments with timed moments — or one slice of it: `part` is `summary`, `segments`, `transcript` (the quotes plus each segment's summary; no verbatim transcript is stored) or `moments` (every timed moment with its segment). It is free. Read the digest and the signals before deciding what a recording, an audio file, a PDF or a set of pictures is about; ask for a part when the digest is not enough, instead of fetching the media bytes yourself. A record with no context answers a `no_context` error: that is data, not a failure — decide from its content and payloads.

`process_media` (`id`, `force?`, `videoResolution?`, `maxMinutesPerRecord?`) computes a record's context now through the platform's processor and answers the document. It **costs the organization's credits** and is advertised to the `router` role only; any other token is refused with `router_role` before a request is made. Use it only when the record has no context. A record whose context is current answers from cache at no cost; `force` recomputes anyway, so do not pass it by habit.

## Reading

`list_items` fetches a bucket newest first, in pages: `slug` (required with an organization token), `kind` (one or several of `capture`, `note`, `recording`, `dictation`, `file`, `package`), `path` (a directory path, the subtree unless `exact` is true), `since` (RFC3339), `signed` (see below), `limit` (default 50, at most 200), `cursor` (from the previous page's `nextCursor`). `list_directories` names the bucket's directory paths.

## Signing

`sign_items` puts this token's mark on records; `unsign_items` takes it off. Every record you fetch answers `signed: true|false` for your token, and `list_items` with `signed: false` is the queue of what you have not handled. Sign a record when you are done with it, and only then. Another agent's marks are invisible to you; a person sees none.

## Writing

- `add_capture` files a record: `title`, `content`, `kind` (`capture` by default; `note` or `file`), `directoryPath`, `tags`, and a `payloads` list of `{name, mimeType, size, role}`. It answers one upload URL per payload: PUT the bytes there with exactly the `Content-Type` and `Content-Length` it lists, within 15 minutes. A mismatch is refused before anything is stored.
- `move_items` moves records to a `toDirectoryPath`, in the same bucket or another bucket of the organization (`toBucketSlug`). The rules are the app's: a record in the Trash refuses, a deleted or intake bucket refuses, a move to where the record already is succeeds and changes nothing.
- `trash_items` moves records to the Trash; a person can restore them. Nothing here deletes for good.
- `add_finding` files a short note at the bucket root.

## Routing

A bucket can have one router, and a router of kind `external` names an agent token as its brain: a token minted with the `router` role, which may add and sign and nothing else. Holding that token, you run the bucket's routing cycle yourself, on your own model, with five more tools (`/bowerbirds-route <bucket>` runs the whole loop):

- `list_routes` answers the bucket's router and its routes: each route's `destinations` (organization `paths` and `buckets` a derivation may land in; `external` targets inside a linked connector), its `connectors` with the tools it `enabled` and `disabled` by name, its `description`, `goal`, `rules`, and `trustedExternalCreate`. It says whether your token is the router's brain and which cycle is running. This is the allowlist the gate holds you to.
- `claim_cycle` takes the bucket's cycle marker under a lease (default 10 minutes, at most 60) and answers the `cycleId` your marks carry; it is refused with the holder while another run is on the bucket. `renew_cycle` extends it, `release_cycle` lets it go — always release when done.
- `derive_record` files a NEW record from a source for a route: `sourceId`, `route`, `kind` (`split` or `augment`), `title`, `content`, `path` or `bucket` inside the route's destinations, `copyPayloads` (default true: the source's bytes are copied onto it), `cycle`. It carries `derivedFrom`, the route, its depth and origin chain; landing in the same bucket it is signed `routed` at birth. The gate judges it first (destinations, depth 3, five per source per day). The source is never written.
- `get_record_context` before deciding: the digest and signals say what a recording or a document is about, and a `part` says more. `process_media` computes a missing context at the organization's expense — only for a record that has none; a router's own model is free, the platform's processor is platform credits.
- `sign_items` has a second shape for the router: `items: [{id, status, links: [{route, kind, destination, connectorId, target, tool, bucket, path}]}]` with the `cycle`. `kind` is one of `derive | create | append | link | none`; `status` is `kept` (every route said none), `processed` (a route derived), `routed` (dispatched externally, the record stays live) or `pending` (a route still owes a report), and it must agree with the links. Every link is judged by the gate against the route's allowlist, grant and trust; the answer says per record what was signed and what was refused. A record signed with a final status leaves the router's queue until it changes, and carries your plain signature too.

Routers only add: as a router you never move, edit or trash a source — a token with the router role is refused at `move_items` and `trash_items`. External items are created or appended through the tools the route enables on its linked connector, and never through a tool the route lists as disabled. A refusal from the gate is a rule, not a retry: report its sentence.

## What you cannot do

Read the Trash, read another organization, mint credentials, or reach a bucket that takes submissions through a publishable web key only. With the router role: move or trash anything. Every refusal names its reason; report it rather than retrying.
