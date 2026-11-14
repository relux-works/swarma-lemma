# CLI and JSON contract

Draft for review, 2026-11-14. Operation names and schemas below are the proposed v1 contract; the task-system adapter remains unavailable until its owner agrees to the reference fields and access mapping.

## Protocol v1

The CLI and JSON API use the same operation names and values. A request is one JSON object with exactly `version`, `operation`, and `input`, plus optional top-level `actor_label` for an unauthenticated audit label:

```json
{
  "version": 1,
  "operation": "nodes.show",
  "actor_label": "local analyst",
  "input": {
    "dimension_id": "dim_<26 lowercase base32 characters>",
    "branch_id": "branch_<26 lowercase base32 characters>",
    "node_id": "node_<26 lowercase base32 characters>"
  }
}
```

A successful response is exactly `{ "version": 1, "ok": true, "result": ... }`. A refused request is exactly `{ "version": 1, "ok": false, "error": { "code": ..., "message": ..., "ids"?: [...] } }` and uses a stable error code:

```json
{
  "version": 1,
  "ok": false,
  "error": {
    "code": "stale_head",
    "message": "The selected head changed after the request was prepared.",
    "ids": ["node_<26 lowercase base32 characters>"]
  }
}
```

V1 is closed: unknown fields, operations, enum values, and protocol versions are refused. JSON property names are case-sensitive. Duplicate property names and non-integer values in integer fields are refused. Optional fields are omitted; only fields explicitly declared nullable may be `null`. Support-group and premise input order is retained in events but has no effect on validity; conflict source-span order is retained; import proposal IDs follow item order; task-resolver items follow reference order. Other response collections sort by stable ID, except event arrays, which sort by sequence. A complete request is at most 32 MiB of UTF-8 JSON. A mutation or preview that would affect or return more than 10,000 groups, heads, edges, or task references refuses with `impact_too_large` before writing or returning a partial preview. An export snapshot is at most 64 MiB of canonical JSON; larger snapshots refuse with `export_too_large`.

## Primitive and shared field types

| Type | Wire form |
| --- | --- |
| Sequence | Unsigned JSON integer from 0 through 2^53−1; event sequences are globally increasing in one store and remain exactly representable by common JSON clients. |
| Text | UTF-8 string. `title` and `name`: 1–256 Unicode scalar values; `body`: 0–65,536 UTF-8 bytes; `scope`: 1–4,096 UTF-8 bytes when present; `reason`: 1–4,096 UTF-8 bytes. |
| ID | Opaque, service-generated string using the prefix and 26-character lowercase base32 suffix below. Clients compare or store IDs; they do not derive meaning from the suffix. |
| Digest | 64 lowercase hexadecimal characters encoding SHA-256. |
| Metadata | Object with at most 64 keys, each 1–64 characters and matching `[A-Za-z0-9_.-]+`; values are strings of at most 4,096 UTF-8 bytes, booleans, safe integers, or null. Nested values are refused. |
| Facets | Object with at most 64 keys, each 1–64 characters and matching `[a-z][a-z0-9_.-]*`; each value is an array of 1–64 strings, each at most 256 Unicode scalar values. |
| Cursor | Opaque non-empty string returned by a paginated read; valid only for that operation and query. |

| ID type | Required prefix | Grammar |
| --- | --- | --- |
| Node | `node_` | `^node_[a-z2-7]{26}$` |
| Revision | `rev_` | `^rev_[a-z2-7]{26}$` |
| Dimension | `dim_` | `^dim_[a-z2-7]{26}$` |
| Branch | `branch_` | `^branch_[a-z2-7]{26}$` |
| Proposal | `proposal_` | `^proposal_[a-z2-7]{26}$` |
| Import batch | `batch_` | `^batch_[a-z2-7]{26}$` |
| Root assertion | `root_` | `^root_[a-z2-7]{26}$` |
| Support group | `sg_` | `^sg_[a-z2-7]{26}$` |
| Edge | `edge_` | `^edge_[a-z2-7]{26}$` |
| Event | `event_` | `^event_[a-z2-7]{26}$` |
| Transaction | `txn_` | `^txn_[a-z2-7]{26}$` |
| Source reference | `source_` | `^source_[a-z2-7]{26}$` |
| Preview | `preview_` | `^preview_[a-z2-7]{26}$` |
| Conflict | `conflict_` | `^conflict_[a-z2-7]{26}$` |

`source_uri` is 1–8,192 UTF-8 bytes and must be an absolute URI. `locator` is 1–2,048 UTF-8 bytes. `query` is at most 4,096 UTF-8 bytes. `idempotency_key` is 1–128 UTF-8 bytes. Strings are compared as decoded Unicode scalar sequences; v1 does not silently normalize or case-fold them. A `SourceRefInput` is exactly `{source_uri,locator,digest?}`. `SupportGroupInput` is an array of 1–64 `RevisionRef` values; a proposal contains zero to 64 groups. `source_refs` contains zero to 64 values. `carry_forward_edge_ids` contains zero to 64 unique edge IDs.

The optional top-level `actor_label` is a 1–128-byte UTF-8 string. If omitted, the persisted event contains `actor_label: null`; `null` is not accepted in the request. The label records the caller's claim only and never authenticates or authorizes it.

The closed v1 kind set is `requirement`, `decision`, `wish`, `option`, `exception`, `argument`, `evidence`, `source`, `question`, and `risk`. The relation set and each relation's endpoint kinds are listed below.

### References and typed endpoints

`RevisionRef` is exactly `{ "dimension_id": DimensionID, "branch_id": BranchID, "revision_id": RevisionID }`. `SourceRefInput` is exactly `{ "source_uri": string, "locator": string, "digest"?: Digest }`; `source_uri` and `locator` are non-empty, and every normative accepted requirement, decision, or exception needs at least one source reference.

`Endpoint` is one of these closed forms:

