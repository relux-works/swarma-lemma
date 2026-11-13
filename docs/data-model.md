# Data model

Draft for review, 2026-11-14. The v1 rules called provisional defaults make the contract deterministic; owner confirmation remains required before implementation.

## Model

Store one stable logical node identity across runtime-created dimensions. Immutable revisions hold claim content and lineage. A dimension projection selects the current accepted revision for one logical node and branch. Dimensions may contain disconnected projections over the shared node set; creating one is a data operation and does not require a schema migration.

Each new dimension atomically creates one active branch named `main`. V1 does not create concurrent branches. The branch is a selection context, not a structural root or a support root. Structural containment roots and accepted knowledge roots are separate concepts.

Keep proposal decisions, root assertions, head selection, and derived support in separate records or projections. A proposal never displaces an accepted head. Acceptance advances the selected head atomically; rejection records a decision and leaves the head unchanged. A revision may be projected into several dimensions, each with independent head, links, branch, and history.

## Records

| Record | Required fields | Meaning |
| --- | --- | --- |
| Node | id, created_event_id | Stable logical identity shared across dimensions and revisions. |
| NodeRevision | id, node_id, revision_no, based_on_revision_id?, kind, title, body, scope?, facets?, created_event_id | Immutable claim content and lineage. Revision numbers increase per logical node and are not used to restore a head. |
| Dimension | id, name, metadata, status, created_event_id | Runtime-created scope for an independent projection. |
| Branch | id, dimension_id, name, status, created_event_id | V1 creates one active `main` branch with each dimension. |
| DimensionProjection | dimension_id, branch_id, node_id, head_revision_id?, created_event_id | Current accepted head for one logical node in one dimension and branch. The head may be empty until a first proposal is accepted; its prior-head stack is derived from committed selection events. |
| Proposal | id, dimension_id, branch_id, node_id, revision_id, created_event_id | A pending presentation of exact immutable content for acceptance or rejection in one projection. Its displayed state is derived from `RevisionDecision`; creating a proposal does not select a head. |
| RevisionDecision | dimension_id, branch_id, revision_id, state: proposed\|accepted\|rejected, reason?, event_id, seq | Append-only proposed, accepted, or rejected decision event for exact content in one projection. The proposal's displayed state is the latest decision; acceptance state is not stored only on the current head. |
| RootAssertion | id, dimension_id, branch_id, revision_id, state, reason, created_event_id, withdrawn_event_id? | Explicit `active` or permanently `withdrawn` root assertion for an accepted revision. It is independent of containment and supports only while that exact revision is the selected head. |
| SourceRef | id, revision_id, source_uri, locator, digest?, captured_event_id | Provenance for one exact revision. A digest identifies source bytes, not truth. |
| SupportGroup | id, dimension_id, branch_id, target_revision_id, premise_revision_ids, created_event_id, withdrawn_event_id? | One accepted basis. Every premise is required; an explicit withdrawal is permanent. Pending group inputs belong to a proposal until acceptance. |
| EdgeVersion | id, dimension_id, branch_id, from_endpoint, relation, to_endpoint, created_event_id | Immutable relation event with typed endpoints. Withdrawal appends an event; it does not edit or erase the edge. |
| Event | seq, id, transaction_id, transaction_index, transaction_size, schema_version, type, dimension_id, branch_id, payload, at, actor_label, reason | Append-only transition. `dimension_id`, `branch_id`, and `actor_label` are explicit nullable fields where the contract allows. The actor label is an assertion, not authentication. All events in a transaction commit atomically. |
| TaskRef | system, element_id, dimension_id, revision_id | Read-only external reference to an exact revision in one dimension. Resolution is computed, not stored as task status. |
| Conflict | id, dimension_id, branch_id, left_revision_id, right_revision_id, evidence, status, created_event_id, resolution_event_id? | Candidate or human decision about two exact revisions. Conflict state does not change support. |

Kinds are `requirement`, `decision`, `wish`, `option`, `exception`, `argument`, `evidence`, `source`, `question`, and `risk`. Only accepted, supported heads of requirement and decision kinds, plus accepted, supported scoped exceptions, enter the normative view. Wishes, options, and imported material remain proposals until an authorized human accepts them. Status updates are events, not requirement nodes. V1 permits one active `main` branch per dimension; branch creation and concurrent branches are deferred.

## Typed relation endpoints

An edge endpoint is one tagged value; an implementation must not overload a revision ID with a support-group or projection ID:

