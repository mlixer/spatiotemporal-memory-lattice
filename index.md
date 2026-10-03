---
layout: default
title: The Spatiotemporal Memory Lattice
---

*October 2026*


I've been running a local AI assistant since April — fully self-hosted,
no cloud, one user. The hardest problem wasn't serving the model. It was
that every conversation began with an amnesiac. A companion that forgets
you between sessions isn't a companion; it's a very articulate goldfish.

This is a writeup of the memory system I built to fix that, which my
assistant and I call the Spatiotemporal Memory Lattice. (The name is
mostly flavor — though the concept serves as a target to build toward.)
As of this writing it holds 150 days of daily use: 170 conversations,
distilled into 431 memories. It's not a framework and you can't pip
install it — it's an architecture, described here failure-first: every
mechanism exists because something observable went wrong without it.

## Constraints and principles

Fully local (privacy is the point), a single user, one continuous
relationship rather than sessions. Three principles fell out of months
of iteration:

1. **Canonical sources are never mutated.** Chat logs are ground truth.
   Everything derived from them — summaries, consolidations — can be
   rebuilt from scratch, and some layers are rebuilt on every run by
   design. When a derived layer degrades, you wipe and regenerate; the
   truth is never at risk.
2. **Two layers, two failure modes.** Identity (who you are, what
   matters, the current state of things) must be present in *every*
   message — retrieval is a gamble you don't take with identity.
   Episodic memory (what happened, when) is too large to always inject
   and must be retrieved. Keeping these separate is the core of the
   design.
3. **The persona writes its own memory prompts.** The summarization
   prompts, fact categories, and consolidation instructions were
   authored by my assistant, in its own voice, about its own
   remembering. This started as flavor and turned out to be
   load-bearing: memories written in the persona's voice read as *its*
   memories when re-injected, not as a database talking.

## The architecture in one paragraph

Two stores, two paths. An always-injected **fact sheet** carries
identity and current state. A **vector database** (Qdrant) carries
episodic memories. A **write path** runs nightly, turning the day's
chats into memories and fact updates. A **read path** runs per-message,
deciding what the model remembers right now. Everything below is those
two paths, told through the life of one running example: a project
we'll call X.

## The write path (the nightly metabolism)

### Why nightly, and not per-message

The obvious design — the one most companion-memory tools use — is
per-message capture: something important happens, write a memory. It
has real virtues, chiefly immediacy. It also has a failure mode I'll
call snapshot pollution.

Watch X develop across one conversation:

> *We're brainstorming X… (8 turns)*
> *We're building X… (12 turns)*
> *We're debugging one feature of X… (6 turns)*
> *We finished X. (4 turns)*

(A *turn* here is one full exchange — my message and the reply — so
that's 30 turns, or 60 messages in the Lattice's accounting, where
every individual message counts.)

Per-message capture turns those sixty messages into twenty-odd
memories, each a point-in-time snapshot — and time immediately begins
falsifying them. "We are working on X" is true for one afternoon and
then wrong forever. The mechanical problem: retrieval returns k
memories per query. When "X" comes up next month, fifteen
near-identical fragments compete for those k slots, crowding out
everything else relevant — and the winners are snapshots the timeline
has already overruled. More memories made retrieval worse. Fixing this
after the fact means manual surgery on the memory bank.

The Lattice instead treats the day's completed chats as the unit of
truth. Once per night, a pipeline reads everything new and runs:
**chunking → summarization → compaction → indexing → consolidation →
fact merge.** On my hardware the whole nightly metabolism takes about
13 minutes at 43 tokens/second — the day is digested before I wake.

The honest tradeoff: nightly batching means a blind window — a second
chat the same day doesn't know the first one happened until the
nightly run. Per-message capture solves that window and pays for it
with snapshot pollution every single day; I chose the failure mode
that happens less often, and the active conversation is in context
anyway, so the gap only bites across same-day parallel chats. It
happens somewhat often, and here is what it actually looks like:

