# CLI and JSON contract

Draft for review, 2026-11-14. Names and examples are illustrative; no CLI is implemented.

## Operations

The CLI and JSON API expose the same versioned operations:

| Operation | Purpose |
| --- | --- |
| search | Search accepted, supported head revisions by default; optionally include unsupported heads, proposals, or historical revisions. |
| show | Read a logical node or exact revision, its head status, acceptance state, support state, source references, revision-pinned links, and history. |
| dimensions | Create, read, update, and delete (retire) dimensions as data; a new dimension does not require a database schema migration. |
| projection | Create, read, update, or remove a logical node's representation in one dimension, selecting or withdrawing its head revision while preserving projection history. |
| links | Create, read, replace, and withdraw dimension-scoped links pinned to exact revisions. |
| move | Reparent a node and its structural descendants in one dimension, recording the old and new topology. |
| delete-revision | Tombstone the latest revision in one dimension and restore its predecessor there, retaining the deleted revision and links in history. |
| delete-node | Tombstone the logical node across dimensions, recording scope, reason, and affected heads and links. |
| ancestors | Follow support groups from a node to their required premises. |
| dependents | Follow reverse depends_on edges to affected support groups and target nodes. |
| impact | Preview which groups would be withdrawn and which head revisions would lose their final valid support. |
| propose | Create a proposal revision with source references and typed support groups over exact revisions. |
| review-conflict | Confirm, resolve, defer, or mark a conflict candidate `not_conflict`, recording the human decision. |
| accept | Human operator accepts a proposal or a newly created replacement revision and records a reason. |
| reject | Human operator rejects a proposal and records a reason. |
| history | Read revisions, revision-pinned links, events, and branches for a node in a selected dimension. |
| resolve-task-ref | Resolve task references to exact revision IDs in a selected dimension and return support state and a detail link. |
| export | Export a bounded, versioned snapshot of a selected dimension and its revision-pinned links. |

Illustrative commands:

    lemma search --dimension spec --query "current requirement" --support-state supported --json
    lemma show <node-id> --dimension spec --json
    lemma show <node-id> --revision <revision-id> --dimension spec --json
    lemma show <node-id> --revision <revision-id> --dimension spec --as-of-seq <seq> --json
    lemma dependents <node-id> --dimension spec --depth all --json
    lemma impact <node-id> --dimension spec --json
    lemma dimension create implementation --json
    lemma projection add <node-id> --dimension implementation --json
    lemma link add <from-revision-id> --type contains --to <revision-id> --dimension implementation --json
    lemma move <node-id> --dimension implementation --to <parent-node-id> --json
    lemma propose --dimension spec --kind decision --title "..." --source "uri#heading" --json
    lemma history <node-id> --dimension spec --json
    lemma delete-revision <revision-id> --dimension spec --reason "..." --json
    lemma projection remove <node-id> --dimension implementation --reason "..." --json
    lemma delete-node <node-id> --reason "..." --json
    lemma export --dimension spec --as-of-seq <seq> --json
    lemma review-conflict <candidate-id> --dimension spec --decision not_conflict --reason "..." --json
    lemma accept <proposal-id> --dimension spec --reason "..." --json

Graph reads and writes are scoped to one explicit dimension unless the operation explicitly targets the whole logical node. Agent-facing use is read and propose; dimension administration, revision deletion, node deletion, and acceptance/rejection remain local operator actions. Node creation is proposal plus acceptance; updating content or relations creates a new immutable revision and head in the selected dimension. Deleting the latest revision restores the prior head; removing a projection deletes the node only from that dimension; deleting the logical node tombstones it across dimensions. These are auditable events, not physical erasure. If a latest revision has no predecessor, `delete-revision` refuses and the operator must choose dimension-only or whole-node deletion. In the first release these commands record an actor label but cannot authenticate it; access control must come from the trusted local environment.

Dimension-scoped operations use an explicit branch ID when supplied; otherwise v1 resolves them against the sole active branch in that dimension and refuses if no active branch exists.

## JSON envelope

Every request has a protocol version and operation. Every response has the same version and either a result or a typed error. The API protocol version is distinct from the persisted event schema version.

    {
      "version": 1,
      "operation": "show",
      "input": {"node_id": "logical-node-id", "dimension_id": "spec"}
    }

    {
      "version": 1,
      "ok": true,
      "result": {
        "node_id": "logical-node-id",
        "revision_id": "revision-id",
        "revision_no": 1,
        "kind": "requirement",
        "dimension_id": "spec",
        "branch_id": "branch-id",
        "head_status": "head",
        "acceptance_state": "accepted",
        "support_state": "supported",
        "as_of_seq": 10,
        "valid_support_group_count": 2,
        "sources": [],
        "support_groups": [],
        "relations": [],
        "event_ids": []
      }
    }

Errors use stable codes, a short message, and optional exact IDs:

    {
      "version": 1,
      "ok": false,
      "error": {
        "code": "dependency_cycle",
        "message": "A support group must be grounded in an accepted root.",
        "ids": ["node-a", "node-b"]
      }
    }

Unknown fields may be ignored only when the protocol version explicitly allows them. Unknown versions and relation kinds are refused. Do not make clients parse prose for status.

## Refusal and mutation rules