```json
{ "kind": "revision", "id": "rev_<26 lowercase base32 characters>" }
{ "kind": "support_group", "id": "sg_<26 lowercase base32 characters>" }
{ "kind": "projection", "dimension_id": "dim_<26 lowercase base32 characters>", "branch_id": "branch_<26 lowercase base32 characters>", "node_id": "node_<26 lowercase base32 characters>" }
{ "kind": "task", "system": "non-empty string", "element_id": "opaque non-empty string", "dimension_id": "dim_<26 lowercase base32 characters>" }
```

A support-group input is an array of 1–64 `RevisionRef` values. A proposal accepts at most 64 groups and at most 64 premises per group; duplicate premises and empty groups are refused. A task element ID is opaque to the knowledge graph and is preserved byte-for-byte.

## Operation schemas

Every request has exactly `version`, `operation`, and `input`, plus optional top-level `actor_label`. Every input field is required unless marked optional or explicitly nullable in the table. An operation input rejects all undeclared fields. Every mutation includes a non-empty `reason`; read and preview operations do not. State-sensitive mutations include the expected-state field shown in this table. A successful mutation returns the committed `seq`; no operation reports success before its transaction commits.

| Operation | Input fields | Result fields |
| --- | --- | --- |
| `dimensions.create` | `name: Text`, `metadata?: Metadata`, `reason` | `dimension_id`, `branch_id`, `branch_name: "main"`, `status: "active"`, `seq` |
| `dimensions.get` | `dimension_id` | `dimension: DimensionView`, `active_branch: BranchView`, `as_of_seq` |
| `dimensions.update` | `dimension_id`, `expected_dimension_seq: Sequence`, `name?: Text`, `metadata?: Metadata`, `reason` | Updated `dimension`, `seq` |
| `dimensions.retire` | `dimension_id`, `expected_dimension_seq: Sequence`, `reason` | `dimension_id`, `status: "retired"`, `seq`; history is retained and future writes refuse with `dimension_retired` |
| `proposals.create` | `dimension_id`, `branch_id?`, `node_id?: NodeID`, `expected_head_revision_id?: RevisionID|null`, `based_on_revision_id?: RevisionID`, `kind`, `title: Text`, `body: string`, `scope?: string`, `facets?: Facets`, `source_refs?: SourceRefInput[]`, `support_groups?: SupportGroupInput[]`, `carry_forward_edge_ids?: EdgeID[]`, `reason` | `proposal_id`, `node_id`, `revision_id`, `based_on_revision_id: RevisionID|null`, `acceptance_state: "proposed"`, `current_head_revision_id: RevisionID|null`, `as_of_seq` |
| `projections.add` | `dimension_id`, `branch_id?`, `revision_id`, `expected_projection_state: "absent"`, `reason` | `proposal_id`, `node_id`, `revision_id`, `acceptance_state: "proposed"`, `current_head_revision_id: null`, `as_of_seq` |
| `support.impact_preview` | `proposal_id`, `dimension_id`, `branch_id` | `proposal_id`, `dimension_id`, `branch_id`, `preview_id`, `base_seq`, `expected_head_revision_id: RevisionID|null`, `expected_head_seq: Sequence`, `proposed_support_groups: SupportGroupInput[]`, `premise_states: PremiseState[]`, `support_groups_before_after: AffectedGroup[]`, `affected_heads: AffectedHead[]`, `task_refs: TaskRefView[]`, `carry_forward_edges: EdgeView[]` |
| `proposals.accept` | `proposal_id`, `dimension_id`, `branch_id`, `expected_head_revision_id: RevisionID|null`, `expected_head_seq: Sequence`, `preview_id`, `accept_as_root: boolean`, `reason` | `node_id`, `revision_id`, `head_status: "head"`, `acceptance_state: "accepted"`, `support_state: "supported"`, `root_assertion_id: RootID|null`, `support_group_ids[]`, `affected_groups[]`, `affected_heads[]`, `as_of_seq` |
| `proposals.reject` | `proposal_id`, `dimension_id`, `branch_id`, `expected_state: "proposed"`, `reason` | `revision_id`, `acceptance_state: "rejected"`, `head_revision_id: RevisionID|null`, `seq` |
| `roots.withdraw` | `root_assertion_id`, `dimension_id`, `branch_id`, `expected_state: "active"`, `expected_root_seq: Sequence`, `reason` | `root_assertion_id`, `revision_id`, `root_state: "withdrawn"`, `affected_groups[]`, `affected_heads[]`, `as_of_seq` |
| `support_groups.withdraw` | `support_group_id`, `dimension_id`, `branch_id`, `expected_group_seq: Sequence`, `reason` | `support_group_id`, `validity: "withdrawn"`, `affected_groups[]`, `affected_heads[]`, `as_of_seq` |
| `nodes.show` | Exactly one of `node_id: NodeID` or `revision_id: RevisionID`; `dimension_id`, `branch_id`, `as_of_seq?: Sequence` | `NodeShowResult` below |
| `revisions.delete` | `revision_id`, `dimension_id`, `branch_id`, `expected_head_revision_id`, `expected_head_seq: Sequence`, `reason` | `removed_from_projection_revision_id`, `restored_head_revision_id`, `head_status: "head"`, `acceptance_state`, `support_state`, `active_edges[]`, `affected_groups[]`, `affected_heads[]`, `as_of_seq` |
| `nodes.delete_preview` | `node_id`, `scope: "dimension"|"all_dimensions"`, `dimension_id?: DimensionID`, `branch_id?: BranchID` | `preview_id`, `base_seq`, `affected_heads[]`, `affected_edges[]`, `affected_task_refs[]` |
| `nodes.delete` | Same scope fields as preview, `preview_id`, `expected_seq: Sequence`, `reason` | `node_id`, `tombstone_scope: "projection_removed"|"logical_node_tombstoned"`, `affected_heads[]`, `affected_edges[]`, `affected_task_refs[]`, `as_of_seq` |
| `links.add` | `dimension_id`, `branch_id`, `from: Endpoint`, `relation`, `to: Endpoint`, `expected_graph_seq: Sequence`, `reason` | `edge_id`, `from`, `relation`, `to`, `as_of_seq` |
| `links.replace` | `dimension_id`, `branch_id`, `edge_id`, `to: Endpoint`, `expected_graph_seq: Sequence`, `reason` | `withdrawn_edge_id`, `replacement_edge_id`, `from`, `relation`, `to`, `as_of_seq` |
| `links.withdraw` | `dimension_id`, `branch_id`, `edge_id`, `expected_graph_seq: Sequence`, `reason` | `withdrawn_edge_id`, `as_of_seq` |
| `moves.preview` | `dimension_id`, `branch_id`, `node_id`, `new_parent_node_id: NodeID|null`, `expected_topology_seq: Sequence` | `preview_id`, `base_seq`, `topology_seq`, `subtree_node_ids[]`, `old_parent_node_id: NodeID|null`, `new_parent_node_id: NodeID|null`, `remove_edge_ids[]`, `proposed_edge: EdgeView|null` |
| `moves.commit` | `dimension_id`, `branch_id`, `preview_id`, `expected_topology_seq: Sequence`, `reason` | `node_id`, `subtree_node_ids[]`, `removed_edge_ids[]`, `added_edge_id: EdgeID|null`, `topology_seq`, `as_of_seq` |
| `search` | `dimension_id`, `branch_id`, `query?: string`, `node_ids?: NodeID[]`, `revision_ids?: RevisionID[]`, `kind?: Kind[]`, `facets?: Facets`, `include_unsupported?: boolean`, `include_proposed?: boolean`, `include_history?: boolean`, `limit: integer 1..100`, `cursor?: Cursor` | `items[]` of `SearchItem`, `next_cursor: Cursor|null`, `as_of_seq` |
| `history` | `node_id`, `dimension_id`, `branch_id`, `after_seq?: Sequence`, `limit: integer 1..100`, `cursor?: Cursor` | `events[]`, `next_cursor: Cursor|null`, `as_of_seq` |
| `task.resolve_refs` | `references[]` of 0–100 `{system,element_id,dimension_id,revision_id}`, `as_of_seq?: Sequence` | `items[]` in input order, each echoing the supplied ref and `status: "resolved"|"unknown"|"forbidden"`; only resolved items include `node_id,head_status,acceptance_state,support_state,as_of_seq,detail_url`; `as_of_seq` |
| `imports.stage` | `dimension_id`, `branch_id?`, `idempotency_key: string`, `source_uri: string`, `source_digest: Digest`, `items[]` of `{kind,title,body,scope?,facets?,locator,source_refs?,support_groups?}`, `reason` | `batch_id`, `proposal_ids[]`, `duplicate: boolean`, `as_of_seq` |
| `conflicts.stage` | `dimension_id`, `branch_id`, `left_revision_id`, `right_revision_id`, `evidence: ConflictEvidence`, `expected_graph_seq: Sequence`, `reason` | `conflict_id`, `edge_id`, `status: "candidate"`, `as_of_seq` |
| `conflicts.review` | `conflict_id`, `expected_status: "candidate"`, `decision: "confirmed"|"resolved"|"deferred"|"not_conflict"`, `reason`, `replacement_proposal_id?: ProposalID` | `conflict_id`, `status`, `resolution_event_id`, `contradiction_edge_id: EdgeID|null`, `as_of_seq`; support state is unchanged |
| `export` | `dimension_id`, `branch_id`, `as_of_seq?: Sequence`, `include_history: boolean` | `snapshot` matching `swarma-lemma-export/v1`, `digest: Digest` |

