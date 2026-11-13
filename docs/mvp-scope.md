# MVP scope and risks

Draft for review, 2026-11-14.

## Goal

Provide a local product knowledge graph that answers:

1. What is the current accepted requirement or decision?
2. Which exact sources and complete support groups justify it?
3. What becomes unsupported if one premise changes?
4. Which claims remain supported through an alternative support group?
5. What was accepted before, what replaced it, and where did the new branch grow?
6. Which task elements reference this knowledge?

## Included

- Go module and CLI with a versioned JSON interface.
- SQLite persistence with append-only, schema-versioned events, transactional writes, materialized head/acceptance/support projections, and replay checkpoints when measured replay cost warrants them.
- The local store is single-writer; a recorded actor label does not authenticate the caller or prove approval authority.
- Shared logical node identities with immutable revisions and per-dimension head pointers, source references, revision-pinned links, support groups, one provisional active approved branch per dimension, and an explicit history view.
- Runtime-created orthogonal dimensions with per-dimension node projections, state, links, and disconnected graph views; dimension creation is data-only and requires no schema migration.
- Dimension CRUD, per-dimension node-projection add/remove, dimension-scoped link CRUD, and history-preserving moves/re-linking within one dimension.
- Auditable revision deletion that restores the prior head, whole-node tombstones, and dimension-only projection removal; these remain distinct from support invalidation.
- Proposal, accept, reject, conflict-candidate review, supersession, impact preview, support-group withdrawal, invalidation by successor revision, and repair through a new revision linked with `regrows_from`.
- Typed scope and node kinds that keep requirements and decisions distinct from wishes, options, exceptions, arguments, evidence, source records, questions, and risks; relation vocabulary separates support, rationale, conflicts, and history.
- Current-state search with exact ID and facet filters; history and proposals are opt-in.
- Read-only task-reference resolver and impact lookup, with each immutable revision reference bound to a dimension; the board remains authoritative for task state and the adapter stays optional.
- Staged import that records source hashes and spans and produces proposals only.
- A versioned export seam for an exact dimension, head revisions, support states, and revision-pinned links.
- A project skill explaining search-first use, exact node citation, support-group semantics, proposal workflows, and the human approval boundary.

## Deferred

- Automatic acceptance of detector output or automated resolution of semantic conflicts.
- Multiple concurrently active approved contexts within one dimension, full ATMS environment enumeration, hyperedges, or a specialized multilayer graph engine.
- A network service, multi-user authorization, remote synchronization, or broad write access from automation clients.
- Full corpus migration, generated product claims, or importing private source contents without explicit review.
- Performance promises before reproducible benchmarks exist.
- Interactive graph visualization and renderer selection until after the first prototype; keep the versioned export seam available for that later work.
- Per-dimension representation overlays such as a specification node displayed as an implementation story with implementation-only task nodes. Keep this as a future option with explicit identity, history, query, and invalidation costs; board replacement is not an MVP goal.
- Additional cross-dimension indexes beyond basic projection and edge indexes until measurements show they are needed.

## Acceptance evidence for implementation

