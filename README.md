# Automated-Invoice-Expense-Tracking-System


# 🧾 Automated Invoice & Expense Tracking (n8n)

Turns invoice emails into clean, de-duplicated expense records in Google Sheets — no manual data entry.

**Flow:** Gmail ➜ filter attachments ➜ AI extraction (OpenAI) ➜ validate ➜ duplicate check ➜ Google Sheets

```
📧 Gmail Trigger → ⚙️ Config → 📎 Filter Attachments → 🧱 Build AI Request → 🤖 OpenAI
      → 🔍 Parse & Validate → ✅ Valid? ──yes──► 📚 Read Sheet → 🧮 Detect Duplicates → 🔁 Duplicate?
                                  │                                                      ├─yes─► 🚫 Duplicates Log
                                  └─no──► ⚠️ Needs Review sheet                          └─no──► 💾 Expenses sheet
```

---

## 📦 What's in this package

| File | Purpose |
|---|---|
| `invoice-expense-tracking.workflow.json` | **Main workflow** (import into n8n) |
| `error-alert.workflow.json` | Optional: emails you when the main workflow fails |
| `sheet-templates/Expenses.csv` | Column headers for the **Expenses** tab |
| `sheet-templates/Needs-Review.csv` | Column headers for the **Needs Review** tab |
| `sheet-templates/Duplicates-Log.csv` | Column headers for the **Duplicates Log** tab |
| `samples/sample-ai-output.json` | Example of what the AI returns (for reference/testing) |
| `CLIENT_HANDOVER_CHECKLIST.md` | Go-live checklist |

---

## ✅ Requirements

- **n8n** 1.70 or newer (Cloud or self-hosted). Self-hosted: use a public HTTPS URL so Google OAuth works.
- **Google account** with Gmail + Google Sheets
- **Google Cloud project** (free) for OAuth credentials — *skip if using n8n Cloud's one-click Google sign-in*
- **OpenAI API key** with billing enabled (model used by default: `gpt-4o-mini`)

---

## 🛠️ Setup — step by step

### Step 1 — Create the Google Sheet
1. Create a new Google Sheet (name it e.g. *Invoice & Expense Tracker*).
2. Create **three tabs** with these exact names (case-sensitive):
   - `Expenses`
   - `Needs Review`
   - `Duplicates Log`
3. In each tab, import the matching CSV from `sheet-templates/` (**File ➜ Import ➜ Upload ➜ "Replace current sheet"**) — or paste the header row into row 1.
4. Copy the **Sheet ID** from the URL:
   `https://docs.google.com/spreadsheets/d/`**`THIS_PART`**`/edit`

> ⚠️ Never rename or reorder-delete header columns. The workflow matches by header name. The `Duplicate Key` column in **Expenses** is what powers duplicate detection.

### Step 2 — Google credentials (Gmail + Sheets)
1. Go to <https://console.cloud.google.com> ➜ create/select a project.
2. **APIs & Services ➜ Library** ➜ enable **Gmail API** and **Google Sheets API**.
3. **OAuth consent screen** ➜ configure (External or Internal) and add your Google account as a **test user** (or publish the app).
4. **Credentials ➜ Create credentials ➜ OAuth client ID ➜ Web application.**
   Add the **OAuth Redirect URL** shown inside n8n's credential screen (e.g. `https://YOUR-N8N/rest/oauth2-credential/callback`).
5. In n8n: **Credentials ➜ New ➜ Gmail OAuth2 API** → paste Client ID/Secret → **Sign in with Google**.
6. Repeat for **Google Sheets OAuth2 API**.

### Step 3 — OpenAI credential
1. Create a key at <https://platform.openai.com/api-keys> and make sure billing is active.
2. In n8n: **Credentials ➜ New ➜ OpenAI API** ➜ paste the key.

### Step 4 — Import the workflow
**Workflows ➜ Add workflow ➜ ⋯ ➜ Import from file** ➜ choose `invoice-expense-tracking.workflow.json`.

### Step 5 — Attach credentials (6 nodes)
| Node | Credential |
|---|---|
| 📧 Gmail Invoice Trigger | Gmail OAuth2 |
| 🤖 AI Invoice Extraction | OpenAI API |
| 📚 Get Existing Expenses | Google Sheets OAuth2 |
| 💾 Save to Expenses Sheet | Google Sheets OAuth2 |
| 🚫 Log Skipped Duplicate | Google Sheets OAuth2 |
| ⚠️ Save to Needs Review | Google Sheets OAuth2 |

> Tip: open each Google Sheets node once and click the *Sheet* dropdown/refresh so n8n loads the column list.

### Step 6 — Fill in `⚙️ Workflow Config`
This is the **only node a client needs to edit**.

