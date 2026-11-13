# MVP scope and risks

Draft for review, 2026-11-14.

## Goal

Provide a local product knowledge graph that answers:

1. What requirement or decision is the accepted, supported head in this dimension?
2. Which exact sources and complete support groups justify it?
3. What becomes unsupported when one exact premise stops being current?
4. Which claims remain supported through an alternative complete group?
5. What was selected before, what replaced it, and what can deletion-only rollback restore?
6. Which exact revisions do task elements reference?

The store records evidence, accepted decisions, and their history. It does not prove that natural-language claims are true or establish the authority of the person named by an actor label.

## Included

- A local Go module and CLI with a closed, versioned JSON protocol.
- SQLite as the first durable-store candidate, with append-only versioned events and transactional materialized projections; replay checkpoints are added only when measured replay cost warrants them.
- Stable logical node IDs, immutable revisions, and per-dimension projection heads. An existing exact revision can be staged into another dimension without copying its content.
- Runtime-created dimensions, each with one active `main` branch in v1. Creating a dimension and its branch is one transaction and needs no schema migration.
- Separate proposal decisions, root assertions, selected heads, and derived support states.
- Explicit support groups with conjunctive premises inside a group and alternative complete groups for one target.
- A provisional event-only invalidation rule: retain dependent revisions, recalculate support, and expose unsupported heads outside the default normative search.
- Revision-pinned links, exact link event history, explicit copy-forward, and dimension-local containment moves over a one-parent forest.
- Separate projection removal, cross-dimension node tombstone, and selected-head rollback operations. All retain immutable history; physical purge is not part of v1.
- Proposal, accept, reject, impact preview, conflict candidate staging/review, support-group withdrawal, import staging, and repair through a new accepted revision.
- A versioned export for one dimension and branch, plus a read-only task-reference resolver that stays unavailable until the task-system owner agrees to its schema and access mapping.
- Search over supported heads by default; unsupported heads, proposals, and historical revisions require explicit filters.

## Deferred

- Concurrent active branches within a dimension, shared branches across dimensions, and full ATMS environment enumeration.
- Automatic acceptance, conflict resolution, or task-state writes.
- Multi-user authorization, a network service, remote synchronization, and broad write access from automation clients.
- Full-corpus import, automatic conversion of parser output into accepted requirements, and source-content copying outside existing source custody.
- Domain valid-time queries, performance promises, and conflict-precision or recall claims before representative measurements and an owner-labeled evaluation set exist.
- Interactive visualization until the first working prototype; the versioned export is the integration seam.
- Multiple structural parents, cross-dimension support propagation, implementation overlays, and secondary indexes beyond the baseline projection and edge indexes.

## Provisional defaults and owner decisions

These recommendations make the v1 contract deterministic. They are not owner approvals; decisions that affect persisted history or wire compatibility must be affirmed before implementation.