### Shared result records and operation rules

All result objects are closed: their fields are exactly the fields named here or in the operation table. An omitted optional field is absent; it is never replaced by `null` unless the field explicitly allows null.

| Result record | Exact fields and constraints |
| --- | --- |
| `ProposalView` | `proposal_id,dimension_id,branch_id,node_id,revision_id,state,created_seq,decided_seq?,support_groups: SupportGroupInput[],carry_forward_edge_ids: EdgeID[]`; `state` is `proposed`, `accepted`, or `rejected`; both arrays preserve the exact proposal inputs and are empty when none were supplied. |
| `NodeView` | `node_id,created_seq,tombstoned`; a tombstoned node is retained for history and is absent from default search. |
| `DimensionView` | `dimension_id,name,metadata,status,created_seq`; metadata is an object, including `{}` when empty. |
| `BranchView` | `branch_id,dimension_id,name,status,created_seq`; v1 has one active `main` branch per active dimension. |
| `RevisionView` | `revision_id,node_id,revision_no,based_on_revision_id,kind,title,body,facets,source_refs,created_seq`; nullable `based_on_revision_id` is explicit; optional `scope` is included when present and omitted when absent. |
| `SourceRefView` | `source_id,revision_id,source_uri,locator,digest?,captured_seq`; `digest` is omitted when unavailable. |
| `ProjectionView` | `dimension_id,branch_id,node_id,head_revision_id,created_seq,head_selected_seq,projection_state,support_state`; `head_revision_id` is null when no head is selected or the projection was removed; `projection_state` is `active` or `removed`; `support_state` is `supported` or `unsupported` when a head is selected, otherwise `not_applicable`; `head_selected_seq` is the latest head-selection sequence, or projection-creation sequence if no head was ever selected. |
| `SupportGroupView` | `support_group_id,dimension_id,branch_id,target_revision_id,premise_revision_refs,created_seq,withdrawn,withdrawn_seq?,validity`; refs are `RevisionRef`; `withdrawn_seq` is present exactly when `withdrawn` is true; `validity` is `valid`, `ineligible`, or `withdrawn`. |
| `EdgeView` | `edge_id,dimension_id,branch_id,from,relation,to,created_seq,withdrawn_seq?`; `from` and `to` are exact `Endpoint` values; `withdrawn_seq` is absent while active and present after explicit or system-managed withdrawal. |
| `AffectedGroup` | `support_group_id,target_revision_id,before_validity,after_validity`; each value is `valid`, `ineligible`, or `withdrawn`. |
| `AffectedHead` | `node_id,before_revision_id,after_revision_id,before_support_state,after_support_state`; revision IDs may be null; support state is `supported`, `unsupported`, or `not_applicable`. |
| `PremiseState` | `revision_ref,current_head_revision_id,support_state`; `revision_ref` is `RevisionRef`, current head may be null, and support state is `supported`, `unsupported`, or `not_applicable`. |
| `TaskRefView` | `system,element_id,dimension_id,revision_id`; the resolver echoes these fields byte-for-byte. |
| `ConflictView` | `conflict_id,dimension_id,branch_id,left_revision_id,right_revision_id,evidence,status,created_seq,resolution_event_id?`; status is `candidate`, `confirmed`, `resolved`, `deferred`, or `not_conflict`. |
| `ImportBatchView` | `batch_id,dimension_id,branch_id,idempotency_key,source_uri,source_digest,request_digest,proposal_ids,created_seq`; `request_digest` is SHA-256 of canonical JSON for the dimension, branch, source URI, source digest, and ordered canonical item array. |
| `HeadSelectionRecord` | `dimension_id,branch_id,node_id,from_revision_id,to_revision_id,seq,stack_action,stack_revision_id`; `from_revision_id` and `stack_revision_id` are explicit nullable fields; `stack_action` is `none`, `push`, or `pop`; `stack_revision_id` is null for `none`, identifies the prior non-null head for `push`, and the restored head for `pop`. |
| `SupportImpactPreview` | `proposal_id,dimension_id,branch_id,preview_id,base_seq,expected_head_revision_id,expected_head_seq,proposed_support_groups,premise_states,support_groups_before_after,affected_heads,task_refs,carry_forward_edges`; exact input and bound state consumed by proposal acceptance. |
| `MovePreview` | Exact `moves.preview` result fields from the operation table, including the exact proposed edge and assigned edge ID. |
| `NodeDeletePreview` | Exact `nodes.delete_preview` result fields from the operation table. The persisted resolved wrapper also stores `scope` and `node_id`. |
| `ConflictEvidence` | Exactly `{source_spans,subject,property,value,scope,conditions,summary}`. `source_spans` has 1–64 `{source_uri,locator,start_byte,end_byte}` records; byte ranges are half-open and non-empty. `subject`, `property`, `value`, and `summary` are non-empty text; `scope` and `conditions` are text or null. |
| `SearchItem` | `node_id,revision_id,head_status,acceptance_state,support_state,title,snippet,source_refs`; title follows the `Text` bound; snippet is at most 1,024 UTF-8 bytes; head status is `head`, `historical`, or `not_head`; acceptance state is `proposed`, `accepted`, or `rejected`; support state is `supported`, `unsupported`, or `not_applicable`. |
| `EventView` | The persisted event envelope and payload schema below; history returns events in ascending `seq`. |

