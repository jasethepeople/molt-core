# MOLT-CORE specification — assumptions and open decisions

Date: 2026-11-12

## Source basis

The specification was prepared from the supplied ZIP:
- `molt-verify-l1/` (including `verifier.mjs`, `molt-canonical.mjs`, README, tests, and `samples/sample-good/`)
- `molt-event/`
- `molt-verify-ledger/`
- Foundry reference files in `source/`
- `gateway-gate/REPORT.md`
- `molt-anchor/README.md`

The Camp 1 rules in the project brief and verified L1 implementation were treated as normative where the supplied Camp 2 tools conflict.

## Assumptions made

1. **Manifest field names:** Used the L1 verifier's concrete schema: `ledger_bytes`, `agent_pubkeys`, `artifacts`, and `artifact_bytes`, with optional `anchor`. This is more specific than the abbreviated manifest description in the project brief.
2. **Package verification key:** Treated root-level `ed25519-public.pem` as a required L1 package control file. The L1 verifier parses it as Ed25519 and requires its raw key to match each event's `pub` and the manifest's actor mapping.
3. **Artifact digest semantics:** Interpreted each `artifacts[path]` value as lowercase SHA-256 hex of the exact artifact file bytes, with `artifact_bytes[path]` as the exact byte count.
4. **Event integer constraints:** Described `seq` and `ts` as non-negative JSON integers, matching the L1 verifier's `Number.isInteger` checks. Did not upgrade this to a safe-integer constraint because the event verifier does not explicitly use `Number.isSafeInteger`.
5. **Payload domain:** Used the brief's rule that payload may be any JSON value. The L1 canonicalizer supports JSON primitives, arrays, and objects. Foundry's TypeScript event type is narrower in its declared shape (`Record<string, unknown>`), so implementations should confirm that Foundry's writer accepts all JSON root types before relying on them in production.
6. **L1 limits:** Included the verifier's implementation limits (1 MB manifest, 32 MiB ledger, 256 KiB per event line) as L1 package-verifier limits, not as fundamental cryptographic limits.
7. **Local manifest anchor:** Described the L1 manifest anchor as `{seq, hash}` because that is what the L1 verifier actually checks. Described the external `molt-anchor` checkpoint separately as `(ledger_id, seq, head_hash)` because that is what the anchor README specifies.
8. **Room pointer:** Identified the Room frozen contract as a separate specification/implementation, but did not invent its rules because no Room contract was included in the supplied ZIP.
9. **Per-caller permissions:** The gateway report documents the approval gate and its caller/target/event bindings, but does not establish a complete general permission model. The spec therefore points to gateway authorization rather than claiming the report defines all per-caller permissions.

## Open decisions explicitly flagged in the spec

1. **Non-ASCII canonicalization:** Foundry escapes characters above U+007E and sorts object keys by UTF-8 bytes. The L1 `canonicalizeJsonText` path is identified as RFC 8785/JCS, emits non-ASCII UTF-8, and uses JavaScript's UTF-16 key ordering. These behaviors can diverge. A maintainer ruling and cross-implementation test vectors are required before claiming non-ASCII interoperability.
2. **Key lifecycle:** Key rotation, revocation, and recovery are not defined by the supplied event, manifest, or verifier contracts. A separate key-management specification is needed.
3. **Room contract details:** The Room contract was not supplied, so only its out-of-scope pointer is included.
4. **Independent anchor witness:** The anchor README says a second witness is needed for a real guarantee. The exact witness protocol, retention policy, timestamping authority, and dispute/recovery procedure are not defined by the supplied materials.

## Known incompatibilities / required follow-up

- The supplied Camp 2 `molt-event` and `molt-verify-ledger` materials still specify the incompatible seven-field body and signature over canonical body bytes. They must be changed to the Camp 1 five-field body and signature over the 32 raw hash bytes before they are considered interoperable.
- The supplied `molt-anchor` README explicitly says its event schema is a guess and does not match MOLT's nine-field event. Its event hash routine must be adapted to Camp 1 before it is used on real MOLT-CORE ledgers.
- The L1 README identifies `canonicalizeJsonText` as RFC 8785, but the Foundry implementation differs for non-ASCII values and key sorting. The spec leaves this unresolved rather than silently selecting a behavior.

## Verification performed for this deliverable

Executed the supplied L1 CLI against `samples/sample-good/`:
- Result: `PASS all checks`
- Anchor: `verified`
- Events: `2`

Executed the supplied `run-tests.mjs`:
- `sample-good`: passed
- `tampered-payload`: expected V-09 hash mismatch
- `tampered-broken-hash-chain`: expected V-09 chain broken
- `tampered-bad-signature`: expected V-09 bad signature
- `tampered-wrong-sequence`: expected V-09 sequence failure
- `tampered-non-canonical-line`: expected V-09 canonicality failure
- `tampered-missing-field`: expected V-09 field-set failure
- `tampered-bad-hash-format`: expected V-09 field-format failure
- All sample expectations passed.

These checks validate the supplied L1 sample harness only. They are not a new independent implementation, a full conformance suite, or a production security audit.
