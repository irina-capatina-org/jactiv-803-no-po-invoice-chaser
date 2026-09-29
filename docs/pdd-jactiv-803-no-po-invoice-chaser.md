# PDD - No-PO Invoice Chaser

## Document History

| Date | Version | Author | Role | Comments |
|---|---|---|---|---|
| 2026-09-29 | 0.1 | uipath-analyst | Analyst | Initial PDD created from docs/jactiv-803-request-details.docx. |

| Date | Version | Author | Role | Comments |
|------|---------|--------|------|----------|
| 2026-09-29 | 0.1 | uipath-analyst | Analyst | Initial analysis from request-work/request-details.md (No-PO invoice finder v1.0, UiPath Cartographer, 16 Sep 2026) |

## 1. Document Control

| Field | Value |
|-------|-------|
| Document | PDD - No-PO Invoice Chaser |
| Story key | JACTIV-803 |
| Epic key | [SME REVIEW] |
| Source file | request-work/request-details.md |
| Branch | analysis-jactiv-803 |
| Author | uipath-analyst |
| Status | Draft - pending SME approval |
| Version | 0.1 |

## 2. Introduction

**Process name:** No-PO Invoice Chaser
**Process Full Name:** `NoPoInvoiceChaser`

**Business objective:** Replace the manual daily Coupa review with a fully automated weekday run that identifies invoices with no properly linked purchase order, counts them, and sends one Slack summary to the AP responsible — enforcing the no-PO-no-pay policy consistently without human filtering effort.

**Owning department:** Accounts Payable

| Role | Name / Contact |
|------|---------------|
| SME / Process Owner | Irina Capatina (irina.capatina@uipath.com, Slack ID WLX9BD8FN) |
| BA | uipath-analyst |
| Developer | [SME REVIEW] |

## 3. Process Overview

| Field | Value |
|-------|-------|
| Process full name | NoPoInvoiceChaser |
| Function and department | Invoice compliance check — Accounts Payable |
| Short description | Queries Coupa daily for invoices (status draft/new, past 7 days, no linked PO, excluding credit notes) and sends one Slack Block Kit message with the count and a filtered Coupa link to the AP SME. Sends nothing on a clean day. |
| Required roles | Automation (unattended); SME Irina Capatina (Slack recipient only) |
| Trigger and schedule | Weekday schedule, 10:00 Romania time (Europe/Bucharest) |
| Volume (items per day / peak) | ~1 run/day; example sample showed 194 qualifying invoices |
| Average handling time | Manual: ~daily ad-hoc review; Automated target: <2 min per run [SME REVIEW] |
| FTE effort | [SME REVIEW] |
| Estimated exception rate | Low — structured data, deterministic rules; credit notes and description-only POs are the only known variants |
| Input data | Coupa invoice list (status, invoice date, PO linkage, invoice type) |
| Output data | Slack Block Kit DM to SME with invoice count, policy note, action request, filtered Coupa URL, and run date window |

## 4. To-Be Process (High Level)

The automation runs unattended on weekdays at 10:00 Romania time. It reads the Coupa invoice list, filters to invoices dated within the past seven days with status draft or new, excludes credit notes, and checks each invoice for a properly linked purchase order (a PO number in the description field alone does not qualify). It counts the invoices that fail the linkage check and, if the count is greater than zero, sends one Slack Block Kit direct message to the SME. If the count is zero, nothing is sent.

**Manual steps that disappear:**
- AP member's daily Coupa navigation and manual filter setup
- Field copying and message drafting
- Inconsistent timing and omission risk

**Stays human:** SME receipt of the Slack message and any follow-up action to raise or link POs. Requester direct notification is out of scope. No Coupa record is modified by the automation.

## 5. Detailed Process Steps