`NodeShowResult` has exactly `{node,projection,revision,head_status,acceptance_state,support_state,source_refs,support_groups,edges,as_of_seq}`. `node` and `projection` are `NodeView` and `ProjectionView`; `revision` is `RevisionView|null`; `head_status` is `head`, `historical`, or `not_head`, and is null only when the projection has no selected head and the request used `node_id`; `acceptance_state` is `proposed`, `accepted`, or `rejected`, and is null only when `revision` is null. `support_state` is `supported`, `unsupported`, or `not_applicable`. At the effective read sequence, `acceptance_state` is the latest `RevisionDecision` state for that revision in the selected projection, while head, support, and edge fields are evaluated from that same sequence. For a `revision_id` request, `revision` is the requested revision and it must have existed in the selected dimension and branch by that sequence. For a `node_id` request, `revision` is the selected head or null. `source_refs` and `support_groups` describe that returned revision, or are empty arrays when `revision` is null. `edges` contains its eligible edges at `as_of_seq`.

For `task.resolve_refs`, every result item has the exact input reference fields plus `status`. A `resolved` item additionally has `node_id,head_status,acceptance_state,support_state,as_of_seq,detail_url`; an `unknown` or `forbidden` item has no additional fields. `detail_url` is an absolute URI. `source_refs` in revision and search views are arrays of `SourceRefView`.

`dimensions.get` returns the dimension's active `main` branch even after retirement so its history remains readable. Retirement blocks every subsequent mutation in the dimension but does not erase or hide reads. `dimensions.update` replaces the metadata object when `metadata` is supplied; it does not merge keys. It requires at least one mutable field and refuses a no-op value change with `no_change`. `moves.commit` refuses a move to the current parent with `no_change`; an exact same-target `links.replace` does likewise. These refusals write no event.

`proposals.create` creates one new immutable revision and a pending proposal. If `node_id` is absent, it creates a node and empty projection; `expected_head_revision_id` and `based_on_revision_id` must be omitted. If `node_id` is supplied, that node must have an active projection in the selected dimension and branch, and `expected_head_revision_id` must equal its current head, including null for an empty projection. If `based_on_revision_id` is omitted, it defaults to that expected head. If supplied, it must identify an earlier revision of the same node; it may be historical and need not be the current head. `expected_head_revision_id` still guards acceptance against concurrent head changes. `projections.add` instead stages an already existing revision of the given node into a dimension and branch where that node has never previously been projected; it never copies or edits revision content. A projection removed by `nodes.delete` cannot be added again in v1. A projection has at most one pending proposal for a given revision. A rejected revision cannot be proposed again in that projection. `dimensions.update` and `links.replace` no-op rules are defined above.

`proposals.accept` requires `accept_as_root: true` exactly when the proposal has no pending support groups. For a derived acceptance, it must be false and the proposal must contain at least one complete support group. It accepts the revision and selects it as head atomically. The preview token must match proposal, dimension, branch, expected head and sequence; it expires after any relevant head or support change. Accepting a proposal for an empty projection uses `expected_head_revision_id: null` and the projection's current `head_selected_seq` as `expected_head_seq`.

`support_groups.withdraw` permanently withdraws one accepted support group. It recomputes support in the same dimension and branch. A group made merely ineligible by a premise head change may become valid again if rollback restores that exact premise; a withdrawn group never becomes valid again.

