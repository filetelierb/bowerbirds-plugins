---
description: Sweep the records this agent has not signed yet in a Bowerbirds bucket, act on each, and sign it
argument-hint: <bucket-slug> [--kind capture,note] [--path dir/sub]
---
# /bowerbirds-triage

Work through everything in the named Bowerbirds bucket that this agent has not signed yet.

1. Call `list_items` with `slug` set to the bucket from the arguments, `signed: false`, and any `kind` or `path` filter given. Page with `cursor` until `nextCursor` is absent.
2. For each record, call `get_capture` and read its content: a capture's content is a numbered list of comments about the picture, and the payload manifest carries fetchable URLs for the annotated screenshot (`annotated.png`), the original (`raw.png`) and the editable annotations (`session.json`). Fetch the annotated image with a plain HTTP GET when you need to see what the comments point at.
3. Do what the record asks for in the repository you are working in. Keep a short note per record of what you did or why you did nothing.
4. When a record is handled, call `sign_items` with its id so it never comes back in the next sweep. Do not sign what you skipped.
5. Finish with a summary: how many records you signed, and the ones you left unsigned with the reason.