| Decision | Options | Recommended v1 default | Consequence |
| --- | --- | --- | --- |
| Selected-head deletion (B1) | Restore the prior selected head; restore `based_on`; or forbid deletion | Restore the immediately preceding selection for the same node, dimension, and branch. Preserve the deleted revision and all links in history; recompute support. | Gives a predictable undo even when lineage differs. It can restore an unsupported head; it never reactivates withdrawn roots, groups, or edges. |
| Lost support (B3) | Create same-content successor revisions for every affected dependent; or retain heads and derive unsupported state | Retain affected heads and recalculate support without synthetic revisions. Do this only in the changed dimension and branch. | Avoids revision fanout and keeps claim-content history honest. Consumers must read support state separately from head and acceptance state. |
| Containment moves (B5) | General multi-parent graph; or one-parent forest | At most one parent per projected node. A move changes only the moved root's parent edge; descendants remain attached. Reject shared-parent, cyclic, malformed, or stale-preview topology. | Makes subtree membership and moves deterministic; a node cannot be structurally shared under two parents in v1. |
| Branch scope | One active branch per dimension; multiple branches per dimension; or a branch shared by dimensions | One active `main` branch per dimension. | Keeps the current selection unambiguous. Concurrent alternatives require a later schema and query extension. |
| Cross-dimension behavior | Global support and invalidation; or independent projections | Heads, support, links, and rollback are local to the selected dimension and branch. | Prevents one dimension from silently changing another; cross-dimension comparisons are explicit queries only. |
| Regrowth count | Count every retry, every descendant, or direct accepted repairs | Count one accepted replacement revision directly anchored to an unsupported historical revision; failed proposals and later descendants do not increment it. | Makes repair lineage measurable without conflating retries or downstream work. |
| Base versus derived requirements | Add a separate kind; or express origin through roots and support | Use root assertions and support groups; do not add a second base/derived kind in v1. | Avoids duplicate semantics. A later need for distinct query behavior can add a versioned field. |
| Acceptance authority | Trust the local actor label; or add an authorization mechanism | Keep v1 single-writer and require a human acceptance step in the workflow; record an actor label as evidence only. Do not treat it as authentication. | The local store cannot prove who approved a change. Multi-user authority is deferred, not implied by the CLI. |
| Source migration and import | Full automatic corpus migration; or reviewed staged intake from an owner-selected, redacted source set | Import only an explicitly selected source set; preserve source URI, digest, and locator; create proposals only. | Adds review work and avoids silently converting extraction into normative knowledge. Corpus size and source quality remain unmeasured. |
| Knowledge source of truth | Make the graph authoritative immediately; or keep it as a reviewed projection of existing documents | Treat v1 as a reviewed projection; preserve original source custody and require human acceptance for graph updates. | Avoids two-way sync and unresolved document ownership in the first release; authors still need a later migration decision. |
| Search defaults | Search all records; or current supported view with opt-in history | Return accepted supported heads in the selected dimension and branch. Require explicit filters for unsupported heads, proposals, and historical revisions. | Keeps normative search focused; broad recall needs an explicit query. |
| Task-reference shape and access | Read/write integration; list of exact `{system,element_id,dimension_id,revision_id}` refs; or repeated scalar fields | Use a list of exact references and a read-only resolver; do not enable it until the task-system owner accepts both the fields and access mapping. | Task state remains authoritative in its own system; an unavailable or denied resolver is distinct from an empty result. |
| Conflict detection | Auto-confirm matches; stage candidates; or omit detection | Candidates are advisory, carry source spans and scope evidence, and need a human decision. Candidate generation never changes support. | False positives and misses remain possible; no quality claim is made without an owner-labeled set. |
| Time semantics | System event time only; or domain valid-time intervals | Use the global event sequence and UTC event time. Defer domain valid time. | The graph can answer when it recorded a change, not when a claim applied in the world. |
| Historical links | Allow non-propagating links to accepted historical revisions; or allow links only to current heads | Allow exact historical endpoints for rationale, conflict, refinement, supersession, regrowth, and task references; require current supported premises for `depends_on`. | Preserves what an edge meant without moving its endpoint when a head changes; each view must display the endpoint's head and support state. |
| Deletion and retention | Physical purge; or auditable tombstones | Keep revisions, sources, and edge history; use projection removal or logical-node tombstones. Decide any secure-purge obligation separately before implementation. | Tombstones hide records from current views but do not erase their content. |
| Export and views | Select a visualization now; or keep a renderer-neutral export | Define deterministic `swarma-lemma-export/v1`; begin visualization after the first prototype. | Consumers can build later views without coupling graph semantics to layout. |
| First graph views | Dependency path, focused neighborhood, impact preview, history timeline; or a combined view | Start with exact dependency path and impact/history views after the first prototype; defer an interactive combined view. | Keeps relation type, direction, support, and history visible before choosing a renderer. |
| Implementation overlays | Add implementation-only representations and task nodes in v1; or defer them | Defer overlays until identity, local history, queries, and cross-dimension invalidation are specified. | Avoids mixing normative requirements with workflow state; later overlays require a versioned contract. |
| Event replay | Best-effort tail recovery; or schema- and sequence-validated frame replay | Validate each length-prefixed frame, event schema, and strictly increasing sequence; v1 stores one event per mutation (`transaction_size: 1`). Ignore only an incomplete final frame; fail closed on a malformed complete frame or sequence gap. | Preserves earlier committed history and exposes incomplete-tail evidence instead of applying a partial write. |
| Indexes and performance | Prebuild broad indexes and targets; or measure first | Keep basic dimension/projection and edge indexes; benchmark representative 1k, 10k, and 50k datasets before adding secondary indexes or targets. | Avoids unmeasured write and storage cost; no latency promise exists yet. |