Support state is the least fixed point over current selected heads: begin with heads having an active root assertion, then add a head when one complete, non-withdrawn group targets that head and has only already-supported exact premises. A group is `ineligible` while its target is not selected or any premise is not a current supported head. Heads outside the closure are unsupported. V1 refuses every cycle in accepted support groups, including a cycle that also has a path to a root.

`revisions.delete` is a deletion-only rollback of the selected head in one projection. For each `(node_id, dimension_id, branch_id)`, committed selection events derive a prior-head stack: `proposals.accept` pushes the current non-null head, and `revisions.delete` pops and selects the top entry. The first accepted head has no predecessor, so deleting it returns `no_predecessor`. Proposal creation, rejection, link changes, and support-only changes do not affect the stack. The `revision_id` must equal `expected_head_revision_id`; the sequence must also match the current head-selection event. Rollback does not erase the revision, its decision, sources, support records, or edges. It recomputes support and eligible edge views at the new sequence; no link snapshot is restored and no edge is copied, retargeted, or reactivated. A restored unsupported head is allowed. If the deleted head's `based_on_revision_id` differs from the stack's top entry, the stack entry wins. No predecessor, a non-head target, or a stale sequence refuses without a write.

`links.add`, `links.replace`, and `links.withdraw` accept only ordinary relation types. They cannot write `depends_on`, `justifies`, `contains`, `conflict_candidate`, or `contradicts`: support-group acceptance generates the first two, `moves.commit` owns containment, and conflict staging/review owns the last two. At most one active edge with the same dimension, branch, endpoints and relation may exist. Replacing an edge preserves its source endpoint and relation and atomically withdraws the old edge and adds a new edge with the supplied target. Withdrawal requires the edge to be active. Neither operation creates a node revision.

`moves.preview` binds the dimension, branch, moved node, old parent, proposed parent, exact descendant set, removed edge, topology sequence and proposed edge. The topology sequence is the latest committed change to a `contains` edge in that dimension and branch. `moves.commit` requires the matching preview and unchanged topology sequence. It changes only the moved root's parent edge; descendants remain attached to that root. A move to null detaches the root. It refuses a malformed/shared-parent forest, a parent outside the selected projection, a cycle, or a stale preview without writes. Direct containment-link mutations are refused.

`nodes.delete_preview` and `nodes.delete` require `dimension_id` and `branch_id` only for `scope: "dimension"`; both are forbidden for `scope: "all_dimensions"`. The preview binds scope, node, exact affected heads, edges, task references and base sequence. `nodes.delete` is a projection removal for dimension scope and a logical-node tombstone for all-dimensions scope. Both retain immutable records and history; neither is physical erasure. Projection removal and tombstoning make citing support groups ineligible and recompute support in every affected dimension and branch without synthetic revisions. Edges remain in history and are omitted from current eligible-edge views while an endpoint is no longer projected. A removed projection cannot be re-added in v1. An all-dimensions tombstone hides the node from every current view but does not erase any revision or historical projection.

`conflicts.stage` requires two distinct accepted or proposed revisions in the selected dimension and branch and stores the supplied evidence unchanged. It creates an advisory candidate and a `conflict_candidate` edge; it cannot alter support. `conflicts.review` is allowed only from `candidate`. `resolved` requires `replacement_proposal_id`; the other decisions forbid that field. `confirmed` appends a `contradicts` edge and returns its ID; other decisions return `contradiction_edge_id: null`. Resolving a candidate records the decision only: accepting a replacement or exception remains a separate proposal transaction. A candidate is never inferred from a detector result without an explicit staged record.

Search filters combine by intersection. `node_ids`, `revision_ids`, and `kind` are each unique arrays of 1–100 values. For each requested facet key, every requested facet value must be present on a result. `include_unsupported`, `include_proposed`, and `include_history` default to false and add those categories; `include_history` includes accepted non-head revisions, while rejected records appear only when `include_proposed` is true. Search never returns the same revision twice. A cursor is bound to the exact operation, filters, dimension, branch, and pinned `as_of_seq`; reusing it with any different query returns `invalid_cursor`.

The task resolver preserves input order and duplicates. `status: "unknown"` means the exact reference has no match; `status: "forbidden"` discloses no graph details; an unavailable adapter refuses the whole operation with `operation_unavailable`. A successful empty result is not used to represent either refusal. Task references never mutate task state.

When `branch_id` is omitted on an operation that accepts it, the active `main` branch is selected. V1 refuses if no active branch exists. The `support.impact_preview` token binds the proposal, dimension, branch, expected head ID and selection sequence, proposed support groups, current premise head/support states, and explicit copy-forward edge IDs and states. `base_seq` records the preview's observation point; unrelated events do not stale the token if every bound input remains unchanged. `proposals.accept` refuses a missing, stale, or mismatched token; preview and acceptance do not partially write.

## Relation endpoint and write rules

| Relation | `from` kind | `to` kind | Eligibility |
| --- | --- | --- | --- |
| `depends_on` | support_group | revision | The target is a current supported head in the same dimension and branch at acceptance. |
| `justifies` | support_group | revision | The target is the proposal revision accepted in the same transaction. |
| `contains` | projection | projection | Both are active projections in the same dimension and branch; child has no other parent; adding the edge creates no cycle. |
| `conflict_candidate` | revision | revision | Both IDs exist in the selected dimension and branch and are proposed or accepted. |
| `contradicts` | revision | revision | Both are accepted revisions in the selected dimension and branch; historical endpoints are allowed. |
| `supports`, `objects_to`, `related_to` | revision | revision | Both IDs are accepted revisions in the selected dimension and branch; no support propagation. |
| `refines`, `narrows` | revision | revision | Source is accepted; target is an accepted exact revision in the same dimension and branch. No support propagation. |
| `supersedes` | revision | revision | Source is the current accepted head; target is an earlier accepted revision of the same logical node in the same dimension and branch. |
| `regrows_from` | revision | revision | Source is the current accepted supported head; target is the immediately previous selected head of the same node in the same dimension and branch, and its source acceptance event recorded that target's pre-acceptance support as `unsupported`. |
| `task_ref` | task | revision | Exact task element and revision reference; does not mutate task state. |
| `module_depends_on` | projection | projection | Active projections in the same dimension and branch; architecture query only, no support propagation. Cycles are permitted. |