| Field | What it does | Default |
|---|---|---|
| `spreadsheetId` | **Required.** Your Google Sheet ID | *(paste)* |
| `expensesSheetName` | Tab for verified records | `Expenses` |
| `reviewSheetName` | Tab for items needing a human check | `Needs Review` |
| `duplicatesSheetName` | Audit tab for skipped duplicates | `Duplicates Log` |
| `aiModel` | OpenAI model (must support vision + PDFs + structured output) | `gpt-4o-mini` |
| `defaultCurrency` | Used when currency can't be determined | `USD` |
| `minConfidence` | AI confidence below this ➜ *Needs Review* (0–1) | `0.6` |
| `maxFileSizeMB` | Larger attachments are ignored | `15` |
| `allowedMimeTypes` | Accepted file types | PDF, PNG, JPG, WEBP |
| `expenseCategories` | Comma-separated list the AI must choose from | 12 common categories |

### Step 7 — (Optional) Tune the Gmail filter
Open **📧 Gmail Invoice Trigger ➜ Filters ➜ Search**. Default query:

```
has:attachment (invoice OR receipt OR bill OR "payment confirmation" OR statement) -in:sent -in:trash -in:spam
```
Better precision: create a Gmail filter that applies a label (e.g. `Invoices`) and change the query to `label:Invoices has:attachment`.

### Step 8 — Test
1. Send yourself an email with a sample invoice PDF (matching the Gmail query).
2. In the workflow click **Execute workflow** (or wait for the poll) ➜ check the **Expenses** tab.
3. Send the **same invoice again** ➜ it must appear in **Duplicates Log**, *not* in Expenses.
4. Send a non-invoice PDF ➜ it must land in **Needs Review**.

### Step 9 — Error alerts (recommended)
1. Import `error-alert.workflow.json`, attach Gmail credentials to **📨 Email Admin**, change `sendTo`, and **activate** it.
2. In the main workflow: **⋯ ➜ Settings ➜ Error Workflow** ➜ select *Invoice Tracking – Error Alert*.

### Step 10 — Activate
Toggle the main workflow to **Active**. It polls Gmail every minute.

---

## 🧠 How duplicates are detected
Each invoice gets a **Duplicate Key**:
- With invoice number: `vendor | invoice-number | total | currency`
- Without invoice number: `vendor | invoice-date | total | currency`

Keys are normalised (case/punctuation ignored). A new invoice is skipped if its key already exists in the **Expenses** tab **or** appears twice in the same batch. Skipped items are recorded in **Duplicates Log** for audit.

## 🚦 What goes to "Needs Review"
- AI says the file is not an invoice/receipt
- Vendor or total missing / total ≤ 0
- AI confidence below `minConfidence`
- The AI call failed (reason is written in *Review Reason*)

After fixing a record manually, copy it into **Expenses** (fill `Duplicate Key` too, so future copies are caught).

---

## 💰 Running cost
Only the OpenAI call costs money — typically a fraction of a cent to a few cents per document with `gpt-4o-mini`, depending on page count. Check current pricing at openai.com/pricing. Use a stronger model (e.g. `gpt-4o`) in `aiModel` for poor scans.

## 🔒 Security & privacy
- Invoice files are sent to OpenAI for processing — confirm this is acceptable under the client's data policy.
- Sheets are written in RAW mode, so text such as `=HYPERLINK(...)` in an invoice **cannot** execute as a formula.
- Keep OAuth credentials inside n8n only; never share the exported workflow with credentials attached.

## 🧯 Troubleshooting
| Problem | Fix |
|---|---|
| Nothing triggers | Workflow not *Active*; Gmail query too strict; Gmail credential expired (OAuth test-mode tokens expire after 7 days — publish the consent screen) |
| "No attachments" / Build node fails | Check `⚙️ Workflow Config` passes binary data (don't replace it with a node that strips binary) |
| 401 from OpenAI | Wrong key or no billing |
| 400 "Invalid schema / file" | `aiModel` doesn't support PDF input or structured outputs — use `gpt-4o-mini` or `gpt-4o` |
| Columns empty in sheet | Header names changed — restore from `sheet-templates/` |
| Everything flagged duplicate | `Duplicate Key` column in Expenses contains test data — clear test rows |
| Scanned/blurry invoices land in Review | Use `gpt-4o`, or lower `minConfidence` slightly |

## ⚠️ Known limits
- Only PDF / PNG / JPG / WEBP attachments (not Word, Excel, ZIP, or links inside email bodies).
- One record per attachment (multi-invoice PDFs are treated as one document).
- Very large Expenses sheets (tens of thousands of rows) will slow the read step; archive old years to another tab/file.
