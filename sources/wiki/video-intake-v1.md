---
type: "documentation"
title: "Video intake v1: videos taken in from their link (Track W)"
description: "How a video becomes a note: its public metadata and its own transcript, with the time each passage was said. Plan: docs/knowledge_layer_improvement_plan_v1.md, Track W. Code: second_brain/video/ (links.py, youtube.py, captions.py..."
status: "stable"
reading_status: "read"
product: "Kovium"
product_version: "0.1.0"
doc_kind: "contract"
audience: "people and agents using or changing Kovium"
doc_path: "docs/video_intake_v1.md"
doc_sha256: "32488c842413b0afc7dd8c7760eda253d8953ae3b4ff2519b5ec742fc59142fc"
---

# Video intake v1: videos taken in from their link (Track W)

How a video becomes a note: its public metadata and its own transcript, with
the time each passage was said. Plan: `docs/knowledge_layer_improvement_plan_v1.md`,
Track W. Code: `second_brain/video/` (`links.py`, `youtube.py`, `captions.py`,
`intake.py`, `batch.py`).

## Why

Talks, lectures, tutorials and interviews are sources for any collection. A
video link given to `/sources/process` used to be fetched as a web page, a
script shell with almost no text, so the note described nothing; one instance
had its transcripts collected by hand outside the platform.

## One video: `POST /sources/process` with a video link

A YouTube watch, short-link (`youtu.be`), Shorts, embed or live link is a
video; anything else is fetched as a page, as before. The note's kind is
`youtube-video` unless the request names another.

What is read, and only this:

- the video's **public watch page** (title, channel, publication date,
  length, description);
- the site's **player endpoint**, asked with the video's id, for the list of
  caption tracks (the caption links on the page itself answer empty to a plain
  request);
- **one caption track**: a language from `languages`, else the video's own
  spoken language (the language of its automatic track); within a language,
  captions written for the video before automatic ones (`prefer_manual`).

Never the video file, never a signed-in session, never a model: the
transcript is the site's own captions, so taking a video in costs no tokens
before extraction. Every request goes through `conversion/web_fetch.py` (the
one module that reaches the web), with its address checks, no cookies, no
identity and a size cap; requests are spaced by `min_seconds_between_requests`.
The player query is that module's one POST, allowed only to the hosts named
in `provider.hosts`.

**The raw source** is Markdown saved under `sources/raw/youtube/<title>-<id>.md`:
the video's facts (channel, publication date, duration, URL, language, where
the transcript came from), its description, and its transcript in passages of
about `paragraph_seconds`, each beginning with its time (`[1:15] ...`), under the
video's chapters when its description declares them (lines starting with a
time, the first at 0:00, at least three, in order: the site's own rule). The
text is scanned for instructions addressed to an AI reader like any fetched
page, and quarantined if it holds them.

**The note** carries `url`, `video_id`, `channel`, `publication_date`,
`duration`, `transcript_source` (`manual`, `automatic`, `description_only`) and
`transcript_quality` in its frontmatter, and `## Timestamped anchors` lists each
chapter with a link to its moment (`...watch?v=<id>&t=70s`). The passages keep
their times, so the extraction can cite when something was said.

The result adds `video`: `video_id`, `title`, `channel`, `published`,
`duration`, `transcript_source`, `language`, `transcript_chars`, `truncated`,
`chapters`.

**Refusals** (422 `{reason, message}`, nothing copied):

| reason | when |
|---|---|
| `no_transcript` | no caption track could be read; with `description_only: true` (the request's, else `config/video.json`'s) it is taken in from its description and chapters and the note says so |
| `video_not_public` | private, members-only or otherwise not playable without signing in, with the site's own reason |
| `video_page_unreachable`, `video_player_unreachable` | the site could not be reached or did not list the tracks; if it persists, the `provider` settings may need updating |

## Many videos: `POST /sources/process_videos`

`{urls: [...], kind?, template_id?, dry_run: true, token_budget?,
description_only?, auto_ingest, concepts, wait}`. Each link is a video or a
playlist (`...playlist?list=<id>`, read from its public page: its first page
of videos). A **dry run** (the default) reads each video's metadata and caption
track, with no model call, and returns `plan.videos`, one row per video in
order:

- `video_id`, `url`, `title`, `channel`, `duration`, `language`,
  `transcript_source`, `transcript_chars`, `estimated_tokens`;
- `action`: `process`, `duplicate` (the same id earlier in the run, or the
  same transcript as an earlier video: a re-upload), `already_ingested` (this
  workspace already holds its transcript), `no_transcript`,
  `video_not_public`, ... with `reason`.

`plan.estimated_tokens` sums the `process` rows, by this install's measured
cost per document and per character when it has one (`estimate_basis`), with
each video's concept upkeep. `refused_links` lists links that name no video or
playlist. With `token_budget`, `would_fit_budget`. At most
`max_videos_per_run` videos are planned (`over_limit` says how many more).

A **real run** (`dry_run: false`, audited) processes the `process` rows in
order through `/sources/process`, reusing what the plan read rather than
asking the site again, stops cleanly before a video whose estimate would pass
the budget (`run.stopped_at_budget`), and makes one concept pass after the
last (`run.concept_pass`). Backgrounded unless `wait`.

## Configuration

`config/video.json`, a workspace's own over it, key by key: `transcripts`
(`captions` or `none`), `languages`, `prefer_manual`, `description_only`,
`paragraph_seconds`, `min_seconds_between_requests`, `max_videos_per_run`,
`max_transcript_chars`, and `provider` (`player_url`, `client_name`,
`client_version`, `hosts`): the site's endpoint and the client it names,
which the site changes; an install follows here without a code change.

## Measured

On 2026-10-15, live, a public 18-minute lecture: the watch page, the player
query and its English manual caption track (18,431 characters), 11 chapters
from its description, a note whose `## Timestamped anchors` linked each
chapter's moment. The caption links on the watch page itself answered empty;
the player endpoint's answered with the track, with the platform's own
user agent. A public playlist's page listed its videos as `lockupViewModel`
entries (an older layout used `playlistVideoRenderer`; both are read).

## Limitations

- **The site's undocumented endpoints.** The player endpoint, its client
  name and the playlist page's layout are the site's, not a published API;
  they change. When they do, videos are refused with the reason and the
  `provider` settings (or a code change, for a page layout) restore it.
- A playlist's videos past its first page are not read.
- No local speech-to-text: a video without captions has only its
  description. A local provider can be added behind the same choice, off by
  default and loopback only, as the plan allows.
- Automatic captions mishear names, numbers and terms; the note says when a
  transcript is automatic, and nothing re-checks it against the audio.
- Only YouTube links are recognised.