## Acceptance controls for an implementation

These examples define observable claims for later executable contract fixtures. Both the valid path and the nearby refusal must be checked; no implementation or product test run is claimed by this design document.

| Area | Valid control | Negative control and required state after refusal |
| --- | --- | --- |
| Dimension and proposal lifecycle (B2) | Create a dimension with active `main`; accept an explicit root; stage and accept a supported proposal; verify one atomic head change. | Reject a proposal and verify the prior head, links, and support are unchanged. Refuse root-plus-groups, an ungrounded derived proposal, and a premise from another dimension without partial events. |
| Rollback (B1) | Select `r1`; accept `r2` with `based_on_revision_id: r0`; delete selected `r2`; restore exactly prior selected head `r1`, not lineage parent `r0`. Change one premise after `r2` is selected; rollback must recompute `r1`'s support against the current premise head and show only edge state active at the new sequence. | Refuse a stale expected head, a non-head target, and deletion with no prior selection. After rollback to the first selected head, another deletion returns `no_predecessor` and leaves that head unchanged. Verify another dimension's head, support, and links are unchanged; direct historical reactivation stays refused. |
| Invalidation (B3) | For A1 → B1 → C1, replace A1 with supported A2; retain B1/C1 and mark them unsupported if their last complete groups are lost. With a valid alternate group through D1, retain support. | For an AND group [A1,D1], loss of A1 makes the target unsupported. Refuse an accepted support cycle, including a cycle reachable from a root. Verify no synthetic B2/C2 revisions and no changes outside the selected dimension/branch. |
| Link history (B4) | Accept a revision at sequence 10, add an edge at sequence 20, and read the revision at both cutoffs: the edge is absent at 10 and present as a separate event at 20. Add a non-propagating relation to an accepted historical revision and preserve that exact endpoint. Add, replace, and withdraw create no node revision; explicit copy-forward creates a new edge ID. | Reject a `depends_on` edge to a historical or unsupported premise, implicit copy-forward, duplicate active edges, an invalid endpoint, and direct writes to generated support or containment relations. A failed replace leaves the old edge active. |
| Move semantics (B5) | Preview and move a node in a one-parent forest; verify exactly one parent edge changes and all descendants stay under the moved root. Other dimensions remain unchanged. | Reject shared-parent or cyclic topology and a stale preview with no edge or projection changes. Confirm direct `contains` link mutation is refused. |
| Protocol and persistence (B6) | Decode every v1 operation against its closed schema; export and replay one committed transaction and reproduce exact heads, support, edges, and sequence. Repeat an identical import key and receive the original proposal IDs without another event. | Reject unknown fields, IDs, enums, invalid event-frame metadata, mismatched idempotency reuse, unsupported endpoint forms, and stale previews with stable codes and no partial writes. Verify a deliberately truncated final frame is ignored while a malformed complete frame fails closed. |
| Task resolver | Resolve a known authorized reference and preserve input order; return a successful empty list for an empty request. | Unknown, denied, and unavailable remain three distinct outcomes; none changes task status or turns an error into an empty success. |

## Main risks

- Support groups are richer than plain edges; incorrect grouping can produce false supported or unsupported state.
- The event-only invalidation default is a product decision: if the owner requires a same-content revision for each affected dependent, persisted revision history and the impact preview must change before implementation.
- A single active branch per dimension may be insufficient for concurrent scenarios.
- Event history adds replay, schema-evolution, and projection-maintenance cost. SQLite has one writer; WAL is not suitable for network filesystems.
- Go SQLite driver choice affects CGO/compiler needs and FTS5 availability. Confirm FTS5 in the selected supported build before treating SQLite as the durable store.
- FTS5 tokenization does not guarantee good Russian morphology or domain vocabulary search.
- A local actor label cannot prove approval authority.
- Source corpus structure, provenance quality, and migration cost remain unmeasured.
- Conflict candidates can be wrong; no precision or recall claim is made without an owner-labeled evaluation set.
- Dimension-local views and workflow state can diverge; implicit cross-dimension updates are forbidden by the provisional default.
- Tombstones preserve history but do not securely erase content.
- The one-parent forest simplifies moves but excludes shared structural parents in v1.
- Overlays can blur shared node identity with local representation, history, queries, and invalidation; they remain deferred.
