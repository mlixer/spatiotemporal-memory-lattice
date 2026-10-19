# The Spatiotemporal Memory Lattice

A hybrid memory architecture for a persistent, fully-local AI
companion: an always-injected fact sheet for identity and current
state, a vector store for episodic memory, a nightly write path that
digests each day's conversations into encapsulated memories, and a
per-message read path that recalls them with their temporal context
attached.

Built and run daily since April 2026 on fully self-hosted hardware.
As of this writing: 150 days, 170 conversations, 431 memories.

**📖 [Read the full write-up](https://mlixer.github.io/spatiotemporal-memory-lattice/)** —
the architecture described failure-first: every mechanism exists
because something observable went wrong without it.

## The pieces

| Component | What it does | Status |
|---|---|---|
| [Memory Lattice](https://github.com/mlixer/memory-lattice) | **Write path + Read Path** SillyTavern extension: nightly chunking → summarization → compaction → indexing → consolidation → fact merge. Per-message recall with temporal neighbor expansion and cluster-summary injection | **Published** |
| Quickstart | A drop-in guide for first-time self-hosters | In preparation |

## Stack

SillyTavern · Qdrant · nomic-embed-text (Ollama) · llama.cpp ·
rootless Podman. Fully local; privacy is the point.

## License

Code repos carry their own licenses (AGPL-3.0). The article is
© its author; quote freely with attribution.
