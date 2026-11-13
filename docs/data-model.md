# Data model

Draft for review, 2026-11-14.

## Model

Store one versioned, typed directed multigraph across runtime-defined orthogonal dimensions, with an append-only event table. A stable logical node identity is shared across dimensions; immutable revisions identify exact content and relation history. The same revision may be projected into one or several dimensions, while each dimension may keep its own head revision and possibly disconnected subgraph. Dimensions are independent projections over the shared node set, not nested graph layers. Keep normative requirements, proposals, and rationale as distinct node kinds.

Each dimension projects the shared node identity into its own structure, links, revision head, and workflow state; a node may be projected into one, several, or all dimensions. Creating a dimension adds data and does not require a schema migration. A projection-local change leaves other dimensions unchanged unless an explicit cross-dimension relation or operation says otherwise. Use one active approved branch per dimension as a provisional default; whether branches are shared or independent remains open.

Revision selection and support validity are separate projections. In the selected dimension and branch, `head_status` says whether an exact revision is the head, historical, deleted, or unknown; `support_state` says whether that revision is supported, unsupported, or not applicable at the requested event sequence. A dimension's selected head is not automatically supported. Invalidation records a cause and creates a successor revision; earlier revisions and their links remain historical. Whether each dependent that loses its final support needs a same-content successor revision, or only an event plus a derived unsupported state, remains open because revision fanout can be substantial.

The first release supports dimension-local graph structure and revision history. A future implementation-dimension overlay could give a shared specification node an implementation-specific representation and workflow state, with task nodes that exist only in that dimension. This is an extension option, not an MVP requirement; preserve the shared node identity and keep overlay state distinct from normative validity.

## Records

| Record | Required fields | Meaning |
| --- | --- | --- |
| Node | id, created_event_id | Stable logical identity shared across dimensions and revisions. |
| NodeRevision | id, node_id, revision_no, based_on_revision_id?, kind, title, body, scope?, facets?, created_event_id | Immutable exact content revision. Revision numbers increase per logical node; based_on records lineage, including branches made by dimension-local edits. Kinds include requirement, decision, wish, option, exception, argument, evidence, source, question, and risk. |
| Dimension | id, name, metadata, status, created_event_id | Runtime-created scope for an independently structured view over shared logical nodes and revisions. The set of dimensions is data, not a schema enum. |
| DimensionProjection | dimension_id, branch_id, node_id, head_revision_id, acceptance_state, view_kind?, local_state?, attributes?, created_event_id | Represents a shared node identity in one orthogonal dimension and branch, selects that projection's head revision, records proposal/acceptance state, and carries dimension-local representation or workflow state. No projection means the node is absent from that dimension. |
| SourceRef | revision_id, uri_or_source_id, locator, digest?, captured_at | Provenance for an exact revision. The locator is a heading, line range, section, or other stable source coordinate. A digest identifies source bytes, not truth. |
| SupportGroup | id, dimension_id, target_revision_id, branch_id, created_event_id, status | One complete basis supporting a target revision in one dimension. All premise revisions in the group are required. |
| EdgeVersion | id, dimension_id, branch_id, from_revision_id, relation, to_revision_id, created_event_id, properties | Immutable typed relation tied to exact revision endpoints in one dimension and branch, or to a support-group record. Multi-edges are allowed. A revision does not inherit links automatically. |
| Event | seq, id, transaction_id, schema_version, type, target_id, revision_id?, dimension_id?, branch_id?, prior_head_id?, new_head_id?, affected_ids?, source_ref_ids?, at, actor_label, reason, basis_ids | Append-only state transition, revision change, dimension operation, deletion, or branch action. Events in one transaction apply atomically; schema_version supports replay across event-format changes. Deletion events record their exact scope and affected projections/links. Actor label records an assertion; it is not authentication. |
| Branch | id, dimension_id, parent_branch_id, recovery_root_id, status, created_event_id | A historical or active approved context for a dimension. One active branch per dimension is the provisional v1 choice. |
| ProjectionSnapshot | dimension_id, branch_id, event_seq, schema_version, digest, snapshot_data, created_event_id | Optional recovery checkpoint for deterministic replay from an event sequence; retain the event log as the source of truth. |
| TaskRef | system, element_id, dimension_id, revision_id | External reference from a task element to an exact revision in a dimension. Resolved support state and graph sequence are returned by the resolver, not stored as task state. |
| Conflict | id, dimension_id, branch_id, left_revision_id, right_revision_id, scope_overlap, source_evidence, comparison, status, resolution_event_id? | Candidate or confirmed incompatibility between exact revisions in a dimension and branch, with source spans and the comparison that raised it. Status is candidate, confirmed, resolved, deferred, or not_conflict. It does not change either endpoint's support validity automatically. |

