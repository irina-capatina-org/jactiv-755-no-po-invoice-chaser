# PDD - No-PO Invoice Chaser

## Document History

| Date | Version | Author | Role | Comments |
|---|---|---|---|---|
| 2026-09-23 | 0.1 | uipath-analyst | Analyst | Initial analysis from request-work/request-details.md (No-PO invoice finder v1.0, 16 Sep 2026) |

## 1. Document Control

| Field | Value |
|---|---|
| Document | PDD - No-PO Invoice Chaser |
| Story key | JACTIV-755 |
| Epic key | [SME REVIEW] |
| Source file | request-work/request-details.md |
| Branch | analysis-jactiv-755 |
| Author | uipath-analyst |
| Status | Draft - pending SME approval |
| Version | 0.1 |

## 2. Introduction

| Field | Value |
|---|---|
| Process name | No-PO Invoice Chaser |
| Process full name | NoPoInvoiceChaser |
| Business objective | Replace daily manual Coupa review with a fully automated weekday control that detects invoices lacking a properly linked PO and notifies the AP responsible via Slack |
| Owning department | Accounts Payable (Finance) |

| Role | Name / Contact |
|---|---|
| SME / Process Owner | Irina Capatina (irina.capatina@uipath.com, Slack ID WLX9BD8FN) |
| BA | uipath-analyst |
| Developer | [SME REVIEW] |

## 3. Process Overview

| Field | Value |
|---|---|
| Process full name | NoPoInvoiceChaser |
| Function and department | Accounts Payable, Finance |
| Short description | Automated weekday control that queries Coupa for invoices with no properly linked PO (status draft or new, past 7 days, credit notes excluded), counts qualifying records, and sends one Slack Block Kit message to the AP SME; sends nothing on a clean day |
| Required roles | Automation (unattended); SME receives Slack notification |
| Trigger and schedule | Weekday schedule at 10:00 Romania time (EET/EEST) |
| Volume (items per day) | ~1 run/day; live sample showed 194 qualifying invoices in a 7-day window |
| Average handling time | Manual: ~daily ad-hoc effort (inconsistent timing); Automated target: <2 min per run [SME REVIEW] |
| FTE effort | [SME REVIEW] |
| Estimated exception rate | Low – data is structured and rules are deterministic; credit-note and description-only-PO exclusions are the primary exception paths |
| Input data | Coupa invoice list (invoice date, status, PO linkage, invoice type) |
| Output data | Slack Block Kit DM to SME with qualifying invoice count and filtered Coupa URL; no output on clean day |

## 4. To-Be Process (High Level)

The automation is a short linear sequence: **trigger → query → filter → count → notify**. No human review is retained in the normal path and no Coupa record is modified.

- **Trigger:** UiPath Orchestrator weekday schedule fires at 10:00 Romania time.
- **Query:** Automation calls Coupa API for invoices dated within the past seven days with status `draft` or `new`, excluding credit notes.
- **Filter:** Automation discards invoices where a PO is properly linked; records where a PO number appears only in the description field are treated as missing.
- **Count:** Qualifying invoices are counted. If count = 0, the run ends silently (clean day).
- **Notify:** One Slack Block Kit direct message is sent to Irina Capatina (Slack ID WLX9BD8FN) carrying the count, an explanation of the no-PO-no-pay policy impact, a request to resolve, and a link to the filtered Coupa invoice list.

**Manual steps eliminated:** manual Coupa login and filtering, manual field copying, manual grouping by requester, manual Slack message drafting and sending.

## 5. Detailed Process Steps

