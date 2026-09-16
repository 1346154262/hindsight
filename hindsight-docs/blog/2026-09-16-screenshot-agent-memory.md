---
title: "What It Takes to Put a Screenshot in an Agent's Memory"
authors: [benfrank241]
slug: "2026/09/16/screenshot-agent-memory"
date: 2026-09-16T16:00
tags: [hindsight, agent-memory, multimodal, retain, provenance, vision, release]
description: "Storing an image is easy. Making a fact cite the screenshot it was read from, without inventing evidence, is the hard part. Here is how attachments work in Hindsight 0.10.0, including where they refuse."
image: /img/blog/screenshot-agent-memory.png
hide_table_of_contents: true
---

![A fact recalled from agent memory, shown with the screenshot it was extracted from](/img/blog/screenshot-agent-memory.png)

A support thread comes in. The customer writes two paragraphs, pastes a screenshot of what the CLI printed, and adds a sentence underneath. The request ID, the bank ID and the error code exist in exactly one place in that thread: inside the image.

Your agent reads the prose and remembers the prose. The screenshot becomes, at best, a file in a bucket somewhere.

<!-- truncate -->

## TL;DR

- Hindsight 0.10.0 lets `content` be an ordered list of text, image and file blocks, so an attachment is read in the position it occupies in the surrounding text.
- A recalled fact comes back with the attachments it was extracted from, and that edge is per fact, not per chunk.
- Attribution isn't inferred from position. The extraction model is asked which attachments a fact came from, and answers that don't resolve are dropped rather than rounded to the nearest one.
- Blocks are flattened into one canonical string at the API boundary, which is why idempotency, `update_mode=append`, chunk re-extraction and export keep working unchanged.
- If the configured model can't read images, the retain fails before a single byte is written.
- It's in v0.10.0 today for self-hosted deployments. Hindsight Cloud is still on 0.9.2 at the time of writing.

## We already had file upload, and that isn't this