- A node’s text cannot be overwritten; a replacement receives a new exact revision ID under the same logical node and preserves the prior revision.
- A logical node has immutable revision history; each dimension selects its own head revision, and opening an earlier revision shows the exact links attached to it at that time.
- Opening a historical revision defaults to its accepted sequence and excludes later links; an explicit later `as_of_seq` shows later edges as separate events with exact endpoints.
- Creating a revision never retargets old links implicitly; copy-forward creates new versioned links in an auditable operation.
- Invalidation creates a successor revision and cause event, retains the predecessor, and marks the new head unsupported when it loses all valid support, even when the successor revision changes no claim text.
- The latest revision is not treated as supported merely because it is the head; head status, acceptance state, and derived support state are reported separately.
- Only accepted, supported head revisions enter normative search by default; proposed and rejected revisions remain explicitly queryable.
- The same exact revision can be projected into several dimensions or only one; changing its projection or links in one dimension leaves other dimensions unchanged.
- A new dimension can be created at runtime without a database schema migration, and create/read/update/delete operations retain dimension history.
- Removing a projection in one dimension does not erase the shared logical node, its revisions, or its projections and history in other dimensions; retiring a referenced dimension does not physically erase its history.
- A dimension can contain a disconnected graph; dimension-scoped link CRUD and re-linking do not mutate links in another dimension.
- Moving a node with its structural descendants in one dimension preserves the prior topology and does not alter its projection or structure in other dimensions.
- Deleting a latest revision in one dimension restores its previous revision there; if there is no predecessor, the operation refuses without changing history.
- Deleting a whole logical node hides it across dimensions through an auditable tombstone; deleting it in one dimension leaves every other dimension and its links unchanged.
- A delete event records the operation scope, reason, prior and new heads, and affected revision/link IDs; an invalidation never uses a delete event.
- A support group is valid only when every required premise is supported in that dimension; a cycle with no accepted root remains unresolved.
- A cycle in accepted depends_on relations is refused without changing support state in the selected or any other dimension.
- Duplicate IDs, dangling links, unknown enums, invalid cardinality, and unsupported support cycles are refused before any event or projection changes.
- If a revision has two complete support groups and one loses a premise, that group is withdrawn but the head remains supported through the explicitly copied forward surviving group.
- If the last complete support group is withdrawn in one dimension, the target becomes unsupported there and only dependents in that dimension that lose their final valid group become unsupported.
- Changing a prerequisite reports exact affected groups, before/after support counts, and the reverse-reachable nodes that would lose all support.
- A conflict candidate includes exact source spans, normalized subject/property/value comparison, scope overlap, and conditions; it does not change support state.
- Matching words with different subjects or non-overlapping scope do not become a confirmed conflict.
- A human can confirm, resolve, defer, or mark a candidate `not_conflict`; the result is an append-only event, and changed knowledge is a separate new-revision proposal.
- Rationale links such as supports/objects_to, refinement links, and confirmed contradictions leave support-group validity unchanged.
- A new support group cannot target an unsupported head or depend on an unsupported premise revision; a history relation may still reference prior revisions.
- A failed or rejected mutation creates no partial event, changed branch, or partial invalidation.
- A stale impact preview is refused and must be recomputed.
- Default search returns accepted, supported head revisions; explicit unsupported-head, history, and proposal filters return those records separately.
- Repair creates a new revision under the same logical node, cites supported premise revisions, links `regrows_from` to the unsupported historical anchor, and preserves the old revision and invalidation event.
- A historical revision cannot become the head again; attempted reactivation is refused without changing history.
- The versioned export preserves the selected dimension, head revision IDs, support state, and links pinned to those revisions.
- Task references resolve exact IDs, dimension, and warnings without changing task content or status.
- Unknown references, denied reads, and service outages are distinct from a successful empty result; all preserve the stored task reference and task state.
- Import preserves source digests and spans and creates proposals only.
- Importing a wish, option, argument, evidence item, or status update cannot silently make a current requirement.
- Replaying to an event sequence reproduces its head, acceptance, support, and link views; a malformed multi-event transaction changes no projection, and incomplete-tail recovery has a documented result.
- Search and impact benchmarks cover current and history queries across planned 1k, 10k, and 50k datasets with realistic text, alternate support groups, high-fanout nodes, and English, Russian, and mixed-language queries. Report p50/p95, database size, cold startup/replay, and ingest/update cost; set targets only after measurements.
- The chosen Go SQLite driver can create FTS5 tables in each supported release build; a missing FTS5 feature fails a bounded storage preflight clearly.

## Main risks