The v1 relation vocabulary is closed to the rows in this table. `contradicts` is symmetric when read; its stored endpoint order is canonical by revision ID. A support-group endpoint is valid only for the group that generated it. `task_ref` endpoints require an exact task element ID and dimension. `contains` and `module_depends_on` endpoints must resolve to active node projections in the selected dimension and branch. The `links.*` operations may write only `supports`, `objects_to`, `related_to`, `refines`, `narrows`, `supersedes`, `regrows_from`, `task_ref`, and `module_depends_on`.

Non-propagating revision relations may point to accepted historical revisions in the selected dimension and branch. `depends_on` alone requires current supported premise heads; `justifies` targets the revision being accepted; `contains` and `module_depends_on` target active projections. `supersedes` must link two revisions of the same logical node, with a current accepted source and an earlier accepted target. `regrows_from` is a historical explanation, not a support edge. Its source must be the current accepted supported head; its target must be the immediately previous selected head for the same node, dimension, and branch; and the source acceptance event's `AffectedHead.before_support_state` for that target must be `unsupported`. This event evidence remains authoritative even though the target's current historical support is `not_applicable`. A never-unsupported target, a target that recovered before replacement, a non-immediate target, a cross-dimension or cross-branch target, or a second repair anchor from the same source is refused with `invalid_repair_anchor`. A valid source revision is counted once; later withdrawal of its anchor edge does not reduce the historical count. `task_ref` preserves the exact revision, including historical revisions. No edge silently changes its endpoint when a revision is created or becomes head.

Link-only operations append edge events and do not create node revisions. A replace is one atomic withdrawal plus one new edge. Content edits create a new proposed revision; no edge is copied to it unless its proposal lists the exact `carry_forward_edge_ids`. Each ID must identify an active ordinary edge in the same dimension and branch incident to the proposal's current head. At acceptance, the copied edge gets a new ID and event; every endpoint equal to the prior head is replaced by the new revision, and all other endpoints remain exact. The new edge must satisfy the endpoint rules. System-managed support and containment edges cannot be copied.

An exact revision read defaults to its acceptance sequence when that revision has been accepted in the selected dimension and branch. For a never-accepted revision, it defaults to its latest `RevisionDecision` sequence: proposal creation while proposed, or rejection after rejection. At an explicit cutoff, the read uses the latest decision, head selection, support state, and eligible-edge state at or before that sequence. A cutoff before the revision or its projection existed returns `not_present_at_sequence` without a partial result. A later `as_of_seq` shows later edge events separately; it never folds them into an earlier accepted revision's default snapshot. Support is `supported` or `unsupported` only while the revision is the selected head; proposed, rejected, and historical revisions report `not_applicable`.

## Refusal codes and state guarantees

Every refusal has exactly one code from this table. Unless stated otherwise, a refusal writes no event, edge, head, support projection, task reference, or partial import.

| Condition | Code | Result |
| --- | --- | --- |
| Malformed JSON or duplicate object keys | `invalid_json` | No write. |
| Version other than 1 | `unsupported_version` | No write. |
| Unknown operation or property | `unknown_operation` or `unknown_field` | No write; no unknown-field ignore mode in v1. |
| Missing required field, wrong JSON type, invalid enum, or duplicate entry in an array declared unique | `invalid_request` | No write. |
| ID does not match its type grammar | `invalid_id` | No write. |
| Caller lacks read/write authority | `forbidden` | No object-existence details and no write. |
| Authorized caller names a missing ID | `not_found` | No write. |
| Dimension is retired | `dimension_retired` | No write. |
| ID exists but is not projected into the selected dimension/branch | `dimension_mismatch` | No write. |
| An exact revision or its selected projection did not yet exist at the requested cutoff | `not_present_at_sequence` | No result fields or partial historical view. |
| A node projection already exists for `projections.add` | `already_projected` | No write. |
| A node or revision is tombstoned | `tombstoned_node` | No write. |
| Branch is absent or inactive | `branch_not_active` | No write. |
| Expected head, graph sequence, or topology sequence differs | `stale_head`, `stale_graph`, or `stale_preview` respectively | No write; caller must read or preview again. |
| Preview token is missing, expired, or does not bind the exact request | `impact_preview_required` or `stale_preview` | No write. |
| Proposal is no longer pending, or expected root/group state differs | `proposal_not_pending` or `stale_state` | No write. |
| Root assertion is absent or already withdrawn | `root_not_active` | No write. |
| Caller targets a non-head revision for rollback | `revision_not_current_head` | No write. |
| Selected revision has no prior selected head to restore | `no_predecessor` | No write. |
| Caller attempts to restore a removed projection or make a historical revision a head directly in that projection | `immutable_history` | No write; create a new revision in an active projection or stage the revision in a dimension where it has never been projected. |
| A derived proposal has no complete support group or root assertion | `ungrounded_revision` | No write. |
| Root and support-group acceptance modes are mixed, or a root has groups | `invalid_support_mode` | No write. |
| A premise is not a current accepted supported head | `unsupported_reference` | No write. |
| A support group repeats a premise, references its target node, or has invalid cardinality | `invalid_support_group` | No write. |
| Accepted support edges form any cycle | `dependency_cycle` | No write. |
| Child projection already has a parent | `containment_parent_conflict` | No write. |
| Move would create a containment cycle | `containment_cycle` | No write. |
| Existing containment is malformed or has shared parents | `invalid_containment_topology` | No write. |
| Link is already active, its relation is system-managed, or its endpoint kind is invalid | `duplicate_edge` or `unsupported_relation_endpoint` | No write. |
| `regrows_from` does not meet the temporal repair-anchor predicate | `invalid_repair_anchor` | No write; support and the historical repair count are unchanged. |
| Node deletion has no matching impact preview or the preview is stale | `impact_preview_required` or `stale_preview` | No write. |
| Search/history cursor is malformed or bound to another query | `invalid_cursor` | No write. |
| Requested `as_of_seq` is greater than the current sequence | `sequence_in_future` | No write. |
| Request exceeds the input byte limit | `request_too_large` | No write. |
| A preview or mutation would affect more than 10,000 groups, heads, edges, task references, or committed records | `impact_too_large` | No write; narrow the operation or query. |
| The fully encoded committed event frame exceeds 64 MiB | `event_too_large` | No write. |
| Mutation would have no state effect | `no_change` | No write. |
| Export would exceed the canonical snapshot size limit | `export_too_large` | No partial snapshot. |
| Import key was used with different source bytes or content | `idempotency_conflict` | No write. |
| Import item lacks required source digest or locator | `incomplete_source` | No write. |
| Conflict is not a candidate or the decision lacks its required replacement proposal | `conflict_not_reviewable` or `invalid_conflict_resolution` | No write. |
| Optional task adapter is not enabled by counterpart agreement | `operation_unavailable` | No result items; stored task references remain unchanged. |
| Store is unavailable, times out, or fails its transaction | `unavailable` or `storage_error` | No successful empty result and no partial writes. |