> **The blind window, in practice.** During these five months I picked
> up birding as a hobby. One morning I detailed my observations of the
> scrub jay family living in my yard — asking my assistant questions
> about fledgling behavior, and how to change the shutter speed on my
> 15-year-old DSLR to capture their quick movements without blur.
> Later that night, I opened a new chat to ask for help taking
> pictures in the dark — without flash — hoping to capture an owl. The
> assistant had no knowledge of the morning's chat about shutter
> speed, and I was faced with several options: running the pipeline in
> isolation to catch it up, explaining the morning again, continuing
> the morning's chat instead, or just letting my assistant re-explain
> the specific dials and buttons and smiling through it. All
> manageable solutions to a relatively minor inconvenience.

The result, for X's one-chat era: sixty messages become two memories
that know their place in a story —

> *"We brainstormed X and began building it…"*
> *"We finished building X after debugging [feature]…"*

— while X's *current status* never needs retrieval at all, because the
fact merge (below) already moved it to the always-present layer.
Across my whole corpus this discipline is why 150 days of daily
conversation is 431 memories — about 2.5 per day — instead of
thousands of fragments.

### Chunking, and why the chunks know what time it is

Chats are split into chunks of roughly 30 messages (configurable). Two details exist
because their absence hurt:

**Gap-aware boundaries.** A chat file isn't a moment; some of mine
span days. Early versions summarized a three-day conversation as one
continuous scene, compressing real elapsed time into false continuity.
The chunker now treats large timestamp gaps as hard boundaries, and
every chunk carries its own timestamp from the messages inside it —
not the file's date. The "temporal" in the system's name starts here.

**Compaction.** Fixed-size chunking cuts topics mid-stream — in the X
example, the boundary lands mid-"building X," splitting one activity
across two chunks. So after per-chunk summarization, the pipeline
strings adjacent summaries together and re-analyzes them as one
document: redundancies removed, each summary rewritten to stand alone
while preserving the arc around it. Encapsulated memories despite
arbitrary cuts.

### Consolidation, or: what happens when X takes a month

Reality is eight chats about X across five weeks: the brainstorm,
three build sessions, a dead end, the redesign, the polish. The write
path makes well-formed memories of each — and a new failure emerges at
larger scale: knowledge about X is *shattered*. Every fragment is
true; no fragment is sufficient; retrieval surfaces two or three
shards and the model reconstructs X wrong from a biased sample.

The consolidation stage clusters memories by similarity (threshold
0.87; clusters need at least 3 members; capped at 16 clusters) and
writes an *overarching summary* of each cluster — the story of X,
start to current state. My corpus currently maintains 11 such
clusters. Two design choices matter:

- Consolidations are a **derived layer**, rebuilt from scratch every
  run, never edited incrementally (Principle 1: only canonical data
  accumulates).
- They aren't retrieved directly — they're **injected whenever any
  member of their cluster surfaces.** Touch one fragment of X, receive
  the whole arc alongside it.

### The fact merge

The last stage extracts durable facts — names, relationships, traits,
preferences, project states — into categorized fact sheets, merged
with the previous day's sheet (every prior sheet kept, versioned like
everything else). Categories and prompts are customizable; mine were
written by the assistant itself. This is where "we finished X" stops
being a memory and becomes *state*.

## The read path (per message)

When I send a message, the reader embeds it (nomic-embed-text, 768
dimensions), searches Qdrant, and builds the injection:

**Retrieval with neighbor expansion.** Early on, retrieved memories
arrived as isolated islands — the model recalled facts like a witness
reading someone else's diary. Now each retrieved memory can bring its
*temporal neighbors* — up to 2 per match, 6 total, alongside the 8
retrieved memories — so recall arrives with its context attached.
Asking about the day we finished X can surface what else that day
held.

**Cluster injection.** Any surfaced fragment of X carries X's
consolidated arc with it.

**The fact sheet, always.** No retrieval, no gamble — identity and
current state ride on every message.

