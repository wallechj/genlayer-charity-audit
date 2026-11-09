# CharityAudit — Studio verification flow

Run this flow top to bottom in GenLayer Studio. Two test accounts: A (audit owner / deployer) and B (claimant). Check every write with a view read-back; a transaction that is FINALIZED and Consensus Accepted is not enough — the GenVM Result column must say SUCCESS for the intended calls, and the intended failures must show GenVM ERROR.

Fixtures (public, on `main`, fetchable as raw text before starting):
- `examples/charity_report.md` — Helping Hands Foundation FY2025 report: 62% program spend, 28% admin, 10% fundraising.
- `examples/charity_registry.md` — registry listing: Registered Charity No. 12345, Status: Active.
- `examples/charity_corrections.md` — corrections page: FY2024 admin cost restated from 12% to 18% (2026-03-01).

## 0. Deploy

Deploy `contracts/charity_audit.py` from account A. The deploy transaction must show GenVM Result SUCCESS. Note the contract address.

## 1. Create audits (account A)

Create audit 0 from `examples/sample_audit.json`:
- charity_name "Helping Hands Foundation", claimed_ratio 85, report_url `examples/charity_report.md`, registry_url `examples/charity_registry.md`, corrections_url `examples/charity_corrections.md`.
- Read back `get_audit(0)`: owner is A, `trust_score` 100, `status` "open", `claimed_ratio` 85.

Negative tests (each must fail with the exact message):
- Empty `report_url` → `report_url is required`.
- `report_url` "not-a-url" → `report_url must be an http(s) URL`.
- `claimed_ratio` 0 → `claimed_ratio must be between 1 and 100`.

Create audit 1 from `examples/sample_audit_dead.json`: same charity, `report_url` `https://example.invalid/helping-hands/report.md`, `corrections_url` empty. `get_audit_count()` must be 2.

## 2. Submit claims (account B)

Submit, in order, from B:
- Claim 0 from `examples/sample_claim_ratio.json` (audit 0): "Program-spend ratio overstated" — the charity claims 85% of donations go to programs, but its published FY2025 report states only 62%.
- Claim 1 from `examples/sample_claim_unregistered.json` (audit 0): "Charity not registered".
- Claim 2 from `examples/sample_claim_admin.json` (audit 0): "Administrative costs hidden" (FY2024).
- Claim 3 from `examples/sample_claim_ratio_audit1.json` (audit 1): same ratio claim, against the dead-report audit.
- Claim 4 from `examples/sample_claim_repeat_rejected.json` (audit 0): title and statement identical to claim 1.
- Claim 5 from `examples/sample_claim_repeat_accepted.json` (audit 0): title and statement identical to claim 0.

Negative tests (each must fail with the exact message):
- Account A (the audit owner) submits a claim → `audit owner cannot submit claims`.
- `claimed_severity` "urgent" → `claimed_severity must be one of critical/high/medium/low`.
- Submit to audit 999 → error (audit does not exist).

`get_claim_count()` must be 6.

## 3. Assess claims 0–3 (any account)

Call `assess_claim` on claims 0–3, then read each back with `get_claim`, `get_claim_severity`, `get_claim_penalty`, `get_claim_evidence`. Expected:

- Claim 0 → **valid / high / penalty 25**. Evidence must cite the fetched report's 62% program-spend figure against the claimed 85%.
- Claim 1 → **invalid / none / 0**. Evidence must cite the registry listing: Status Active, Registered Charity No. 12345.
- Claim 2 → **invalid / none / 0**. Evidence must cite the corrections page: the FY2024 admin restatement (12% → 18%) was already published.
- Claim 3 → **invalid / none / 0**. Evidence must state the annual report could not be retrieved, so there is no corroborating material.

After claim 0, `get_audit_score(0)` must be 75 (100 − 25).

## 4. Duplicate authority (any account)

- Assess claim 4 (repeat of rejected claim 1) → **invalid / none / 0**. Its evidence must judge the claim on its own grounding — the registry lists the charity Active — and must **not** call it a duplicate of claim 1. Claim 1 was rejected, so it is not duplicate authority.
- Assess claim 5 (repeat of accepted claim 0) → **invalid / none / 0**. Its evidence must call it a duplicate of the prior accepted finding (claim 0, valid/high).

## 5. Severity invariant

Read back all six claims: the only valid claim (0) carries a real bucket (high); every invalid claim (1–5) carries "none". `get_audit_score(0)` is still 75.

## 6. Lifecycle

- Account B closes audit 0 → error `only the audit owner can do this`.
- Account A closes audit 0 → SUCCESS; `get_audit(0)` shows `status` "closed".
- Account B submits a claim to closed audit 0 → error `audit is not open for claims`.

Leave audit 0 closed and audit 1 open.

## 7. Final read-back

- `get_audit_count()` = 2
- `get_claim_count()` = 6
- `get_audit_score(0)` = 75
- Audit 0 status "closed", audit 1 status "open"
