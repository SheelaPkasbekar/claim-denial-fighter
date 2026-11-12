# Claim Denial Fighter

An automation that reads a health insurance claim denial letter, explains why it was denied, flags whether the denial was preventable, and drafts an appeal letter for human review. Every processed claim feeds a Power BI dashboard of denial trends.

> **All data in this project is synthetic.** No real patient, member or insurer information is used anywhere. Appeal letters are AI-generated drafts and always require human review. This tool does not approve, deny or give medical or legal advice.

## Why this project

Claim denials are expensive and frustrating, and many are preventable (missing prior authorization, missing or incorrect information, coding mismatches, duplicate claims). This project explores how a low-code automation plus a simple analytics layer can:

- turn an unstructured denial letter into structured data in seconds,
- explain the denial and suggest what to fix next time,
- give a first draft of an appeal for a person to review,
- show patterns across many denials in a dashboard.

## Demo

<!-- Add your screenshots and demo video link here -->
| n8n workflow | Denial analytics dashboard | Live appeals page |
|---|---|---|
| `images/n8n-workflow.png` | `images/dashboard-overview.png` | `images/live-appeals.png` |

Demo video: _add link here_

## How it works

```
Form (denial letter text)
        |
        v
AI extraction (Gemini) --> JSON: claim_id, member_id, payer, service_date,
        |                         cpt_code, procedure, diagnosis, billed_amount,
        v                         denial_code, denial_reason
Code node: clean JSON, categorize denial, mark preventable, add "what to fix" tip
        |
        v
Wait node (avoids API rate limits)
        |
        v
AI appeal draft (Gemini), marked "DRAFT - HUMAN REVIEW REQUIRED"
        |
        v
Google Sheets (append row) --> CSV --> Power BI dashboard
```

### n8n nodes

| Step | Node | Purpose |
|---|---|---|
| 1 | On form submission | Collects the denial letter text |
| 2 | Basic LLM Chain + Google Gemini Chat Model | Extracts key fields as JSON |
| 3 | Code (JavaScript) | Parses the JSON and applies denial-code rules |
| 4 | Wait | Pauses between the two AI calls to avoid rate limits |
| 5 | Basic LLM Chain (Draft Appeal) + Gemini | Writes the appeal letter draft |
| 6 | Google Sheets (Append row) | Logs the claim, category, fix tip and appeal draft |

Retry On Fail is enabled on both AI nodes to handle free-tier rate limits.

### Denial rules used in the Code node

| Denial code | Category | Preventable? |
|---|---|---|
| CO-197 | Missing prior authorization | Yes |
| CO-16 | Missing or incorrect information | Yes |
| CO-18 | Duplicate claim | Yes |
| CO-4 | Coding mismatch | Yes |
| CO-29 | Timely filing | Yes |
| CO-242 | Out-of-network provider | No |
| CO-50 | Not medically necessary | No |

## Dashboard

Built in Power BI from a synthetic dataset of 200 claims (`data/claims_dummy.csv`) plus the live output logged by n8n.

**Page 1: Denial Overview**
- KPI cards: denial rate, denied claims, amount at risk, preventable %
- Top denial reasons
- Denial rate by payer
- Preventable vs not preventable
- Denials over time
- Appeal status breakdown
- Table of urgent appeals sorted by days left to appeal
- Slicers: payer, service date

**Page 2: Live Appeals**
- Claim selector, claim details table and the full AI-drafted appeal letter for the selected claim

### Key DAX measures

```
Total Claims = COUNTROWS(claims_dummy)

Denied Claims =
CALCULATE(COUNTROWS(claims_dummy), claims_dummy[claim_status] = "Denied")

Denial Rate = DIVIDE([Denied Claims], [Total Claims])

Amount at Risk =
CALCULATE(SUM(claims_dummy[billed_amount]), claims_dummy[claim_status] = "Denied")

Preventable Denials =
CALCULATE(COUNTROWS(claims_dummy), claims_dummy[preventable] = "Yes")

Preventable % = DIVIDE([Preventable Denials], [Denied Claims])

Appeal Win Rate =
DIVIDE(
    CALCULATE(COUNTROWS(claims_dummy), claims_dummy[appeal_status] = "Won"),
    CALCULATE(COUNTROWS(claims_dummy), claims_dummy[appeal_status] IN {"Won", "Lost"})
)
```

## Tech stack

- **n8n** (cloud) for workflow automation
- **Google Gemini** for extraction and appeal drafting
- **Google Sheets** as the log
- **Power BI Desktop** for the dashboard
- **Python** (standard library) to generate the synthetic dataset

## Repository structure

```
claim-denial-fighter/
|-- README.md
|-- workflow/
|   `-- claim-denial-fighter.json      # n8n workflow export (credentials removed)
|-- data/
|   |-- claims_dummy.csv               # 200 synthetic claims
|   `-- sample_letters/                # 5 fake denial letters
|-- dashboard/
|   `-- Claim Denial Fighter.pbix      # Power BI file
`-- images/
    |-- n8n-workflow.png
    |-- dashboard-overview.png
    `-- live-appeals.png
```

## Run it yourself

1. **Import the workflow.** In n8n, create a new workflow, open the menu, choose Import from file and select `workflow/claim-denial-fighter.json`.
2. **Add credentials.** Connect your own Google Gemini API key and Google Sheets account. No credentials are included in this repo.
3. **Create the sheet.** Make a Google Sheet named `Claim Denial Log` with these headers in row 1:
   `claim_id, member_id, payer, service_date, cpt_code, procedure, diagnosis, billed_amount, denial_code, denial_reason, category, preventable, fix_tip, appeal_status, appeal_draft, logged_at`
4. **Test.** Click Execute workflow, open the form Test URL and paste one of the sample letters from `data/sample_letters/`.
5. **Dashboard.** Open the `.pbix` file in Power BI Desktop, point the data sources to your local CSV files and refresh.

## Safety and limitations

- Synthetic data only. Do not paste real patient or insurance documents into this workflow.
- The appeal drafts are starting points, not final letters, and are not legal or medical advice.
- The denial rules are simplified examples for demonstration.
- Free-tier AI limits can slow or block runs. The Wait and Retry settings reduce this but do not remove it.
- AI extraction can make mistakes, so a person should check each result before any use.

## What I learned

- Asking the model for strict JSON and parsing it in a Code node makes the workflow far more reliable.
- Rate limits are a real design constraint, so retries and wait steps are part of the build.
- Analytics is where an automation becomes useful to decision makers, not just a time saver.

## Ideas for next steps

- Gmail or Google Drive trigger so letters are processed automatically
- File upload support (PDF or image letters)
- Email the appeal draft for review
- Pre-submission claim checker to prevent denials before they happen
- Authentication on the form before sharing it publicly

## Author

Sheela Kasbekar | https://www.linkedin.com/feed/update/urn:li:activity:7526572247990329344/

## License

[Choose a license, for example MIT]