Only accepted, supported head revisions of requirement and decision nodes, plus accepted supported scoped-exception heads, enter a dimension's normative view. Wishes and options are proposals; arguments, evidence, sources, questions, and risks explain or qualify them but do not become requirements by being imported. Status updates are events, not requirement nodes.

The optional `view_kind` and local state on a dimension projection are overlays; they do not mutate the shared node identity or revision. For example, a requirement could be represented as a story in an implementation dimension, with separate task nodes and containment links in that dimension. The example overlay tracks code work and development state, not an assignee. Overlay state is not normative validity.

Create, read, update, and delete operations on nodes, links, dimensions, and projections must preserve an auditable history. Updating content creates a new revision and advances the selected dimension's head; a revision may be selected in several dimensions. Deleting the latest revision in one dimension tombstones that selection and restores its previous revision when one exists. Deleting a node in one dimension withdraws only that projection; deleting the whole node tombstones it across dimensions. Deletion events record scope, target, reason, and prior/new heads. Revision and link records remain available in history rather than being physically erased; any need for secure physical purge is a separate retention decision.

Links attach to exact revisions. Opening an earlier revision defaults to its accepted event sequence and returns the links that existed then, never later links folded onto that revision. An explicit graph `as_of_seq` reads a later snapshot and reports newly created edges by their own event and exact endpoints. Creating a new revision does not retarget historical links to the new or old revision; any copy-forward of links must be explicit and recorded. A selected dimension may point to a different head revision than another dimension for the same logical node.

When a new head revision should retain unaffected relations or support groups, the operation explicitly creates replacement edge/group records that point to the new exact revision. It never changes the historical records. Impact previews include these copy-forward edges and any groups that no longer support the new head.

## Support and relation semantics

| Relation | Direction | Effect |
| --- | --- | --- |
| depends_on | support group → required premise revision | Each edge is one required premise in its group. The whole group is supported only when every premise is supported in the selected dimension and branch. |
| justifies | support group → target revision | Multiple valid groups are alternative justifications. Support is present when the target is an accepted root or at least one complete group is valid. |
| contains | parent revision → child revision | Organizes a dimension-local subtree. Moving or relinking changes this structure only in the selected dimension and does not propagate support validity. |
| conflict_candidate | revision → revision | Records a detected pair for review. It does not assert a conflict or alter support validity. |
| contradicts | revision → revision | Records a human-confirmed incompatibility. It does not invalidate either endpoint. Treat the pair as symmetric in queries. |
| supports / objects_to | argument revision → option, decision, or requirement revision | Rationale links only; they do not establish support validity. |
| refines / narrows | new revision → prior revision | Describes scope or lineage. It does not propagate validity unless the required premises are also represented in a support group. |
| supersedes | new revision → prior revision | Records an accepted replacement. The prior revision and its links remain in history. |
| regrows_from | new revision → unsupported or historical anchor revision | Records that an accepted replacement grew from a prior basis in the same dimension. No validity propagation. |
| related_to | revision → revision | Navigation only; no automatic propagation. |
| task_ref | task element → exact revision in a dimension | External reference only; never changes task status or content. |

