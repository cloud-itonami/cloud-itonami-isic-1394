# cloud-itonami-isic-1394: Manufacture of cordage, rope, twine and netting

Open Business Blueprint for **ISIC Rev.5 1394**: manufacture of cordage, rope, twine and netting — an autonomous "actor" (LLM advisor behind an independent Governor, langgraph-clj StateGraph, append-only audit ledger) that coordinates back-office cordage/rope/twine/netting-plant **operations**: production-batch data logging (fibre-twisting/braiding/winding output, length, and output quality via breaking-strength testing), fibre-twisting-machine/braiding-machine/winding-line maintenance scheduling, equipment-safety/quality-defect concern flagging, and outbound cordage/rope/twine/netting shipment coordination.

This repository designs a forkable OSS business for cordage/rope/twine/
netting-plant operations: run by a qualified operator so a plant keeps its
own operating records instead of renting a closed SaaS.

## What this actor does

Proposes **plant operations coordination**, not machine operation:
- `:log-production-batch` — fibre-twisting/braiding/winding batch, length, and output-quality (breaking-strength test) data logging (administrative, not an operational decision)
- `:schedule-maintenance` — fibre-twisting-machine, braiding-machine, or winding-line maintenance scheduling proposal
- `:flag-safety-concern` — surface an equipment-safety/quality-defect concern (always escalates)
- `:coordinate-shipment` — outbound cordage/rope/twine/netting shipment coordination proposal

## What this actor does NOT do

**CRITICAL SCOPE BOUNDARY** (fibre-twisting machines, braiding machines, winding lines; materials-handling and equipment-safety hazards):

- Does NOT control fibre-twisting, braiding, or winding-line equipment directly
- Does NOT make plant-safety, labor-safety, or materials-safety decisions (that's the plant supervisor's exclusive human authority)
- Does NOT directly operate twisting/braiding/winding-line equipment under any proposal (permanently blocked, see Architecture)
- ONLY proposes/coordinates operations back-office; all actuation requires explicit human approval
- Safety-concern flagging ALWAYS escalates — never auto-decided, no confidence threshold or phase below escalation

## Architecture

Classic governed-actor pattern (`cordageops.operation/build`, a langgraph-clj StateGraph):
1. **`cordageops.advisor`** (sealed intelligence node, `CordageAdvisor`): proposes decisions only, never commits
2. **`cordageops.governor`** (independent, `Cordage & Netting Plant Operations Governor`): validates against domain rules, re-derived from `cordageops.registry`'s pure functions and `cordageops.store`'s SSoT -- never trusts the advisor's own self-report
   - HARD invariants (always `:hold`, no override):
     - Plant/batch record must be independently verified/registered (`:verified?` AND `:registered?`) before any action is taken against it (equipment before maintenance scheduling, batch before shipment coordination)
     - The request's own `:effect` must be `:propose` (never a direct-write bypass)
     - `:op` must be in the closed four-op allowlist
     - The proposal's own `:effect` must be one of the four propose-shaped effects (no direct twisting/braiding/winding-line-equipment control)
     - Directly operating twisting/braiding/winding-line equipment (`:direct-operate? true`) is a PERMANENT, unconditional block
     - A shipment may not push a batch's own recorded shipped length past its own logged production length (independently recomputed)
     - No double-scheduling the same maintenance record
     - No fabricated `:quality-grade` value on a production-batch patch
     - No physically implausible `:breaking-strength-kn` value on a production-batch patch
     - No physically implausible `:defect-rate-percent` value on a production-batch patch
   - ESCALATE (always human sign-off, overridable by a human):
     - `:flag-safety-concern` always escalates, regardless of confidence
     - Low-confidence proposals
3. **`cordageops.phase`** (Phase 0->3 rollout): `:schedule-maintenance`/`:flag-safety-concern`/`:coordinate-shipment` are NEVER in any phase's `:auto` set (permanent, matching the governor's own posture); only `:log-production-batch` may auto-commit at phase 3 when clean
4. **`cordageops.store`** (append-only audit ledger + SSoT): a single `MemStore` backend behind a `Store` protocol (see ns docstring for why a second Datomic-backed backend is out of scope for this build)

## Development

```bash
# Run tests (top-level deps.edn already pins langgraph+langchain local/root)
kbb -M:test

# Run tests via the workspace :dev override alias (equivalent, kept for sibling-repo parity)
kbb -M:dev:test

# Run the demo
kbb -M:dev:run

# Regenerate docs/samples/operator-console.html by driving the REAL actor
# (every id/number/status on that page is actual operation -> governor ->
# store output; two consecutive runs are byte-identical)
kbb -M:dev:render-html

# Lint
kbb -M:lint
```

## Status

`:implemented` — `governor.cljc`/`store.cljc`/`advisor.cljc`/`registry.cljc` + `deps.edn` complete the module set; tests green, demo runnable, langgraph-clj integration verified.

## License

AGPL-3.0-or-later