| Step | Action | Application | Expected Result | Remarks |
|---|---|---|---|---|
| 1.1 | Weekday schedule fires at 10:00 Romania time (EET/EEST) | Orchestrator | Run context initialised; run date and timezone confirmed | Trigger only fires Monday–Friday |
| 1.2 | Calculate invoice date window: window_end = today; window_start = today − 7 days | Automation | window_start and window_end values ready for query and for message placeholders | See BR-04 |
| 2.1 | Call Coupa API: retrieve invoices where invoice_date >= window_start AND invoice_date <= window_end AND status IN (draft, new) | Coupa | Raw invoice list returned | Access method [SME REVIEW] – API or UI. See BR-04 |
| 2.2 | **Decision:** Is the Coupa response valid (HTTP 200 / non-empty parseable result)? | Coupa | Yes → step 2.3; No → step S1 path | See BR-08; diagram (image2.png) shows retry before failure – see OQ-01 |
| 2.3 | Discard records whose invoice type is credit note | Automation | Credit notes removed from working set | See BR-03 |
| 2.4 | Iterate over remaining records — **START LOOP** | Automation | Per-record evaluation begins | |
| 2.5 | **Decision:** Does the invoice have a properly linked PO (PO field on invoice header/line is populated with a valid PO reference)? | Coupa | Yes → step 2.6 (skip); No → step 2.7 | PO number typed into description field only does not qualify. See BR-01, BR-02 |
| 2.6 | Skip invoice (PO present and linked) | Automation | Record excluded from count | |
| 2.7 | Add invoice to qualifying list; increment count | Automation | Qualifying invoice counted | |
| 2.8 | **END LOOP** – all records evaluated | Automation | qualifying_count and coupa_url ready | coupa_url = Coupa invoice list filtered to same window_start/window_end and status=draft |
| 3.1 | **Decision:** Is qualifying_count > 0? | Automation | Yes → step 3.2; No → step 3.5 | See BR-07 |
| 3.2 | Build Coupa URL: `https://uipath-test.coupahost.com/invoices?q%5Binvoice_date_gteq%5D=<window_start>&q%5Binvoice_date_lteq%5D=<window_end>&q%5Bstatus_eq%5D=draft` | Automation | coupa_url placeholder resolved | URL filters to same date window as the run |
| 3.3 | Compose Slack Block Kit payload: title `:receipt: {{invoice_count}} invoices need a purchase order`; body lines `:warning:`, `:no_entry:`, `:point_right:`; primary button "Open the list in Coupa" linking to coupa_url; footer with window_start, window_end, run_date | Automation | Block Kit JSON payload ready | Full payload JSON to be recorded in SDD per source section 4.3. Count appears in title AND first body line. See BR-05, BR-06 |
| 3.4 | Send Slack Block Kit DM to Slack member ID WLX9BD8FN | Slack | Message delivered to Irina Capatina | Recipient addressed by Slack member ID, not email. See BR-06 |
| 3.5 | **Clean day** – qualifying_count = 0; send nothing; end run normally | Automation | Run completes; no Slack message sent | See BR-07. Note: source section 7 states a congratulatory message is sent on clean day – contradicts BR-07 and scope table. See OQ-02 |
| 4.1 | Log run outcome (qualifying_count, run_date, status: success or failed) | Orchestrator | Run result recorded | Allows clean result to be distinguished from technical failure |

## 6. Applications and Systems

| Application | Interface type | Access method | Login method | Credential handling | Comments |
|---|---|---|---|---|---|
| Coupa | API [SME REVIEW] | REST API [SME REVIEW] | API key or OAuth [SME REVIEW] | Orchestrator credential asset [DEFAULT] | Base URL: uipath-test.coupahost.com; read-only access required; PO linkage is on invoice lines not header |
| Slack | API | HTTP POST to Slack API (Block Kit) | Bot token | Orchestrator credential asset [DEFAULT] | Recipient: Slack member ID WLX9BD8FN; DM only; message format is Block Kit JSON not plain text |
| UiPath Orchestrator | Scheduler | Orchestrator trigger | Service account [DEFAULT] | Managed by Orchestrator [DEFAULT] | Weekday schedule at 10:00 Romania time |

## 7. Business Rules

| ID | Rule | Source | Applies at step |
|---|---|---|---|
| BR-01 | Invoices without a properly linked purchase order are subject to the no-PO-no-pay policy and must be reported | BR-001 | 2.5 |
| BR-02 | A PO number typed into the invoice description field but not properly linked does not satisfy the PO-linkage requirement | BR-002 | 2.5 |
| BR-03 | Credit notes are excluded from the qualifying population before PO-linkage evaluation | BR-003 | 2.3 |
| BR-04 | Only invoices with status draft or new AND an invoice date within the past seven days are evaluated | BR-004 | 1.2, 2.1 |
| BR-05 | The notification reports the total count of qualifying invoices; individual invoices are not listed in the message | BR-005 | 3.3 |
| BR-06 | The count is sent via Slack DM to Irina Capatina (Slack member ID WLX9BD8FN) with a no-PO-no-pay policy note, a remediation request, and a link to the filtered Coupa list | BR-006 | 3.3, 3.4 |
| BR-07 | Nothing is sent when a successful query returns no qualifying invoices | BR-007 | 3.1, 3.5 |
| BR-08 | No retry, fallback or recovery behaviour is required; a run that cannot complete is reported as a failed run with no notification | BR-008, BR-009 | 2.2, 4.1 |
| BR-09 | The automation does not create or modify purchase orders, approve invoices, change Coupa records, or track requester completion | BR-010 | All steps |

