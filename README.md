# CharityAudit

A donor transparency audit for charities, as a GenLayer Intelligent Contract.

## The problem

A charity can claim anything on its website — "85% of donations go to programs" — and a donor has no cheap way to check it. The annual report is public, the registry listing is public, but nobody reads them side by side, and a single reader's word is not proof.

## How it works

An audit owner (a donor group, a watchdog, or the charity itself) opens an audit for one charity and declares three evidence sources up front:

- `report_url` — the charity's published annual report (the numbers under test)
- `registry_url` — the public charity registry listing
- `corrections_url` — the charity's published corrections / errata page (optional)

Anyone except the owner can submit a transparency claim against the audit: a statement that something in the charity's public claims does not match its own published material, with a claimed severity (critical / high / medium / low).

When a claim is assessed, every validator fetches the report, the registry listing, and the corrections page itself and judges the claim against that fetched material only:

1. **Grounded** — a claim is never valid on the claimant's word alone. If the report or registry listing cannot be retrieved, or neither mentions the alleged discrepancy, the claim is invalid.
2. **Duplicate** — a claim matching the published corrections page, or a prior *accepted* finding on the same audit, is a duplicate and invalid. A match against a *rejected* claim does not count: the same allegation was already rejected on its own merits, and a repeat must be judged again on its own merits.
3. **Strict verdict** — `valid` must be an actual JSON boolean (a string `"true"` or a `1` is a malformed verdict and the claim is rejected). A valid claim always carries a real severity bucket; an invalid claim always carries `"none"`. A valid verdict with no usable severity is rejected, not rewarded.

## Deterministic outcome: the trust score

Every audit starts at a trust score of 100. An upheld claim reduces it by a fixed penalty that depends only on the ruled severity bucket:

| Severity | Penalty |
| --- | --- |
| critical | 40 |
| high | 25 |
| medium | 15 |
| low | 5 |

The score is computed on-chain from the bucket, never from free-form text, and is readable at any time via `get_audit_score`. Each verdict stores a one-sentence evidence note on-chain naming the fetched material that supports or rejects the claim, so any donor can audit the audit.

## Demo (see `test_flow.md`)

- Audit 0: Helping Hands Foundation, claims 85% program spend. Its own report says 62%.
  - Claim 0 (ratio overstated) → **valid / high, penalty 25**, score 100 → 75.
  - Claim 1 (charity not registered) → **invalid**: the registry lists it Active.
  - Claim 2 (admin costs hidden, FY2024) → **invalid**: already disclosed on the corrections page.
  - Claim 4 (repeat of rejected claim 1) → **invalid** on its own grounding, not called a duplicate.
  - Claim 5 (repeat of accepted claim 0) → **invalid** as a duplicate of the accepted finding.
- Audit 1: same charity, unreachable report URL → claim 3 **invalid** (no corroborating material).

Final state: 2 audits, 6 claims, trust score 75. 10 negative tests all rejected as expected.

## Verification

Deployed and verified in GenLayer Studio on 2026-11-09: contract [0x931264E6Deb36684E693cDc7C6E975e6e228f46e](https://explorer-studio.genlayer.com/address/0x931264E6Deb36684E693cDc7C6E975e6e228f46e), 24 transactions (16 SUCCESS, 8 intentional ERRORs from the negative tests). Claim 0 valid/high (penalty 25, score 100 → 75); claims 1–5 invalid/none/0 — including a repeat of the rejected claim ruled invalid on its own grounding (not called a duplicate) and a repeat of the accepted claim ruled invalid as a duplicate. The severity invariant holds on all six claims. 8 negative tests rejected as expected. See `test_flow.md` for the full run.

Known boundary: submitting a claim to a nonexistent audit fails with a raw `KeyError` instead of a friendly message (the TreeMap lookup raises before any check). The submission is still rejected, so the behavior is correct; the message is just unfriendly.

## Layout

- `contracts/charity_audit.py` — the contract
- `test_flow.md` — the full Studio verification flow, step by step
- `examples/` — the demo fixtures (annual report, registry listing, corrections page) and the sample inputs used in the flow