| Step | Action | Application | Expected Result | Remarks |
|------|--------|-------------|-----------------|---------|
| 1.1 | Weekday schedule fires at 10:00 Europe/Bucharest | Scheduler | Run starts; current date and seven-day window (today − 7 days to today) computed | Weekdays only; Saturday/Sunday suppressed by schedule |
| 1.2 | Authenticate to Coupa API using stored credentials | Coupa | Valid session / API token obtained | Credential stored in UiPath Orchestrator credential asset [DEFAULT] |
| 1.3 | Query Coupa invoice list: status IN (draft, new), invoice_date >= window_start, invoice_date <= window_end | Coupa | Page 1 of invoice records returned | Apply BR-04; use Coupa REST API with query params; paginate if result set exceeds page size [SME REVIEW — confirm page size limit] |
| 1.4 | For each invoice record: check invoice type | Coupa | Records identified as credit notes collected for exclusion | Apply BR-03; if type = credit note → skip to next record |
| 1.5 | For each non-credit-note record: check PO linkage on invoice LINES (not header) | Coupa | Boolean flag set: PO linked = true / false | Apply BR-01, BR-02; a PO number present only in the description field does not satisfy linkage |
| 1.6 | Collect all records where PO linked = false into qualifying list | Automation | Qualifying invoice list populated | Count derived from this list |
| 1.7 | Count qualifying records → invoice_count | Automation | Integer invoice_count ≥ 0 | Apply BR-05 |
| 1.8 | **Decision:** invoice_count = 0? | Automation | Branch: zero → step 1.9; non-zero → step 1.10 | Apply BR-07 |
| 1.9 | invoice_count = 0: end run, send no notification | Automation | Run completes silently; nothing sent to Slack | Apply BR-07; clean day = no message |
| 1.10 | Build filtered Coupa URL using window_start and window_end query parameters | Automation | coupa_url string constructed | URL pattern from source: `https://uipath-test.coupahost.com/invoices?q%5Binvoice_date_gteq%5D=<window_start>&q%5Binvoice_date_lteq%5D=<window_end>&q%5Bstatus_eq%5D=draft` [SME REVIEW — confirm production host] |
| 1.11 | Compose Slack Block Kit JSON payload: substitute {{invoice_count}}, {{coupa_url}}, {{window_start}}, {{window_end}}, {{run_date}} | Automation | Block Kit JSON ready; title shows count for sidebar visibility | Title: `:receipt: {{invoice_count}} invoices need a purchase order`; body: three icon-led lines; primary button linking to coupa_url; footer with dates and run_date |
| 1.12 | Send Slack DM to SME Slack member ID WLX9BD8FN via Slack HTTP Request activity | Slack | Message delivered; HTTP 200 response | Apply BR-06; recipient identified by Slack member ID, not email; DM only |
| 1.13 | Log run outcome (success, invoice_count, run timestamp) | Automation | Run result recorded in Orchestrator job log | [DEFAULT] |

## 6. Applications and Systems

| Application | Interface type | Access method | Login method | Credential handling | Comments |
|-------------|---------------|---------------|--------------|---------------------|----------|
| Coupa | API | REST API (read-only) | API key / OAuth [SME REVIEW] | Orchestrator credential asset [DEFAULT] | PO linkage is on invoice lines, not header; paginate large result sets |
| Slack | API | HTTP Request activity (Incoming Webhook or Bot API) [SME REVIEW] | Bot token | Orchestrator credential asset [DEFAULT] | Recipient: Slack member ID WLX9BD8FN; Block Kit JSON payload; send DM only |
| UiPath Orchestrator | Platform | Orchestrator triggers | Robot credentials | N/A | Hosts schedule, credential assets, and job logs |

## 7. Business Rules

| ID | Rule | Source | Applies at step |
|----|------|--------|-----------------|
| BR-01 | Apply the no-PO-no-pay policy: any invoice without a properly linked purchase order qualifies for the notification. | BR-001 | 1.5, 1.6 |
| BR-02 | A PO number typed into the invoice description but not formally linked on invoice lines does not satisfy the PO requirement. | BR-002 | 1.5 |
| BR-03 | Exclude credit notes from the qualifying population before counting. | BR-003 | 1.4 |
| BR-04 | Include only invoices with status draft or new and an invoice date within the past seven calendar days. | BR-004 | 1.3 |
| BR-05 | Count the qualifying invoices; report the total only — individual invoices are not listed in the message. | BR-005 | 1.7, 1.11 |
| BR-06 | Send the count, policy context, action request, and a filtered Coupa link to Irina Capatina (Slack ID WLX9BD8FN) by Slack DM. | BR-006 | 1.12 |
| BR-07 | Send nothing when a successful query returns zero qualifying invoices. | BR-007 | 1.8, 1.9 |
| BR-08 | No retry, fallback or recovery behaviour. A run that cannot complete is simply a failed run. | BR-008 | All |
| BR-09 | A run that cannot complete produces no notification; no second message path exists. | BR-009 | All |
| BR-10 | The automation must not create or modify purchase orders, approve invoices, change Coupa records, or track requester completion. | BR-010 | All |