| Condition | Result |
| --- | --- |
| Unknown operation, version, node kind, relation, or malformed identifier | invalid_argument or unsupported_version; no write. |
| Logical node ID or exact revision ID does not exist | not_found; no write. |
| Dimension ID does not exist or is retired | not_found or dimension_retired; no write. |
| A link or support group refers to a logical node absent from the selected dimension | dimension_mismatch; no write. |
| Caller tries to retarget a historical link to a new revision without an explicit copy-forward operation | revision_link_retarget; no write; preserve the original exact endpoints. |
| Caller tries to delete a non-head revision by deleting only that revision | historical_revision; no write. Use whole-node deletion only when that is the intended scope. |
| Caller tries to delete the latest revision when no predecessor exists | no_predecessor; no write. Choose dimension-only or whole-node deletion explicitly. |
| Caller deletes a logical node with affected relationships | Require a dimension-aware impact preview; tombstone the node and recalculate affected support projections atomically. |
| A task reference names an unknown revision or dimension | Return `unknown` for that reference; do not treat it as supported or drop the stored reference. |
| Caller lacks permission to read a referenced node | forbidden; do not reveal node content or change the task reference. |
| Knowledge service is unavailable or times out | unavailable; keep the task reference and report no successful resolution or empty impact result. |
| New support group would form an unsupported cycle | dependency_cycle; no write. |
| Move or re-link preview is stale | stale_preview; no write; recompute against the current dimension topology. |
| Detector finds a possible conflict | Record a `candidate` with exact source spans, normalized comparison, and overlapping scope; revision heads and support groups remain unchanged. |
| Candidate is confirmed, resolved, deferred, or marked `not_conflict` | Require an explicit human review event and reason. Any accepted knowledge change is a separate new-revision proposal; never rewrite either original revision. |
| New support group targets an unsupported head in the selected dimension or uses an unsupported premise revision | unsupported_reference; no write. History-only relations may still point to prior revisions. |
| Caller tries to make a historical revision the head again | immutable_history; create a new revision with current premises and a `regrows_from` history relation to the old revision. |
| Caller tries to edit an immutable revision | immutable_revision; create a new revision based on the old one instead. |
| Impact preview is stale at commit time | stale_preview; recompute before acceptance. |
| Storage transaction fails | storage_error; do not publish partial events or a changed current projection. |
| Import source lacks a stable locator or digest when available | incomplete_source; retain as an unaccepted draft or refuse. |
| Caller requests physical deletion of an immutable revision or referenced dimension | immutable_history or dimension_in_use; use the requested auditable revision, logical-node, projection, or dimension-retirement operation instead. Physical erasure is not supported by this contract. |

An impact preview includes the selected dimension, exact old/new revision IDs, every support group that would cease to be valid, each target’s valid-group count before and after, nodes that would remain supported through an alternative group, nodes that would become unsupported, branch IDs, and task references. Acceptance must bind to the preview's source revision and current branch. Any mismatch refuses without partial writes.

Only depends_on relations within accepted support groups propagate support changes inside the selected dimension. If any required premise revision changes or is withdrawn, that support group becomes unsupported. The head revision stays supported if another complete group remains valid; otherwise it becomes unsupported and dependent groups are recalculated. The directly invalidated node receives a successor revision and the predecessor is retained as historical with an invalidation cause. Whether every dependent needs a same-content successor revision or only an event plus unsupported projection remains open. Candidate conflicts, confirmed contradictions, rationale links, containment, refines/narrows links, task references, and module dependencies never propagate support changes. No invalidation propagates across dimensions implicitly. A circular chain without an accepted root remains unresolved.

The open fanout question concerns dependents that lose their final support without a content edit of their own: each may receive a same-content successor revision, or only an event plus a derived unsupported state.

Conflict detection is advisory. A candidate includes the exact source spans, normalized subject/property/value comparison, overlapping scope and conditions, and a short reason. A human may confirm it, resolve it by accepting a new revision or scoped exception, defer it, or mark it not_conflict. No detector result changes support state by itself. Detector failure or missing evidence is not a successful empty conflict result.

Repair never reactivates a historical revision as head. Create a new proposal revision, cite supported premise revisions, and link it to the historical anchor with `regrows_from`; accept it to make it the selected head. The old revision and its links remain unchanged.

## Search, history, and task-board seam

Accepted, supported-head lexical search is the default within an explicit dimension. Add explicit flags for unsupported heads, proposals, and historical revisions. The response includes logical node IDs, revision IDs, head status, acceptance state, support state, support-group count, snippets, and revision-pinned source references. Search ranking is advisory and must not be presented as a conflict verdict.

Use event sequence for “what did the system believe at this point?” Add valid-time queries only if the product later needs “when did this claim apply in the world?” The optional board seam proposes a list-shaped `knowledge_refs` field containing exact revision IDs paired with dimensions. The resolver returns each reference with its logical node ID, exact revision, dimension, head status (`head`, `historical`, `deleted`, or `unknown`), acceptance state (`proposed`, `accepted`, or `rejected`), support state (`supported`, `unsupported`, or `not_applicable`), graph `as_of_seq`, a detail link, and relevant cause/successor IDs. The board remains authoritative for task status; an optional adapter uses the read-only resolver and does not share a database, import board internals, or rewrite task state. A denied read returns `forbidden`; a service timeout or outage returns `unavailable`, distinct from a successful empty result. It is read-only: it must not add notes, change task status, or rewrite references. The access-control mapping between tasks and knowledge nodes remains to be agreed with the task-system owner.

Historical revision reads default to the revision's accepted event sequence and return only links that existed then. An explicit `--as-of-seq` returns a later graph snapshot, with subsequent links represented as their own events and exact endpoint revisions; they are not folded into the historical revision view. History includes each event's schema version and transaction ID; replayed mutations apply atomically. A new revision has no implicitly retargeted links; copying an old link forward creates a new versioned edge with exact endpoints and an auditable event.

## Import

Import Markdown or decision records only as staged proposals. Preserve source URI, content digest, and source span for every extracted item. A batch import must be reviewable and repeatable; it must not promote extracted text into current knowledge automatically. Do not copy private source contents into public exports.