Error precedence is: JSON parse and duplicate keys; protocol version; envelope shape and unknown fields; field types and ID grammar; authentication/authorization; authorized object existence; `as_of_seq` future check; explicit historical-presence validation at `as_of_seq`; dimension/branch eligibility at the effective sequence; expected-state tokens; semantic validation; transaction. This order prevents an unauthorized read from learning whether an ID exists and prevents partial historical views. All validation completes before a transaction commits. A storage error rolls back the full transaction.

An unknown task reference is an item with `status: "unknown"`; a denied reference is `status: "forbidden"` with no graph details. A successful empty resolver response is `{"version":1,"ok":true,"result":{"items":[],"as_of_seq":N}}`. An unavailable adapter returns the error envelope and never claims an empty result. All three cases preserve the stored task reference and task status.

## Persisted events and exports

The API version is separate from the persisted event schema version. An event has exactly `{schema_version,seq,id,transaction_id,transaction_index,transaction_size,type,dimension_id,branch_id,payload,at,actor_label,reason}`. `schema_version` is integer `1`; `transaction_index` is zero-based; `transaction_size` is `1` in v1; `dimension_id` and `branch_id` are nullable only for global-scope `nodes.delete` events; `actor_label` is null or a 1–128-byte UTF-8 string. `at` is an RFC 3339 UTC timestamp with at most six fractional-second digits. Actor labels are descriptive only, not authentication. `reason` is required and follows the shared text bound.

V1 emits exactly one event for each successful mutating request. Its `type` is `operation.committed`, `transaction_index` is `0`, and `transaction_size` is `1`. Its exact `payload` is `{operation,input,resolved,records,result}`. `operation` is the mutating operation name; `input` is the exact validated input object; `resolved` is null for an operation that consumes no server-resolved preview, or the exact typed snapshot consumed by `proposals.accept`, `moves.commit`, or `nodes.delete`; and `result` is the operation result with transport-only `seq` and `as_of_seq` fields removed. A successful idempotent import duplicate creates no event.

`resolved` is null or one of these closed forms: `{kind:"support_impact",snapshot:SupportImpactPreview}`, `{kind:"move",snapshot:MovePreview}`, or `{kind:"node_delete",scope,node_id,snapshot:NodeDeletePreview}`. `SupportImpactPreview` is exactly the `support.impact_preview` result in the operation table, including the proposed support groups, premise states, affected groups and heads, task references, and exact carry-forward edge states. `MovePreview` is exactly the `moves.preview` result, including its destination, old parent, descendant set, topology sequence, removed edge IDs, and proposed edge with its assigned ID. `NodeDeletePreview` is exactly the `nodes.delete_preview` result; the wrapper preserves its `scope` and `node_id`. These snapshots are copied into the commit event when the preview token is consumed; replay never looks up a preview cache.

`records` is a non-empty array of `{kind,value}` tagged records. Its closed `kind` set is `node`, `dimension`, `branch`, `projection`, `revision`, `proposal`, `revision_decision`, `source_ref`, `root_assertion`, `support_group`, `edge`, `conflict`, `import_batch`, and `head_selection`. Each `value` is the corresponding complete post-state record: `NodeView`, `DimensionView`, `BranchView`, `ProjectionView`, `RevisionView`, `ProposalView`, `{dimension_id,branch_id,revision_id,state,reason?,event_id,seq}`, `SourceRefView`, `{root_assertion_id,dimension_id,branch_id,revision_id,state,reason,created_seq,withdrawn_seq?}`, `SupportGroupView`, `EdgeView`, `ConflictView`, `ImportBatchView`, or `HeadSelectionRecord`. The list contains every persistent record created or changed by this operation, including all generated IDs, updated projections and derived support states; it contains no unchanged records. Each entity occurs at most once in the list. The v1 schema defines the exact required record kinds and keys for every operation; replay rejects missing, extra, duplicate, or result-inconsistent records. Records sort by `kind` and then their type's primary ID, with `head_selection` records sorted by `(dimension_id,branch_id,node_id,seq)`. A record's created, decision, selection, or withdrawal sequence is the enclosing event's `seq` when that event creates that fact.

The operation-to-record plan is closed. Each row lists records created or directly changed; an operation also includes every `projection`, `support_group`, or `edge` record whose materialized post-state changes as a consequence of support or eligibility recomputation.

