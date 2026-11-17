# Extract Invoice Data with AI

A [Make.com](https://www.make.com) scenario that picks up PDF invoices received by email, reads them with OCR, uses an AI model (Mistral) to extract the key financial data, and logs each invoice in Google Sheets with a list of anomalies to check.

## The scenario

```
Gmail (email with attachment) ──► Attachments ──► Google Cloud Vision (OCR) ──► Mistral AI (extract as JSON) ──► Parse JSON ──► Google Sheets (add row)
```

## What it does

1. **Watch new emails** (Gmail): checks your inbox for emails with attachments.
2. **Get the attachments** (Gmail): downloads the PDF invoice.
3. **Read the invoice** (Google Cloud Vision): extracts the text of the PDF (up to 5 pages) with OCR.
4. **Extract the data** (Mistral AI, `mistral-medium-latest`): sends the text to the model with a system prompt (see [`prompt.md`](prompt.md)). The model only uses what is written on the invoice (missing values are `null`) and returns:
   - vendor and customer name and address, invoice number, invoice date, due date
   - currency (ISO code), subtotal, tax, discount, shipping, total amount
   - payment terms, VAT number, PO number, bank details (IBAN, SWIFT, account number)
   - line items (description, quantity, unit price, total)
   - anomalies: due date before invoice date, or missing invoice number
5. **Parse JSON**: turns the model's reply into structured data.
6. **Log the invoice** (Google Sheets): adds a row with the main fields. The scenario also checks the maths itself: if subtotal + tax + shipping − discount doesn't equal the total, it adds an anomaly.

> **The AI can make mistakes.** OCR can misread a digit and the AI can put a value in the wrong field. Always check amounts, dates and bank details against the original invoice before paying, and be especially careful with IBANs.

## Requirements

- A Make.com account (the free plan is enough to try it)
- A Gmail account
- A Google Cloud project with the **Cloud Vision API** enabled
- A Mistral AI API key ([console.mistral.ai](https://console.mistral.ai))
- A Google account (Sheets)

## Google Sheet setup

Create a spreadsheet with these headers in the first row:

| Header         | Filled with                     |
|----------------|---------------------------------|
| InvoiceNumber  | `invoice_number`                |
| InvoiceDate    | `invoice_date`                  |
| DueDate        | `due_date`                      |
| VendorName     | `vendor_name`                   |
| VendorAddress  | `vendor_address`                |
| Currency       | `currency`                      |
| Subtotal       | `subtotal`                      |
| TaxAmount      | `tax_amount`                    |
| ShippingAmount | `shipping_amount`               |
| TotalAmount    | `total_amount`                  |
| PaymentTerms   | `payment_terms`                 |
| LineItems      | Line item descriptions, separated by ` \| ` |
| Anomalies      | Anomalies from the AI + total check |

## Installation

1. Download [`Extract Invoice Data with AI.blueprint.json`](Extract%20Invoice%20Data%20with%20AI.blueprint.json).
2. In Make, create a new scenario, open the **⋯** menu, choose **Import Blueprint** and select the file.
3. Reconnect each module to your own accounts:
   - **Gmail**: add your connection.
   - **Google Cloud Vision**: add your connection.
   - **Mistral AI**: add a connection with your API key.
   - **Google Sheets**: pick **your** spreadsheet and sheet.
4. In the **Parse JSON** module, rebuild the data structure if Make asks for it.
5. Click **Run once** and send yourself an email with a test PDF invoice.
6. Turn on scheduling when you are happy with the result.

## Customizing

- **Fields**: add or remove fields in the Mistral system prompt. [`prompt.md`](prompt.md) has a readable copy.
- **More columns**: the AI also extracts the customer, VAT number, PO number and bank details. Add columns to log them.
- **Model**: any Mistral model works. A lower `temperature` (for example `0.2`) gives more consistent extraction.
- **Email filter**: limit the Gmail trigger to a label or a sender (for example your invoices mailbox).

## Privacy and security

- **Invoice contents are sent to Google Cloud Vision and to Mistral AI** for processing. Invoices often contain bank details and personal data: only use this with documents you are allowed to share with these services.
- **API keys and OAuth tokens stay in your Make connections** and are never exported in the blueprint.
- **Invoices are untrusted input.** A fake invoice could contain text designed to trick the AI (prompt injection). The extraction never triggers a payment, but always verify the vendor and bank details before paying.
