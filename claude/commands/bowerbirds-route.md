---
description: Route a Bowerbirds bucket as its external router — claim the cycle, read the routes, read each unsigned record's findings, search the destination, dispatch it (merge, derive, create, append, link or none), sign it with the links, release the cycle
argument-hint: <bucket-slug>
---
# /bowerbirds-route

You are the external router of one Bowerbirds bucket: the bucket's router document names your token as its brain, and you run the same loop a native router runs, on your own model, at zero platform credits. The bucket slug is the first argument; pass it as `slug` on every `bowerbirds` tool call.

Routers only add. In this whole run you never move, edit or trash a record, and you never use `move_items` or `trash_items`. What you may write is exactly two things: a derived record through `derive_record`, and an item at an external destination through a tool the route enables on its linked connector. Everything else is reading and signing.

## 1. Claim the cycle

Call `claim_cycle` with the slug (and `ttlSeconds` if you expect more than ten minutes). Keep the `cycleId` it answers and pass it as `cycle` to every `derive_record` and `sign_items` call. If it answers `cycle_held`, another run — yours or the fleet's — holds the bucket: stop and report the holder; do not work the bucket. If it says this token is not the router's brain, or the bucket's router is not external, stop and report the sentence.

Call `renew_cycle` with the cycle before `expiresAt` when a run is long.

## 2. Read the routes

Call `list_routes`. Each route carries its `destinations` (workspace `paths` and `buckets` a derivation may land in; `external` targets inside a linked connector, with `connectorId` and `target`), its `connectors` with the `tools.enabled` and `tools.disabled` lists, its `description`, `goal`, `rules` and `trustedExternalCreate`. This is the allowlist: a route can only derive into its paths and buckets, and only reach its external targets with the tools it enables. A tool a route lists under `disabled`, or one it never enabled, is not yours to call for that route, even when the connector offers it.

## 3. Fetch what is unsigned, and read its findings

Call `list_items` with the slug and `signed: false`, paging with `cursor` until `nextCursor` is absent. These are the records the router has not signed yet. For each one call `get_capture` to read its content and payload manifest; fetch the annotated image with a plain HTTP GET when the comments point at something you need to see.

When a record carries a `context` stamp — the platform's media processor has read its recording, audio, PDF or pictures — call `get_record_context` with its id and read the **summary and the findings with their chunk references**: each finding is a detailed description of one thing the media establishes, with the chunk and the moment, pages or scenes it was found at, and verbatim quotes when the exact words matter. Ask for a `part` (`summary`, `findings` or `chunks`) when you want one slice, and call `get_media_chunk` with the chunk id a finding cites (`c1`, `c2`, …) when the description is not enough — it answers what the processor observed in that chunk, and it costs no model call. Never fetch the media bytes to re-read them yourself.

A record with no findings answers `no_context`: decide it from its content and payloads — its own words are its findings. `process_media` computes findings through the platform's processor, and it **costs the workspace's credits** — use it only when the record has no findings and its media is what the decision turns on, never with `force` by habit.

## 4. Decide, per record, per route

For every record, ask of every route whether its description and rules fit the record's findings — read the findings themselves, not the record's title or kind. A finding may fit several routes (a meeting that raised a bug and a decision reaches both), a route may receive several findings of one record, and a finding that fits no route stays where it is: report it as unrouted, never force it. Answer per record per route with exactly one dispatch kind:

- `merge` — the destination is internal and a related record already exists: a derived record carrying the findings, placed INTO the Package that holds the related record, or a new Package made of the related records and the derived one.
- `derive` — a new record in the workspace with nothing related beside it: a split (one of several pieces) or an augmented copy, filed at a path or bucket in the route's destinations.
- `create` — a new item at one of the route's external targets, through an enabled tool.
- `append` — a comment or attachment on an existing external item, through an enabled tool. This is the external form of an update; on an internal destination an update is a `merge` beside the related record, because routers add and never edit a source.
- `link` — the record refers to an existing external item and nothing needs writing.
- `none` — the route has nothing to do with this record. Say why in one line; that is a decision.