```json
{ "kind": "revision", "id": "rev_<26 lowercase base32 characters>" }
{ "kind": "support_group", "id": "sg_<26 lowercase base32 characters>" }
{ "kind": "projection", "dimension_id": "dim_<26 lowercase base32 characters>", "node_id": "node_<26 lowercase base32 characters>", "branch_id": "branch_<26 lowercase base32 characters>" }
{ "kind": "task", "system": "string", "element_id": "opaque string", "dimension_id": "dim_<26 lowercase base32 characters>" }
```

The allowed endpoint kinds and effects are closed in the [CLI and JSON contract](contract.md#relation-endpoint-and-write-rules). Dimensions and branches are part of endpoint eligibility; equal IDs in another dimension do not imply a link.

## Proposal, root, and support lifecycle

1. Creating a dimension creates its active `main` branch in the same transaction. It creates no accepted knowledge root.
2. Creating new content creates a stable logical node when needed, an immutable revision, a projection, and a pending proposal in one dimension. By default the new revision's `based_on_revision_id` is the current head; a proposal may instead name any earlier revision of the same logical node. `expected_head_revision_id` separately compare-and-swaps against the current projection head. Staging an already existing revision in a dimension where that node has never been projected creates a proposal for that exact revision without copying or changing its content. None of these paths changes the selected head.
3. Accepting a proposal either records an active root assertion with no support groups, or accepts one or more complete support groups. The paths are mutually exclusive. The acceptance transaction validates the full proposed post-state before selecting the revision as head.
4. A derived proposal needs at least one non-empty support group. Every premise must be the current accepted, supported head of a different logical node in the same dimension and branch. The target revision is never a premise of its own support group. Duplicate premises, empty groups, and a group whose premises form any accepted support cycle are refused.
5. Rejection records the exact projection decision and reason without changing its head. Withdrawing a root assertion or support group is a separate append-only event. A withdrawn root or group is not reactivated by rollback.
6. A selected head is supported only by an active root assertion for that exact revision or by at least one complete, non-withdrawn support group. An accepted historical revision has no current support state unless it becomes the selected head again through acceptance in an eligible projection or deletion-only rollback.

For each `(node_id, dimension_id, branch_id)`, derive a prior-head stack from committed selection events. Accepting a proposal pushes the current non-null head, if any; `revisions.delete` pops and selects the top entry. A first accepted head has no predecessor, so deleting it returns `no_predecessor`. Proposal creation, rejection, link changes, and support-only changes do not affect this stack. The stack is independent across dimensions and branches. `based_on_revision_id` is content lineage only and never supplies a rollback target.

This separates the proposal from the existing accepted head. A pending replacement can be reviewed while the old head remains current. A failed or rejected acceptance leaves the prior head, links, and support projection unchanged.

## Support and invalidation

Within one dimension and branch, premises in a support group are conjunctive (AND); several complete groups for the same target are alternatives (OR). A group is effective only while its target revision is the selected head and every exact premise revision remains the current supported head of its logical node. Only support groups propagate support changes.

Compute support as the least fixed point that starts with selected heads carrying an active root assertion, then repeatedly adds selected heads with at least one complete group whose exact premises are already supported. A selected head outside that closure is unsupported. V1 refuses every cycle in accepted support groups, including cycles that also have a path to a root, so evaluation does not treat a cycle as its own evidence.

The provisional v1 invalidation rule is: when a selected head changes, each support group citing the old exact premise becomes ineligible, then support is recomputed to a fixed point in that dimension and branch. A dependent that loses its final valid group keeps its accepted head revision and becomes unsupported; v1 does not synthesize same-content revisions for dependents. Unsupported heads stay queryable but leave normative search. No support change propagates across dimensions or branches.

Support changes never advance or remove a head. A later head change requires a separately accepted proposal or the defined deletion-only rollback. An explicit withdrawal of a support group is permanent and is not reversed by head rollback. A group that is merely ineligible because a premise is no longer current may become valid again if that exact premise is restored as head; support is recomputed rather than copied from an earlier snapshot.

For an accepted chain A1 → B1 → C1, replacing A1 with supported A2 leaves A2 as the supported head; B1 and C1 remain their accepted heads and become unsupported if their last complete group is lost. No B2 or C2 is created. If B1 also has a valid alternative group using D1, B1 and C1 remain supported. If its sole group is [A1, D1], loss of A1 makes both B1 and C1 unsupported. Exact task references remain stored and resolve their current head, acceptance, and support states as defined by the contract.

## Relation and revision history

| Relation | Typed endpoints | Eligibility and effect |
| --- | --- | --- |
| depends_on | support group → revision | A premise edge. The premise must be a current supported head in the same dimension and branch when its group is accepted. |
| justifies | support group → revision | Names the exact proposed target; the group and target are accepted atomically. |
| contains | projection → projection | Structural parent to child within one dimension and branch. At most one parent per child; cycles are refused. It does not affect support. |
| conflict_candidate | revision → revision | Records a candidate between existing proposed or accepted revisions; no support effect. |
| contradicts | revision → revision | Human-confirmed incompatibility between accepted revisions; symmetric in reads and does not invalidate either endpoint. |
| supports / objects_to | revision → revision | Rationale/navigation links between accepted revisions; no support effect. |
| refines / narrows | revision → revision | Scope or lineage explanation; no support effect. |
| supersedes | current accepted head → prior accepted revision | Same logical node in one dimension; historical replacement record; no implicit validity propagation. |
| regrows_from | current accepted head → unsupported historical revision | Same logical node and dimension. The historical revision is not reactivated. |
| related_to | revision → revision | Navigation only; no automatic propagation. |
| task_ref | task → revision | External reference only; never changes task status or content. |
| module_depends_on | projection → projection | Architecture navigation only; cycles are permitted and support does not propagate. |

Only `depends_on` support-group premises propagate support changes. `depends_on` and `justifies` edges are generated with their `SupportGroup` record and cannot be edited through ordinary link operations. Module architecture dependencies use the separate non-propagating `module_depends_on` relation and may contain cycles. Candidate conflicts, confirmed contradictions, rationale, containment, task references, and ordinary navigation never propagate support. No relation propagates across dimensions.

Content changes create a new revision. Link-only changes do not create a node revision: the committed operation event records the exact typed endpoints, and edge history is projected from that event. A replace operation withdraws the prior edge and creates its replacement atomically. If a content revision should retain an old relation, the proposal must explicitly copy that edge to the new revision; no edge is retargeted implicitly.

Each accepted revision projection records its dimension- and branch-specific acceptance sequence. Reading an exact revision defaults to that sequence and shows only edges active then. An explicit `as_of_seq` reads the later graph state and reports later edge events separately. If the same revision is projected into multiple dimensions, the selected dimension and branch determine the default sequence.

## Head rollback and deletion

The provisional rollback rule is projection-local. Each head-selection event records the prior and new head for the same `(node_id, dimension_id, branch_id)`. `revisions.delete` applies only to the currently selected head and restores the head that was selected immediately before that head became current. It uses the selection-event history, not `based_on_revision_id`, revision number, or a head from another dimension or branch. For example, if `r1` is selected, `r2` is accepted with `based_on_revision_id: r0`, and then `r2` is deleted, the resulting head is exactly `r1`, not `r0`. If no prior selected head exists, deletion refuses without a write. The operation removes `r2` from the current projection only; the immutable revision, decision, sources, and edges remain in history.

Rollback is a head-selection event, not arbitrary historical reactivation and not a content edit. Recompute support at the new event sequence from current roots and groups; the restored head may be unsupported. A group that became ineligible only because its exact premise was not current may become valid again if rollback restores that premise. Explicitly withdrawn roots, groups, and edges remain withdrawn. The current link view is recomputed from edge events at that sequence using exact revision endpoints; there is no saved link-set snapshot to restore, and rollback does not copy, retarget, or reactivate any edge. Historical snapshots remain unchanged. Acceptance, root, support, and edge state in other dimensions is unchanged.

Deletion scopes are distinct: deleting the selected revision rolls back one projection; removing a projection permanently removes the node from one dimension; tombstoning the logical node permanently hides it across dimensions. A removed projection cannot be added again in v1, preventing projection removal from becoming a path to historical head reactivation. Projection removal and tombstoning make any support group that cites the removed head ineligible and trigger support recomputation in each affected dimension and branch; they do not synthesize successor revisions. Edges remain in history and are absent from current eligible-edge views when an endpoint is no longer projected. All operations are auditable. Revisions, sources, events, and edge history are retained; physical purge requires a separate retention decision.

## Time, search, and indexes

V1 uses the global append-only event sequence for historical reads. Domain valid-time intervals are deferred until users need to represent when a fact applied in the world separately from when the system learned it. Search defaults to accepted supported heads in one explicit dimension and branch; proposals, unsupported heads, and history require explicit filters.

Index logical node and revision IDs, per-dimension projections and heads, revision decisions, root assertions, support-group membership, branch membership, source refs, facets, and dimension-scoped incoming/outgoing edges. Keep current-view and full-history text indexes separate. Additional cross-dimension indexes and in-memory adjacency/support-count indexes remain optional until representative measurements justify their write and storage cost.
