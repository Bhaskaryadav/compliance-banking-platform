# Sanctions Screening — Insertion Point & Design

## 1. Pipeline Position

**MCP query result** (`get_compliance_pipeline` via `banking-api` MCP server):

```json
[
  {"step": 1, "name": "kyc_check",    "description": "Verify customer identity"},
  {"step": 2, "name": "aml_analysis", "description": "Anti-money laundering screening"},
  {"step": 3, "name": "approval",     "description": "Final compliance approval"}
]
```

**Insertion point:** between step 2 (`aml_analysis`) and step 3 (`approval`).

Updated pipeline: `kyc_check → aml_analysis → [NEW: sanctions_screening] → approval`

**Rationale:** Sanctions screening must run after AML analysis (which establishes a risk
score and flags suspicious patterns) but before approval (which requires a clean
compliance record). At this position the transaction has already been identity-verified
and risk-scored; sanctions screening adds the OFAC/UN list check as the final gate
before any approval decision is issued. Placing it here ensures that high-risk AML
flags and sanctions hits are both visible at the approval step without either check
silencing the other.

---

## 2. Data Inputs Required

| Field | Type | Source | Notes |
|---|---|---|---|
| `customer_id` | `str` | Transaction payload / banking-api | Non-empty string; used for audit logging (masked in logs per PCI DSS Req 3) |
| `counterparty_details` | `dict` | Transaction payload | Must contain at least `"name"` key; may include `country`, account/IBAN |
| `transaction_amount` | `float` | Transaction payload | Non-negative; used for risk-weighting, not for list-matching logic |

---

## 3. Output Schema

```json
{
  "screened": true,
  "match_list": [
    {
      "list_source": "OFAC",
      "matched_name": "Acme Trading Ltd",
      "score": 0.91
    }
  ],
  "confidence": 0.91
}
```

| Field | Type | Description |
|---|---|---|
| `screened` | `bool` | `true` if the screening step executed successfully; `false` on API failure or invalid input |
| `match_list` | `list[dict]` | Zero or more OFAC/UN matches, each with `list_source`, `matched_name`, `score` |
| `confidence` | `float` | `max(score)` across all matches; `0.0` when `match_list` is empty |

---

## 4. Integration Contract with the AML Module

- **Input from AML module:** The `aml_result` (status, risk_score) passes through to the approval step unchanged. Sanctions screening runs independently — neither a clean AML result clears a sanctions hit nor a dirty AML flag auto-fails sanctions.
- **Output to Approval step:** Any non-empty `match_list` with `confidence >= 0.75` is treated as a **hard stop** requiring manual compliance review before approval proceeds.
- **Failure mode:** If the sanctions API is unreachable or input validation fails, `screened` must be `false` and the transaction routed to manual review — it must **never** silently pass.
- **Audit trail:** All screening results are stored via `_store_result_encrypted()`, satisfying PCI DSS Req 3/4 (protection of stored and transmitted data). `customer_id` is masked in logs (`cust[:2]***[:-2]`).

---

## 5. Open Questions / Risks

| Item | Detail |
|---|---|
| False-positive rate | Fuzzy name matching may flag legitimate counterparties with similar names; a similarity threshold of 0.75 (configurable) reduces noise |
| Name-matching fuzziness | Current stub uses exact-lowercase match; production should use phonetic/edit-distance algorithm |
| List refresh cadence | OFAC/UN lists update daily; production integration must poll the API at least every 24 h and cache locally for resilience |
| Dual-hit precedence | When both OFAC and UN match the same counterparty, `confidence` takes `max(score)` — the stricter list's score wins |