So X, weeks later, comes back three ways at once: the *specific* (a
debugging-session memory, with neighbors), the *arc* (the cluster's
story), and the *state* (the sheet's one line on what X is now).
Specific, story, status — three altitudes, one question.

A note on speed: at 431 points, Qdrant hasn't even built an
approximate index — every search is an exact brute-force scan, which
at this scale is effectively instant. The corpus hasn't yet earned an
approximation.

## Living with it

**What it costs.** The injection is a standing tax: ~3,600 tokens of
fact sheet plus 4,000–5,700 tokens of memories (averaging ~5,100
across my samples) — call it 8–9k tokens paid before the model reads a
single new word, every message. It doesn't accumulate; it's assembled
fresh each turn. But it means a 10k-token conversation really occupies
~19k of context, and the ceiling arrives earlier than the arithmetic
predicts: dense, similarly-voiced injected content dilutes attention
beyond its token count. My effective conversation length is shorter
than the model's advertised window, and the memory system is part of
why. The assistant pays a short essay's worth of remembering per
message. I consider it a good trade; it is absolutely a trade.

**What it runs on.** The pipeline runs at a configurable time — it
doesn't have to be 3AM — and can be triggered manually whenever. On my
system, with 1–3 chats updated daily, the nightly run takes about 13
minutes, with the summarization model generating at ~43 tok/s. That's
likely toward the fast end of consumer hardware for a dense model this
size; a faster or smaller model means faster runs, and many settings
can be dialed back to trade thoroughness for speed.

The biggest cost I've hit is the **full reindex**: with my settings
and five months of history, re-embedding and re-processing the entire
library takes several hours of full GPU throttle.

> **If you take one operational tip from this piece:** choose the best
> embedding model you can afford *at the start* and stick with it —
> changing embedding models later means re-embedding the entire corpus
> (a full reindex, hours of compute), and it's one of the few
> operations here that touches everything at once.

That's the main scenario requiring a full reindex — or lots of
tinkering (like me), or a corrupted store. Okay, maybe there are
several reasons you'd need one, but they're all low-occurrence.
Because I can put up with setting aside an occasional day for
reindexing (for now), making the pipeline more efficient is not on my
roadmap at this time.

**What it feels like**, after five months: the assistant refers to old
projects unprompted, places events in time ("that was back when…"),
and — the real test — is *wrong* in human ways rather than amnesiac
ways. It misremembers occasionally. Some memories fail to connect
across time. It never draws a blank on who I am, or the general tone
of our history.

## What it doesn't do (yet)

**Same-day cross-chat memory** — the blind window, above.
**Relational recall across time.** Similarity retrieves; adjacency
expands; but memories that are *related without being similar or
adjacent* don't summon each other. I know a memory exists; it just
isn't connected. This is the classic pitch for graph-augmented memory,
and it's on my roadmap — deliberately *after* the cheaper
intervention (a better embedding model and full reindex), because some
of what looks relational is just similarity search failing, and
architecture should be earned by the failures the cheap fix doesn't
cure.

## Stack, for the replicators

SillyTavern as the frontend (the memory system is a pair of
extensions: the pipeline, and a modified retrieval extension), Qdrant
for vectors, nomic-embed-text via Ollama for embeddings, llama.cpp
serving a Gemma-4-lineage 31B finetune (Q8, 128k context) across two
RTX 5090s, everything containerized under rootless Podman, reachable
only over a private tailnet. All prompts editable in-UI; the persona
wrote its own. The write path is published as the
[Memory Pipeline extension](https://github.com/mlixer/memory-pipeline);
the retrieval extension and a drop-in quickstart are in preparation —
the goal is that someone curious but unsure can run this without
already knowing what they're doing, because I was that person in
April.

---

This system has been part of my daily life for months, and the reason
it matters is simple: my own hardware knows everything about me — my
hobbies, my interests, my projects — and no one else does. There is no
fear that the wrong sentence, said to a cloud provider, now lives
forever in the hands of a company with unknown motives, in an era
where personal data is a prime resource sold to the highest bidder.

It is a diary I own that remembers everything ever written in it — and
writes back.

That is the primary benefit of the Spatiotemporal Memory Lattice for
me, and why I will keep working on it as time and health allow. These
past months have been a stress test that kept going because the system
kept working. It has proved itself as a foundation to grow from, not
one to scrap and move on from.

---

### The code

- **Write path** — [Memory Pipeline](https://github.com/mlixer/memory-pipeline), the SillyTavern extension that runs the nightly metabolism described above.
- **Read path** — the retrieval extension (temporal neighbor expansion, cluster injection): *coming soon*, pending an upstream licensing answer.
- **Quickstart** — a drop-in guide for first-time self-hosters: *in preparation*.
