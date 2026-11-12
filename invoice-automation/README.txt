# AI Invoice & Document Processing (n8n + Local AI)

An automation that reads invoice PDFs, extracts the key fields with a local AI model, checks the result, and then either files the invoice automatically or sends it to a human for review. It runs entirely on my own computer with free tools, with no paid APIs.

**Demo video:** [paste Loom link]

## What it does

1. An invoice PDF arrives (by upload form or webhook).
2. The text is extracted from the PDF.
3. A local AI model (Qwen 2.5 3B, run by Ollama) returns structured JSON: vendor, invoice number, date, currency, total, and line items.
4. A validation step checks the AI's answer: required fields present, date format, line items add up to the total, invoice number really appears in the document, and the total is not unusually high.
5. Every invoice is saved to PostgreSQL with a status of `filed` or `needs_review`, plus the reason if flagged.
6. Clean invoices are added to a Google Sheet. Flagged invoices trigger an email with the reason.
7. A separate error workflow emails me if any run crashes.

## Workflow

```
Form Upload ─┐
             ├→ Extract PDF text → Local AI (Qwen via Ollama) → Parse JSON → Validate → PostgreSQL → IF
Webhook ─────┘                                                                                      ├─ needs_review → Email alert
                                                                                                    └─ filed → Google Sheet

Any failure → Error workflow → Email alert
```

## Tools used

n8n (self-hosted, Docker) · Ollama with Qwen 2.5 3B · PostgreSQL 16 (Docker) · Google Sheets · Gmail · JavaScript (Code nodes) · Webhooks and JSON

## Results

Tested on 30 synthetic invoices from 5 fictional vendors, each with a different layout, date format, and currency (PHP and USD). 6 of the 30 were deliberately broken (missing invoice number, missing date, wrong total, unusually high total) to test the validation.

| | First run | Final run |
|---|---|---|
| Invoices saved | 29 / 30 | 30 / 30 |
| All fields extracted correctly | 27 / 30 | 30 / 30 |
| Routed correctly (filed vs. flagged) | 26 / 30 | 29 / 30 |
| Bad invoice wrongly filed | 1 | 0 |
| Good invoice wrongly flagged | 2 | 1 |

All 6 deliberately broken invoices were flagged for review in the final run. The one routing miss was a false alarm: the model dropped part of a line item, the validator noticed the mismatch, and the invoice went to human review.

### What went wrong in the first run, and how I fixed it

| Problem | Cause | Fix |
|---|---|---|
| Invoice with no number was filed with "INVOICE" as its number | The model picked up the page heading | Stricter prompt, plus a check that blank or non-numeric values become `null` |
| An invoice vanished from the database | Two invoices returned the same fake number, and the duplicate-protection silently skipped the second | Fixed the root cause (above) |
| Invoices with no number got a stray digit as their number | The model invents plausible values when data is missing | Validation now confirms the invoice number appears right after an invoice label in the PDF text |
| Philippine peso amounts read as USD, triggering a false high-total flag | The invoice used "P" instead of "PHP" | Added currency rules to the prompt |

The main lesson: with a small local model, the most reliable fix was not a better prompt but a validation step that checks the AI's answer against the source document.

## Setup

Requirements: Docker Desktop, Ollama, n8n.

1. Pull the model: `ollama pull qwen2.5:3b`
2. Start PostgreSQL with the included `docker-compose.yml`: `docker compose up -d`
3. Create the table (see `schema.sql` or the SQL below).
4. In n8n, import `invoice-processing-workflow.json` and `error-alert-workflow.json`.
5. Create your own credentials in n8n for Postgres, Google Sheets, and Gmail (none are included in the export).
6. Set `Invoice Error Alert` as the error workflow in the main workflow's settings.
7. Publish the workflow. Send a test invoice:

```
curl -X POST -F "Invoice=@sample.pdf" http://localhost:5678/webhook/invoice-upload
```

```sql
CREATE TABLE invoices (
  id SERIAL PRIMARY KEY,
  vendor TEXT,
  invoice_number TEXT UNIQUE,
  invoice_date DATE,
  total NUMERIC(12,2),
  currency TEXT,
  line_items TEXT,
  status TEXT DEFAULT 'filed',
  review_reason TEXT,
  source_file TEXT,
  created_at TIMESTAMP DEFAULT NOW()
);
```

## Known limits

- **Synthetic data only.** Tested on invoices I generated, not real business documents.
- **PDFs with selectable text only.** Scanned images would need an OCR step, which is not included.
- **Small model.** A 3B model can occasionally miss a line item or misread a value. That is why every result goes through validation and uncertain ones go to a human.
- **Fixed rules.** The high-total limits (PHP 100,000 / USD 2,000) and the 12% VAT check are set for this test and would need adjusting per client.
- **Duplicate invoice numbers** are skipped by the database. A production version should report these instead of skipping them silently.

## Possible next steps

Add OCR for scanned invoices, handle multi-page invoices, report duplicates, and add a weekly summary email of totals by vendor.