- Support-group semantics are richer than a plain edge list; incomplete or misgrouped premises can create false supported or unsupported results.
- Semantic conflict candidates depend on correct subject and scope data; missing conditions, exceptions, or time ranges create false positives and misses. No precision/recall claim is made without an owner-labeled evaluation set.
- The supplied vision's tree-wide invalidation wording may mean every descendant becomes invalid when any premise changes. Truth-maintenance semantics instead withdraw only affected support groups and retain a claim while another complete accepted group remains. An earlier conservative model would invalidate more; settle this semantic difference before implementation.
- A single active branch per dimension simplifies support queries but may be insufficient for concurrent product scenarios.
- Event sourcing adds replay, schema-evolution, and projection-maintenance costs; keep it only because audit and history are required.
- SQLite WAL allows concurrent readers with a writer on the same host but still has one writer and is not for network filesystems.
- Go driver choice changes CGO/compiler needs and feature availability; confirm FTS5 in the selected build before treating SQLite as the durable store.
- FTS5 tokenizer behavior does not guarantee good Russian morphology or domain vocabulary search.
- The local CLI can record an actor label but cannot prove that the person is authorized to accept.
- Existing source corpus structure, provenance quality, and migration cost remain unmeasured.
- Visualization can imply meaning through layout; every view must distinguish dependency from proximity.
- Orthogonal dimension projections and workflow state can diverge; implicit cross-dimension updates or invalidation would surprise users.
- Creating a same-content successor revision for every dependent that loses support may create large revision cascades; an event plus derived unsupported state avoids this churn but must be reconciled with the owner’s invalidation requirement.
- Collapsing head status, acceptance, and support into one flag would hide unsupported accepted heads and proposed or rejected revisions; keep the three axes explicit in storage and responses.
- Auditable tombstones preserve history but do not securely erase content; any physical-purge requirement needs a separate retention policy.
- Moves over a general graph may encounter shared descendants or multiple structural parents; subtree selection and reparenting rules need a precise contract.
- Per-dimension overlays can blur shared node identity with local representation and state, increasing history, query, indexing, and invalidation complexity.

## Owner decisions needed before implementation

1. Invalidation: should every descendant become invalid when a premise changes, or only claims that lose their final complete support group? The supplied vision and an earlier conservative model suggest the broader reading; truth-maintenance semantics recommend the latter. The design currently recommends support-group semantics, pending owner decision.
2. Are approved branches independent per dimension or shared across dimensions?
3. Confirm the regrowth count: one per accepted replacement revision/root directly anchored at an unsupported historical revision.
4. Should changes to a node's revision history or support validity propagate across dimensions, and which explicit cross-dimension relations are allowed?
5. Does the board schema support a list-shaped `knowledge_refs` field of exact revision IDs paired with dimensions, or is a stable repeated-field encoding needed?
6. Who may accept or reject proposals and approve a repair, and what local authority mechanism is required?
7. Is the knowledge graph initially a reviewed projection of existing documents, or should it become the authoritative source?
8. Which source collections are in migration scope, and what redaction/approval rules govern import?
9. What search latency and corpus-size targets matter after the first local dogfood?
10. Which first graph views are needed: dependency path, focused neighborhood, impact preview, history timeline, or a combination?
11. Does the task-system owner accept a read-only reference and resolver seam before any write integration?
12. How does the task system's access policy map to knowledge-node visibility and resolver output?
13. Does v1 suggest conflict candidates automatically, and what source evidence should a reviewer see? A candidate never resolves a conflict by itself.
14. Does v1 need domain valid-time queries, or is event/system history sufficient?
15. What happens when a node with several structural parents is moved with its descendants in one dimension?
16. Is the implementation-dimension story/task overlay worth the added local representation, state, history, and cross-dimension invalidation rules?
17. Are multi-dimensional secondary indexes needed at launch, and which query cost justifies maintaining them?
18. When a dependent loses its final support without a content edit of its own, must it receive a same-content successor revision, or can an invalidation event plus derived unsupported state cover it?
19. What is the required audit/retention behavior for whole-node, latest-revision, and dimension-only deletion, including any physical-purge requirement?
20. Which versioned export format is needed before visualization begins after the first prototype?
21. Can an active dimension-scoped link point to a historical revision, and which relation types are history-only? The as-of view must still keep later edges separate from an earlier revision snapshot.
22. Should requirement nodes distinguish base requirements from derived requirements, or do accepted roots and support groups express that difference sufficiently?
23. Which event-log framing, snapshot/checkpoint, and incomplete-tail recovery behavior is required for deterministic replay?