Back in March, Hindsight Cloud [added document file upload](https://hindsight.vectorize.io/blog/2026/03/09/hindsight-document-upload): drop in a PDF, a spreadsheet or a PNG, run text extraction over it, retain the result. That's genuinely useful and it isn't going anywhere.

It also has a shape. The unit is the file. You hand over a document, it becomes text, the text becomes memories. A screenshot pasted into the middle of a conversation is not a document, and turning it into one loses the thing that made it worth keeping: it was attached *there*, after that sentence, as evidence for that specific claim.

Two things are new in 0.10.0, and both are about position rather than storage.

**An attachment is read where it sits.** A chart between two paragraphs is read as the chart between those two paragraphs, not as an appendix.

**Provenance is per fact.** When recall returns "the error was a dimension mismatch on a PNG declared as image/png," it can also return the screenshot that sentence was read out of. Not the whole document. Not every image in the chunk. The one the extractor said it needed.

## One string, or everything breaks

The obvious implementation is to thread a new content shape through the whole pipeline. Hindsight does the opposite: blocks are flattened at the API boundary into a single canonical body, and each attachment becomes an atomic placeholder token sitting on its own paragraph.

The reason is that a lot of existing behaviour is built on content being one string. Idempotency by content hash. `update_mode=append`. Chunk-delta re-extraction, which only reprocesses the chunks that actually changed. `reprocess_document`. Export and import. All of that assumes a document has a body it can hash and diff.

So the flattening is designed to be exactly equivalent for text. A single text block canonicalizes to precisely its own text, which means this:

```json
{"content": "The importer rounds half-up."}
```

and this:

```json
{"content": [{"type": "text", "text": "The importer rounds half-up."}]}
```

produce an identical stored body and an identical content hash. A plain string behaves byte for byte as it did before, and a text-only chunk produces a byte-identical request to the model.

The placeholders are expanded back into real multimodal parts at one point only: prompt assembly, with each image sitting exactly where its placeholder stood. The bytes themselves never travel in the request payload. They're written to file storage first, content-addressed by SHA-256, so a retry of a queued operation doesn't drag a base64 blob along with it.

There's a small security consequence worth noting. Because the placeholder is just a token in text, a caller could try writing one by hand to cite an attachment their document never had. Text arriving from callers is scrubbed of anything shaped like a placeholder, while a document's own existing attachment ids are exempted, so re-sending a document's original text on an edit doesn't silently delete its screenshots.

## Asking, rather than guessing

Here's the part I find most interesting, because the obvious approaches are all wrong.

You could attribute by proximity: a fact gets the nearest image. You could attribute by chunk: every fact in a chunk gets every attachment in that chunk. Both produce an answer for every fact, and both are frequently lying.

Instead the extraction prompt numbers the attachments in the order they appear and asks the model to report, for each fact, which numbers it needed. The instruction is narrow on purpose: list an attachment only when the fact could not be stated without looking at it. Each extracted fact carries a list of integers back, and those integers are resolved against that chunk's placeholder order.

What happens to a number that doesn't resolve is the design decision that matters. It's dropped. A fact that claims attachment 3 in a chunk holding two attachments ends up citing nothing at all, and the code comment says why: attaching the nearest one would be inventing provenance. Zero and negative numbers go the same way. A text-only chunk resolves to nothing even if the model insists it used an image.

That leaves a system that will happily return a fact with no evidence attached, which is the correct failure. "I can't show you where this came from" is a usable answer. "Here's a screenshot that doesn't support this claim" is not.

The test suite pins both directions of that. There's a case with a fact stated in prose next to an entirely unrelated image, asserting nothing gets attributed. And there's a test using a real model against a diagram whose facts appear nowhere in the prose, which asserts a partition: at least one fact must cite the image, or the evidence is invisible, and not every fact may cite it, or it's being cited for prose it doesn't support.

Worth being precise here, because it's a model judgement rather than a guarantee. The resolution around it is deterministic and range-checked, and the extraction is not.

## An image costs about 22 characters

A detail that sounds trivial and isn't.

Chunking has a character budget. If you count an image against that budget the way you'd count its base64 payload, or even a generous estimate of its token cost, then a long article with three screenshots gets split in a way that separates the screenshots from the sentences that reference them. An earlier iteration did exactly this, and the result was articles split away from their own images.

In 0.10.0, the character budget counts text, and an attachment costs only the length of its placeholder token, roughly 22 characters. The real limit on how many attachments land in one chunk is a separate, bank-configurable cap matched to what the provider will accept in a single request, defaulting to 8.

The effect is that adjacency survives. One sentence followed by ten images stays in a single chunk, because splitting it would destroy the only thing that made those images interpretable. Going over the per-chunk cap doesn't reject anything either. It just produces more chunks.

## Two kinds of "attachments" in one recall response

A recall response can carry attachment data in two places, and they deliberately don't hold the same thing.

| | `chunks{}.attachments` | `results[].attachments` |
|---|---|---|
| **Scope** | everything that chunk's text references | only what the extractor attributed to that fact |
| **Granularity** | per chunk | per fact |
| **Answers** | "what was in the source material here?" | "what is this specific claim based on?" |
| **When empty** | chunk has no attachments | fact was stated in the text |

Both are useful. If you're rendering the source document, you want the chunk set. If you're showing a user why the agent believes something, you want the fact set, and showing the chunk set there would quietly overstate your evidence.

One convention to know when you're parsing: when there's nothing to report, the field is omitted rather than returned as an empty list.

## Where it refuses

Most of the interesting engineering in this feature is in the failure paths.

**No vision model, no retain.** If the configured model can't read images, an image-bearing retain fails at ingress, before anything is written. The alternative would be extracting from the prose and quietly skipping the images, which leaves a document that looks retained while the information the caller cared about is simply gone. A hard error is recoverable. A silent omission isn't, because nothing in the system will ever tell you it happened.

**"Probably fine" counts as no.** Capability is three-valued: yes, no, and can't tell. Can't tell refuses. This one will catch self-hosters, because an OpenAI-compatible endpoint that isn't OpenAI reports can't tell, which covers a lot of popular local and gateway setups. The override exists, and it's an operator decision rather than a default: set the vision flag explicitly and the server takes your word for it.

**Vision runs in its own slot.** You can configure a separate vision model rather than upgrading your whole retain path, and only chunks that actually carry attachments go to it. That slot deliberately can't fall back to a text-only chain.

**Batch mode is incompatible.** With the batch retain API enabled, an attachment-bearing retain fails with an explicit error, because that path builds provider request bodies directly and never sees the interleaved parts.

There's exactly one place the system degrades instead of refusing: if an attachment's bytes have gone missing from storage, the prompt gets a literal "attachment unavailable" marker rather than failing the whole retain. The placeholder stays in the text, and the read surfaces skip what they can't resolve.

## What it costs to keep

Attachments are content-addressed by SHA-256, and the storage key is derived purely from the hash. Retain the same screenshot in forty documents and you get one blob and one row. The short id used in placeholders is a 48-bit prefix of that hash, and collisions aren't silent: a uniqueness constraint makes the second insert fail loudly rather than quietly attach the wrong image.

A nice consequence of hashing the bytes is that the filename can't live with them, because it isn't a property of the bytes. The same PDF can be `policy-v1.pdf` in one document and `escalation-runbook.pdf` in another. The name belongs to the reference, not the file.

Cleanup runs off references rather than a scheduled sweep. The document-to-attachment edges are derived from the canonical text on every write, so re-ingesting a document without an image drops that edge. When the last reference goes, the row is deleted and the blob removal is best effort, on the principle that the row is the authority and a stranded blob is wasted bytes rather than a correctness bug.

On upgrading: the migration adds two new tables, which are empty, and one array column to the memory tables, which may not be small in a mature deployment.

## Limits worth knowing before you try it

| Limit | Value | Notes |
|---|---|---|
| Source type | base64 inline only | no URL fetch, no pre-uploaded handle yet |
| Max size | 20 MB per attachment | decoded, server-level setting |
| Max count | 50 per retain item | split across items to go higher |
| Per chunk | 8 by default | bank-configurable, not a rejection |
| Media types | any well-formed type | no allowlist; the provider's own error surfaces if it can't read it |
| Per-fact provenance | `facts` extraction mode | in `chunks` and `verbatim` modes the fact is the chunk, so it takes everything |
| MCP | not yet | attachments are returned over HTTP, not through the MCP tools |
| Hindsight Cloud | 0.9.2 at time of writing | self-host v0.10.0 to use this today |

The extraction-mode caveat is the one most likely to surprise you. Per-fact attribution is a property of fact extraction. If you've configured chunk or verbatim mode, the fact is the chunk, so it carries every attachment the chunk holds, and no model is consulted.

## FAQ

**Does this mean my agent can "see" screenshots now?**
During retain, yes, if you've configured a vision-capable model. The image is read in position during extraction, and the facts it produces are stored as text alongside a reference to the image. Recall returns those facts and can hand back the attachment behind them.

**Do I have to change existing code?**
No. `content` still accepts a plain string, and that path is unchanged down to the content hash. Blocks are opt-in per item.

**What happens if I send an image to a model that can't read it?**
The retain fails with a clear error rather than silently dropping the image. If your provider is OpenAI-compatible but not OpenAI, it will report an unknown capability and be refused until you set the vision flag explicitly.

**Can two documents share the same image without storing it twice?**
Yes. Attachments are deduplicated by SHA-256 across the bank, and the filename is stored on each document's reference rather than on the bytes.

**Is the fact-to-image link guaranteed to be correct?**
No, and it's worth being honest about that. The extraction model decides which attachments a fact required. What's guaranteed is that an answer which doesn't resolve to a real attachment in that chunk is discarded rather than approximated, so a wrong citation is less likely than no citation.

**Can I get attachments through MCP?**
Not in 0.10.0. The attachment fields are on the HTTP responses; the MCP recall tool returns facts without them.

**How expensive is the upgrade migration?**
Two new tables that start empty, plus an array column added to the memory unit tables. Plan it like any column addition on a large table rather than assuming it's free.

## Learn more

- [Inside retain(): What Actually Happens When Your Agent Remembers](https://hindsight.vectorize.io/blog/2026/07/13/inside-retain-agent-memory) is the write path this feature extends
- [What's new in Hindsight 0.10.0](https://hindsight.vectorize.io/blog/2026/09/14/version-0-10-0) covers the rest of the release, including the faster request path and the prompt preview endpoint
- [Cross-Encoder Reranking](https://hindsight.vectorize.io/blog/2026/08/28/cross-encoder-reranking-agent-memory) explains what happens to a recalled fact after it's retrieved
- [Stop Growing Your Always-On Context](https://hindsight.vectorize.io/blog/2026/09/16/stop-growing-your-system-prompt) makes the case for retrieving evidence per turn instead of carrying it everywhere
- [Document file upload in Hindsight Cloud](https://hindsight.vectorize.io/blog/2026/03/09/hindsight-document-upload) is the whole-file path, which is still the right tool when the file is the unit
