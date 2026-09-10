---
name: bowerbirds-records
description: How to read, file, move, trash and sign records in a Bowerbirds workspace through the bowerbirds MCP server — captures with numbered comments, payload URLs that need no credential, and the signature that marks what this agent already handled. Use whenever a task mentions Bowerbirds, a bucket, a capture, or a screenshot someone filed for you.
---

# Bowerbirds records

Bowerbirds is where people file annotated screenshots (captures), notes, dictations and screen recordings for their agents. You reach a workspace through the `bowerbirds` MCP server; the first call opens an approval page in the person's browser, where they pick the workspace and the role you get. You act as your own token, never as the person.

## The records

- A **capture** is a picture with numbered pins. Its `content` is the list of comments, one per pin, in the same numbering. `get_capture` also answers `comments` parsed out of the content.
- A record's payloads are listed in `get_capture` as `{name, role, mimeType, size, url}`. For a capture: `annotated.png` (role `annotated`, the picture with the pins drawn on it), `raw.png` (role `original`), `session.json` (role `capture-session`, the editable annotations). Recordings carry video slices and, when processed, a transcript.
- **Payload bytes never come through MCP.** Each `url` is short-lived (15 minutes) and self-authenticating: fetch it with a plain HTTP GET and no header. Fetch the annotated picture when the comments refer to something you need to see.

## Reading

`list_items` fetches a bucket newest first, in pages: `slug` (required with a workspace token), `kind` (one or several of `capture`, `note`, `recording`, `dictation`, `file`, `package`), `path` (a directory path, the subtree unless `exact` is true), `since` (RFC3339), `signed` (see below), `limit` (default 50, at most 200), `cursor` (from the previous page's `nextCursor`). `list_directories` names the bucket's directory paths.

## Signing

`sign_items` puts this token's mark on records; `unsign_items` takes it off. Every record you fetch answers `signed: true|false` for your token, and `list_items` with `signed: false` is the queue of what you have not handled. Sign a record when you are done with it, and only then. Another agent's marks are invisible to you; a person sees none.

## Writing

- `add_capture` files a record: `title`, `content`, `kind` (`capture` by default; `note` or `file`), `directoryPath`, `tags`, and a `payloads` list of `{name, mimeType, size, role}`. It answers one upload URL per payload: PUT the bytes there with exactly the `Content-Type` and `Content-Length` it lists, within 15 minutes. A mismatch is refused before anything is stored.
- `move_items` moves records to a `toDirectoryPath`, in the same bucket or another bucket of the workspace (`toBucketSlug`). The rules are the app's: a record in the Trash refuses, a deleted or intake bucket refuses, a move to where the record already is succeeds and changes nothing.
- `trash_items` moves records to the Trash; a person can restore them. Nothing here deletes for good.
- `add_finding` files a short note at the bucket root.

## What you cannot do

Read the Trash, read another workspace, mint credentials, or reach a bucket that takes submissions through a publishable web key only. Every refusal names its reason; report it rather than retrying.