## 8. Business Exceptions

| ID | Name | Trigger step | Trigger condition | Action |
|----|------|-------------|-------------------|--------|
| B1 | Credit note encountered | 1.4 | Invoice type = credit note | Exclude record from qualifying list; continue to next record (BR-03) |
| B2 | Description-only PO | 1.5 | PO number found in description field but no formal line-level linkage | Treat as missing PO; include in qualifying list if other rules pass (BR-02) |
| B3 | Zero qualifying invoices | 1.8 | invoice_count = 0 after all filtering | End run silently; send no Slack message (BR-07) |

## 9. System Errors

| ID | Name | Trigger condition | Severity | Retry policy | Action |
|----|------|-------------------|----------|-------------|--------|
| S1 | Coupa authentication failure | API auth call returns 401/403 | High | No retry (BR-08) | Fail run; log error in Orchestrator; no notification sent (BR-09) [DEFAULT] |
| S2 | Coupa query error | Invoice query returns non-2xx or network timeout | High | No retry (BR-08) | Fail run; log error; no notification sent [DEFAULT] |
| S3 | Slack delivery failure | Slack HTTP Request returns non-2xx | High | No retry (BR-08) | Fail run; log error; no second message attempted (BR-09) [DEFAULT] |
| S4 | Application unresponsive | Coupa or Slack API unreachable | High | No retry (BR-08) | Fail run; log error [DEFAULT] |
| S5 | Credential expiry | Orchestrator credential asset missing or expired | High | No retry (BR-08) | Fail run; alert via Orchestrator job failure notification [DEFAULT] |
| S6 | Unhandled exception | Unexpected runtime error in any step | High | No retry (BR-08) | Fail run; log full stack trace in Orchestrator [DEFAULT] |

## 10. Assumptions, Dependencies and Open Questions

1. **OQ-01 - Coupa API access confirmed.** Does the AP team have an active Coupa API key or OAuth client with read-only invoice scope? `[SME REVIEW]`
2. **OQ-02 - Coupa production host URL.** Source example uses `uipath-test.coupahost.com`; confirm the production hostname for the Coupa URL in the notification. `[SME REVIEW]`
3. **OQ-03 - Slack integration method.** Confirm whether to use a Slack Bot API token (chat.postMessage) or an Incoming Webhook; determines credential type and DM capability. `[SME REVIEW]`
4. **OQ-04 - Coupa API pagination.** Confirm maximum page size for the invoice query endpoint so the robot paginates correctly on high-volume days. `[SME REVIEW]`
5. **OQ-05 - Block Kit JSON payload.** Source states the exact Block Kit JSON is in the architectural considerations (section 4 of source). That appendix content is not present in the provided file; the SDD must supply or confirm the payload. `[SME REVIEW]`
6. **OQ-06 - Clean-day message conflict.** Section 7.1 of the source states a zero-count run should send a "congrats" Slack message; BR-007 and scope table say send nothing. `[SME REVIEW — resolve before build]`
7. **OQ-07 - Romania timezone handling.** Schedule must use Europe/Bucharest (UTC+2/+3 DST); confirm Orchestrator timezone setting. `[DEFAULT: Europe/Bucharest]`
8. **Coupa PO linkage field.** PO linkage is on invoice lines, not the invoice header — confirmed by source text; the API query must traverse line items. `[SME REVIEW — confirm field path in Coupa API response]`
9. **No retry scope.** BR-08 and BR-09 explicitly exclude retry/fallback; system errors are failed runs only. `[DEFAULT applied throughout section 9]`
10. **Appendix A diagrams.** Source references embedded process maps in sections 3.4 and 4.1; these are image placeholders only and were not read. If the diagrams contain rules not captured in text, they must be reviewed by the SME.

## 11. Success Criteria

1. A weekday test run executes at 10:00 Europe/Bucharest and does not run on weekends.
2. Only invoices with status draft or new and invoice_date within the past seven days are evaluated.
3. Credit notes are excluded from the count.
4. An invoice with a PO number only in the description field is counted as missing a linked PO.
5. A Slack Block Kit DM is sent to Slack member ID WLX9BD8FN containing the correct count and a working filtered Coupa link when count > 0.
6. No Slack message is sent when the qualifying count is zero.
7. The automation does not create, modify or approve any Coupa record or purchase order.
8. A run that encounters a system error is recorded as a failed job in Orchestrator and sends no Slack notification.