The `depends_on` relation is only the support-group-to-premise edge and is the only support-validity-propagating relation within a dimension in v1. Candidate conflicts, confirmed contradictions, rationale, containment, refinement, navigation, task references, and module architecture dependencies do not propagate support validity. No relation or state change propagates across dimensions implicitly. A `module_depends_on` relation is a separate architecture query and may contain cycles. Facets filter and classify nodes; they are not extra graph layers.

All depends_on edges within one support group and dimension are conjunctive (AND). Multiple support groups for one target revision in that dimension are alternatives (OR). Only support-group premises participate in validity propagation. A change to one premise invalidates the groups that cite it; a node's head revision is unsupported in that dimension only if no other complete group remains valid and it is not an accepted root.

Reject cycles that have no path to an accepted root in the selected dimension. A circular pair of support groups must not make itself supported. A new support group must cite supported premise revisions in the selected dimension and branch. An unsupported revision cannot be reused as a premise or target for new support in that dimension; history relations such as supersedes and regrows_from may still refer to it.

## State projection

Events are append-only; dimension heads, acceptance state, and support-group status are derived per-dimension projections. These three axes are not interchangeable.

| State axis | Values and meaning |
| --- | --- |
| `head_status` | `head`, `historical`, `deleted`, or `unknown`, relative to one dimension, branch, and event sequence. Another dimension may select a different head. |
| `acceptance_state` | `proposed`, `accepted`, or `rejected`; acceptance records review authority but does not establish support or truth. |
| `support_state` | `supported`, `unsupported`, or `not_applicable`. A head is supported when it is an accepted root or has at least one complete valid support group in this dimension and branch. Unsupported can occur without replacement claim text. |

A conflict state is stored separately and does not imply support or invalidation.

A head revision may be supported or unsupported while it has a candidate, confirmed, deferred, or resolved conflict. Conflict status is stored separately from support validity. A detector creates candidates only; a human confirms and records the outcome. “Latest revision” means selected head; it does not by itself mean supported or true.

Revision and support transitions are separate:

- proposed → accepted head, rejected
- accepted head → historical after a successor revision is accepted; the successor becomes the selected head
- invalidation is a cause event that creates a successor revision; it does not mutate or delete the prior revision
- support state is computed independently for the selected head and event sequence; the new head may be `unsupported`
- repairing an unsupported basis creates a new revision, supported by exact premise revision IDs and linked to the old anchor with `regrows_from`
- deleting the latest revision in one dimension is an auditable tombstone that selects the previous revision there; deleting a node in one dimension and deleting the whole logical node are separate scopes

When a premise changes, first compute affected support groups and dependent nodes that would lose their final support in that dimension. Acceptance writes the source revision, explicit link versions, affected support-group updates, any successor revisions required by the selected invalidation policy, and resulting head/support projections atomically. If the transaction fails, the previous heads and support projections remain intact. Whether every dependent needs a same-content successor revision or only an invalidation event plus unsupported projection is an open choice because revision fanout can be large. Cross-dimension invalidation behavior is also open.

## Branches, time, and regrowth

A branch is an accepted projection with a dimension, parent branch, and recovery root. When an unsupported basis is repaired and accepted, create a successor revision (and, if chosen, a successor branch) in that dimension with a `regrows_from` relation to the historical anchor. Preserve the old branch and revision. Show a node's regrowth count as accepted replacement revisions directly anchored to it; count each accepted revision once, not each descendant. Confirm this metric with the product owner before presenting it publicly.

Keep event sequence and system time in v1. Add domain valid-time intervals only if users need to represent when a fact applied in the world separately from when the system learned or revised it.

## Indexes and queries

Maintain indexes for dimension projection/head revision, logical node and revision IDs, revision lineage, dimension plus acceptance/support state, node kind, facets, source references, branch membership, dimension-scoped incoming/outgoing revision links, support-group membership, and task references. Keep current-view and full-history FTS indexes distinct so history stays opt-in. Search a selected dimension by default. Cross-dimension secondary indexes may speed queries but increase storage and write maintenance; measure need before adding them. Keep small in-memory adjacency and support-count indexes for hot impact queries if measurements justify them.