Each decision carries a reason, a confidence, and the finding ids it rests on. Prefer the smallest kind that does the job: none < link < append < merge < derive < create.

## 5. Look before you act

Search the destination BEFORE you decide, never after. An internal destination: `search_records` with `q` and the route's destination `path` answers the bucket's matching records (id, title, path, kind, snippet), and `get_record` reads a hit whole, its numbered comments included. An external destination: the route's enabled read tools (`search_issues`, `get_issue`, and the like on the linked connector). Something related already there turns a `create` into a `merge`, an `append` or a `link`. The links other runs recorded are the dedupe memory too: a record that is already linked to an item never earns a second one.

## 6. Act, only through the two doors

- `derive_record` with `sourceId`, `route`, `kind` (`split` or `augment`), `title`, `content`, `path` or `bucket` inside the route's destinations, and the `cycle`. For a **merge**, add `mergeInto` (the id of the existing Package the derived record joins as its next scene) or `related` (the record ids a NEW Package is made of, with the derived one) — and `path` must then be the Package's, or the related records', own directory, the one the search hit named, or the merge is refused. The gate judges it first (the route's destinations, depth 3, five derivations per source per day); the source is never written, and a record is never merged into itself. Keep the derived record's `id` and the Package id from the answer.
- The route's enabled external tools, on the connector the route links, for `create` and `append`. Keep the item id the tool answers (an issue identifier, a page id).

If a route's `trustedExternalCreate` is false, the gate will not accept a `create` link from it yet: do not create the item. Record the record as needing a person, and say so in the report; the first accepted create in the apps earns the route unattended creates.

## 7. Sign

When a record's routes have all reported, call `sign_items` with `items` (not `ids`) and the `cycle`:

```
{"slug": "<bucket>", "cycle": "<cycleId>", "items": [{
  "id": "<record>", "status": "routed",
  "links": [
    {"route": "<route id>", "kind": "create", "destination": "ENG-123", "connectorId": "<connector id>", "target": "<target as list_routes names it>", "tool": "create_issue", "findings": ["f1", "f3"]},
    {"route": "<other route id>", "kind": "none"}
  ]}]}
```

One link per route per record, kind as decided: a `derive` names the derived record's id as `destination` (and `bucket`, `path` where it landed); a `merge` names the Package the derived record joined or made; `create` and `append` name the item, `connectorId`, `target` and the `tool` used; `link` names the item, `connectorId` and `target`; `none` names nothing. Every link may carry `findings`: the finding ids that decision rested on (`f1`, `f-text-2`), so a person re-reading the record sees what it was decided from — a `none` rests on findings too. The status must agree with the links: `kept` when every route said none, `processed` when a route derived or merged, `routed` when the record was dispatched externally and stays the live record, `pending` only when a route still owes a report. The gate judges every link against the route's allowlist, grant and trust before a mark is written, and the answer says per record whether it was signed, what was refused and why.

Prefer leaving a record unsigned over signing it `pending`: an unsigned record comes back in the next run, and a signed one does not.

## 8. Stop on the gate's clamps

A refusal from the gate — in `derive_record`'s error or in a `sign_items` item's `refused` or `error` — is a rule, not a retry: a destination outside the route's allowlist (`destination not allowed`), a Package that has since moved, a tool the route did not enable, a derivation past depth 3 or over the day's cap, a create from a route not yet trusted, a status the marks would not say. Do not rephrase the dispatch to get past it. Leave that record as it is, note the sentence, and go on to the next record.

## 9. Release and report

Always call `release_cycle` with the cycle when you are done, including after a stop. Then report: the records signed and their statuses with the links (issue ids, derived record ids, Package ids), the findings that reached no route, the records left unsigned and why, and every clamp the gate answered, verbatim.
