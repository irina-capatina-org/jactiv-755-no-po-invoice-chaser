# Build Notes — JACTIV-755 No-PO Invoice Chaser

## Plan

1. Read skills (uipath-api-workflow, uipath-platform, uipath-solution) and design docs
2. `uip solution init no-po-invoice-chaser-755` from `code/`
3. `uip api-workflow init no-po-invoice-chaser-api` inside solution dir (auto-registers in .uipx)
4. Extract reference Workflow.json from `docs/architectural-considerations.md` §4.5 via awk
5. Write `bindings_v2.json` with both connection bindings (Coupa + Slack)
6. Write solution connection resource files and process resource file
7. Run `validate-build.sh` gate; fix any errors
8. Run `uip solution pack` to prove deployability

## Summary

Single API Workflow project inside a solution. Queries Coupa for no-PO invoices (draft/new, last 7 days, credit notes excluded), sends Slack Block Kit DM when count > 0, silent on clean day.

## Task Status

| Task | Project | Status | Notes |
|---|---|---|---|
| Solution scaffold | no-po-invoice-chaser-755 | done | uip solution init |
| API Workflow project | no-po-invoice-chaser-api | done | uip api-workflow init, auto-registered |
| Workflow.json | no-po-invoice-chaser-api | done | Extracted from §4.5 reference; 19 activities |
| bindings_v2.json | no-po-invoice-chaser-api | done | 2 connection bindings |
| Connection resource files | Solution/ | done | Coupa + Slack |
| Process resource file | Solution/ | done | no-po-invoice-chaser-api.json |
| validate-build.sh | — | done | passed, 19 activities |
| solution pack | — | done | no-po-invoice-chaser-755_0.0.1.zip |

## Deviations from the SDD

None. Workflow started from the proven §4.5 reference. SDD variable names, activity keys, connection IDs, Slack member ID, and Coupa query params all match exactly.

## Left for a human

- OQ-01: BR-08 prohibits retry; image2.png shows retry nodes — SME must confirm which governs (no retry implemented per BR-08)
- OQ-02: BR-07 says silent clean day; PDD §7 mentions congratulatory message — SME must confirm (silent implemented per BR-07)
- OQ-05: Confirm `invoice-lines[].po-number` and `invoice-lines[].order-header-num` are authoritative PO linkage fields (SDD §4)
- OQ-06: Verify `slack-product-test-app` has `chat:write` and DM access to `WLX9BD8FN`
- OQ-09: Production Coupa base URL needed before PROD go-live (test URL `uipath-test.coupahost.com` is used)
- Orchestrator schedule: set to Europe/Bucharest, weekday-only, 10:00 (deploy prerequisite)
- `coupa-uipath-test` ping shows Failed/403 — confirmed working at runtime per §4 note; do not re-provision

## How to Test

```bash
# Static validation
cd code/no-po-invoice-chaser-755/no-po-invoice-chaser-api
/usr/local/n/versions/node/22.23.2/bin/node /usr/local/bin/uip api-workflow validate Workflow.json --output json

# Build gate
cd <repo-root>
bash .github/scripts/validate-build.sh code docs/architectural-considerations.md

# Pack (deploy readiness)
/usr/local/n/versions/node/22.23.2/bin/node /usr/local/bin/uip solution pack \
  code/no-po-invoice-chaser-755 /tmp/buildcheck \
  --name no-po-invoice-chaser-755 --version 0.0.1 --output json
```