## 8. Business Exceptions

| ID | Name | Trigger step | Trigger condition | Action |
|---|---|---|---|---|
| B1 | Credit note in result set | 2.3 | Invoice type is credit note | Discard record from working set; continue loop |
| B2 | Description-only PO | 2.5 | PO number found only in description field, not in linked PO field | Treat as missing PO; include in qualifying count if other rules pass |
| B3 | No qualifying invoices (clean day) | 3.1 | qualifying_count = 0 after all filters applied | End run silently; send no Slack message (see OQ-02 for contradiction in source) |

## 9. System Errors

| ID | Name | Trigger condition | Severity | Retry policy | Action |
|---|---|---|---|---|---|
| S1 | Coupa query failure | Coupa API returns error, non-200, or unparseable response | High | None per BR-08 [DEFAULT] | Log failure; mark run as failed; send no notification |
| S2 | Slack delivery failure | Slack API returns error or non-200 on DM send | High | None per BR-08 [DEFAULT] | Log failure; mark run as failed |
| S3 | Application unresponsive | Coupa or Slack endpoint unreachable (timeout) | High | None per BR-08 [DEFAULT] | Log failure; mark run as failed |
| S4 | Credential expiry | Orchestrator credential asset retrieval fails | High | None [DEFAULT] | Log failure; mark run as failed; alert Orchestrator admin [DEFAULT] |
| S5 | Unhandled exception | Any uncaught runtime exception | High | None per BR-08 [DEFAULT] | Log exception with stack trace; mark run as failed |

## 10. Assumptions, Dependencies and Open Questions

1. **OQ-01 - Retry vs no-retry contradiction.** BR-08 prohibits retry, but image2.png shows Retry Coupa query and Retry Slack message (after 3 retries → Send error message) – SME must confirm which governs. `[SME REVIEW]`
2. **OQ-02 - Clean-day behaviour contradiction.** BR-07 and scope table say send nothing on a clean day; section 7.1 exceptions table says send a congratulatory Slack message – SME must confirm. `[SME REVIEW]`
3. **OQ-03 - Coupa access method.** Source states read access to Coupa but does not confirm API vs UI scraping; API is assumed. `[SME REVIEW]`
4. **OQ-04 - Coupa authentication.** API key vs OAuth for Coupa not specified in source. `[SME REVIEW]`
5. **OQ-05 - Coupa PO linkage field name.** Exact Coupa API field(s) representing a properly linked PO on invoice header/lines not named in source. `[SME REVIEW]`
6. **OQ-06 - Slack bot token scope.** Slack bot token with `chat:write` and DM permission to WLX9BD8FN required; provisioning not confirmed. `[SME REVIEW]`
7. **OQ-07 - Romania timezone handling.** EET (UTC+2) vs EEST (UTC+3) daylight saving transition must be handled by Orchestrator schedule or by automation. `[DEFAULT]` Orchestrator timezone set to Europe/Bucharest.
8. **OQ-08 - Block Kit JSON payload.** Source states exact Block Kit JSON is recorded in architectural considerations section 4; that section is not present in the source document – content is missing input. `[SME REVIEW]`
9. **OQ-09 - Coupa environment.** Source URL uses uipath-test.coupahost.com; production URL not provided. `[SME REVIEW]`
10. **Current-state process map (image1.png) readable.** Shows manual steps: Review Coupa invoice list → Missing PO invoice? → Copy invoice details → Group by requester → Post Slack message; confirms all manual steps to be eliminated.
11. **Future-state diagram (image2.png) readable.** Shows automated flow including retry nodes contradicted by BR-08; noted in OQ-01.

## 11. Success Criteria

1. A weekday test run executes at 10:00 Romania time and does not fire on weekends.
2. Only invoices with status draft or new and invoice date within the past seven days are evaluated.
3. Credit notes are absent from the qualifying count.
4. An invoice with a PO number present only in the description field is counted as missing a linked PO.
5. The Slack message reports the correct total count and is delivered as a DM to Slack member ID WLX9BD8FN.
6. The Slack message contains a working link to the Coupa invoice list filtered to the same date window used by the run.
7. A successful run returning zero qualifying invoices delivers no Slack message.
8. A run that cannot complete is recorded as a failed run in Orchestrator with no Slack message sent.