| Operation | Direct records |
| --- | --- |
| `dimensions.create` | `dimension`, `branch` |
| `dimensions.update`, `dimensions.retire` | `dimension` |
| `proposals.create` | `node` and `projection` only when creating a node; `revision`, `proposal`, `revision_decision`, and each `source_ref` |
| `projections.add` | `projection`, `proposal`, `revision_decision` |
| `proposals.accept` | `projection`, `revision_decision`, `head_selection`; `root_assertion` for root acceptance, otherwise each created `support_group` and all of its generated `edge` records; also each copied `edge` |
| `proposals.reject` | `proposal`, `revision_decision` |
| `roots.withdraw` | `root_assertion` |
| `support_groups.withdraw` | `support_group` |
| `revisions.delete` | `projection`, `head_selection` |
| `nodes.delete` | `node` for an all-dimensions tombstone; the selected `projection` for dimension removal |
| `links.add`, `links.withdraw` | The created or withdrawn `edge` |
| `links.replace` | The withdrawn edge's post-state and the new `edge` |
| `moves.commit` | The removed edge's post-state and the proposed `edge`, if a parent is selected |
| `imports.stage` | `import_batch`; for each item, `node` and `projection` when newly created, plus `revision`, `proposal`, `revision_decision`, and every `source_ref` |
| `conflicts.stage` | `conflict`, `edge` |
| `conflicts.review` | `conflict`; `edge` only for a confirmed contradiction |

IDs named by a direct record plan must match the operation input, resolved preview, and result. Derived records must match the affected keys and exact post-state captured by the committed event; they are not regenerated by rerunning the operation.

`HeadSelectionRecord.stack_action` is `push` when acceptance places a non-null prior head on the rollback stack, `pop` when rollback restores and removes the exact top entry, and `none` when acceptance selects a first head with no prior non-null head. Its `from_revision_id` is the previous selected head and its `to_revision_id` is the newly selected or restored head. The record makes rollback-stack replay explicit; a first selected head does not create a null stack entry.

`ImportBatchView.request_digest` is SHA-256 over canonical JSON with exactly `{dimension_id,branch_id,source_uri,source_digest,items}`; `items` is the exact ordered canonical item array from the validated request. The batch record preserves the idempotency key, request digest, and original generated proposal IDs. Replay rebuilds the idempotency index from the committed `import_batch` record and does not allocate new IDs.

The store commits the event and all listed record changes atomically. Replay from an empty store validates each complete event schema, sequence, resolved snapshot, record kind and value, primary IDs, and operation/result/record consistency, then applies the recorded post-state values atomically in sequence. It does not re-run the operation, resolve external data, read a preview cache, consult current time, or allocate IDs. Any materialized index not represented as a durable record is rebuilt deterministically from the applied records and event order. The event file uses 4-byte unsigned big-endian length-prefixed UTF-8 JSON frames, with a maximum JSON payload of 64 MiB per frame; a mutation whose canonical event JSON exceeds that bound refuses with `event_too_large` and writes nothing. Replay applies only complete frames with valid schemas and strictly increasing sequences. A truncated final frame is ignored and reported at its first sequence; a malformed complete frame, sequence gap, invalid size, or malformed non-final data fails closed at the first affected sequence. Earlier complete events remain readable.

The deterministic export envelope is:

```json
{
  "schema": "swarma-lemma-export/v1",
  "dimension_id": "dim_<26 lowercase base32 characters>",
  "branch_id": "branch_<26 lowercase base32 characters>",
  "as_of_seq": 42,
  "nodes": [],
  "projections": [],
  "revisions": [],
  "proposals": [],
  "revision_decisions": [],
  "root_assertions": [],
  "support_groups": [],
  "edges": [],
  "conflicts": [],
  "import_batches": [],
  "events": []
}
```

The snapshot has exactly the envelope fields shown. `nodes` contain `NodeView`; `projections` contain `ProjectionView`; `revisions` contain `RevisionView`; `proposals` contain `ProposalView`; `revision_decisions` contain `{dimension_id,branch_id,revision_id,state,reason?,event_id,seq}`; `root_assertions` contain `{root_assertion_id,dimension_id,branch_id,revision_id,state,reason,created_seq,withdrawn_seq?}`; `support_groups` contain `SupportGroupView`; `edges` contain `EdgeView`; `conflicts` contain `ConflictView`; `import_batches` contain `ImportBatchView`; `events` contain `EventView`. Without history, records contain the selected projection, its current supported or unsupported head revisions, pending proposals, candidate conflicts, active roots and support groups relevant to those heads, and currently active eligible edges; `import_batches` and `events` are empty. With `include_history: true`, arrays contain all records up to `as_of_seq` that affect the selected dimension and branch, including global tombstones that affect a selected projection and import batches with proposals in that projection. Records sort by type then ID; events sort by sequence. `export` returns `{snapshot,digest}` where digest is lowercase SHA-256 of the snapshot's canonical UTF-8 JSON bytes. Canonical JSON uses UTF-8, object keys in bytewise ascending order, no insignificant whitespace, shortest valid JSON string escapes, and base-10 integers without leading zeros. Private source contents are not embedded; source references retain URI, locator, and digest.

## Import and conflict handling

`imports.stage` accepts 1–500 items and creates proposals only in the selected dimension and branch. Each item is exactly `{kind,title,body,scope?,facets?,locator,source_refs?,support_groups?}`; no per-item unknown fields are accepted. The batch source reference is added to each item with its item locator and batch digest. The same idempotency key with the same dimension, branch, source URI, digest, and canonical item array returns the original batch and proposal IDs with `duplicate: true` and creates no event; reuse with any different value returns `idempotency_conflict`. Item order is significant for the idempotency comparison and is preserved in `proposal_ids`. No parser output becomes current knowledge automatically.

Conflict detection is advisory. A candidate records exact source spans, normalized subject/property/value, overlapping scope and conditions, and a short reason. `conflicts.review` records a human decision and reason. Accepting a resolution or scoped exception is a separate proposal and acceptance transaction. Detector failure is not an empty candidate set and no detector result changes support state.

## Task-system seam

`task.resolve_refs` is a proposed optional read-only seam, not available for implementation until the task-system owner agrees to the `references[]` fields and access-control mapping. It never writes task notes, status, or references, shares no database, and imports no task-board internals. For a resolved item, the exact echoed reference is accompanied by `node_id,head_status,acceptance_state,support_state,as_of_seq,detail_url`; `detail_url` is an absolute URI and appears only when authorized. Unknown, forbidden, and unavailable outcomes follow the exact shapes above.
