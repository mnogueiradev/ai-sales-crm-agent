# Project: AI Agent (Flowise + Make)

This repository contains:

- `vip_email_generator.json` (Flowise) — a chatflow that generates a sales email from **Customer Name** and **Budget**.
- `google_integration_sheets.json` (Make) — a scenario that:
  - watches new rows in a spreadsheet (Google Sheets)
  - calls the Flowise endpoint to generate the email
  - sends the email via Gmail
  - writes the generated output back to the spreadsheet

> Important: these files were **sanitized** for publication on GitHub. They do **not** include real credentials, real spreadsheet IDs, or the real Flowise endpoint. To run it, you must fill in/reconnect everything in the platform UI.

---

## 1) Prerequisites

- A **Flowise** account/instance (self-hosted or cloud) with permission to create credentials and import chatflows.
- A **Make** (Integromat) account with permission to import a blueprint.
- Access to:
  - **Google Sheets**
  - **Gmail** (for sending)
- An API key for the LLM provider used by Flowise:
  - **Groq API Key** (for the `GroqChat` node)

---

## 2) Flowise setup (import and enable the Chatflow)

### 2.1 Import the chatflow

1. Open Flowise.
2. Go to **Chatflows**.
3. Click **Import** and select `vip_email_generator.json`.
4. Open the imported chatflow.

### 2.2 Configure the LLM credential (Groq)

This chatflow uses a `GroqChat` node that references a credential of type `groqApi`.

1. In Flowise, go to **Credentials**.
2. Create a **Groq API** credential (or the equivalent in your Flowise build).
3. Fill it with your **GROQ_API_KEY**.
4. Go back to the chatflow and, in the `GroqChat` node, select that credential in **Connect Credential**.
5. Save the chatflow.

> Security: **never** put your API key inside the exported JSON or in files committed to GitHub.

### 2.3 Get the chatflow endpoint (Prediction URL)

You will need the chatflow execution URL to use it from Make.

1. In Flowise, open the chatflow.
2. Find the **API / Endpoint / Prediction URL** option.
3. Copy a URL like:

```text
https://YOUR_FLOWISE_HOST/api/v1/prediction/YOUR_CHATFLOW_ID
```

> In the Make blueprint published in this repo, this URL is intentionally fake (`example.com`). You will replace it with yours.

### 2.4 Quick test inside Flowise

1. Use the chatflow **Playground/Test**.
2. Send something like:

```text
Customer Name: John | Budget: 3000
```

3. Confirm it returns the generated email (no 401/403 errors).

---

## 3) Make setup (import and configure the Scenario)

### 3.1 Import the blueprint

1. In Make, create a new scenario.
2. Use **Import Blueprint**.
3. Select `google_integration_sheets.json`.

### 3.2 Reconnect the connections (Google / Gmail)

The blueprint was exported with connection IDs removed (`__IMTCONN__ = 0`).

For each module that asks for a **Connection**:

- `google-sheets:*` modules:
  - connect your Google account with access to the spreadsheets.
- `google-email:sendAnEmail` module:
  - connect the Gmail (or Google Workspace) account that will send the emails.

> Make will use Google OAuth. You **do not** need to put any password in the blueprint.

### 3.3 Select the spreadsheets and sheets (Spreadsheet/Sheet)

The `spreadsheetId` values were redacted in the JSON to avoid exposing real IDs.

Configure the following modules manually:

- `google-sheets:watchRows`
  - select the correct Spreadsheet
  - select the Sheet/tab (e.g., `Form Responses 1`)
  - adjust headers/range if needed

- `google-sheets:addRow`
  - select the destination Spreadsheet (e.g., your leads database)
  - select the Sheet/tab (e.g., `Sheet1`)

- `google-sheets:updateRow`
  - select the source Spreadsheet
  - select the Sheet/tab
  - confirm `rowNumber` and the column where the AI output will be written

### 3.4 Configure the HTTP request to Flowise (endpoint)

In the `http:MakeRequest` module, replace the fake URL:

```text
https://example.com/api/v1/prediction/REDACTED_FLOWISE_CHATFLOW_ID
```

with your real Flowise URL:

```text
https://YOUR_FLOWISE_HOST/api/v1/prediction/YOUR_CHATFLOW_ID
```

#### If your Flowise endpoint is private
Depending on how you secured Flowise, you may need to configure authentication in Make:

- **No authentication (noAuth)**:
  - works only if your instance allows requests without a token.

- **API key / Bearer token** (if applicable in your Flowise):
  - configure request headers in the HTTP module, for example:

```text
Authorization: Bearer YOUR_TOKEN
```

> Security: **do not commit** tokens to GitHub. Store credentials/headers securely in Make (connections/secrets) or configure them manually in your scenario.

### 3.5 Test the scenario

1. Click **Run once**.
2. Trigger an event:
   - if your trigger is `watchRows`, add a new row in the monitored spreadsheet.
3. Validate the execution:
   - `watchRows` reads the row
   - `http:MakeRequest` returns the generated text
   - `google-email:sendAnEmail` sends the email
   - `updateRow` writes the result back to the spreadsheet

---

## 4) Expected fields and mapping

The HTTP module sends Flowise a JSON payload like:

```json
{
  "question": "Customer Name: {{Name}} | Budget: {{Budget}}"
}
```

Flowise returns a text output (the email) that Make uses to:

- send via Gmail
- write to the spreadsheet

---

## 5) Security (recommended before publishing)

- Do not commit:
  - API keys (Groq)
  - tokens/`Authorization` headers
  - private internal URLs
  - real spreadsheet IDs if they are sensitive for you

- Prefer:
  - **Flowise Credentials** for provider keys
  - **Make Connections** (OAuth)
  - environment variables/secrets (self-host)

---

## 6) Troubleshooting

- **401/403 in Flowise**:
  - confirm the Groq credential is selected in the `GroqChat` node.
  - if the endpoint is private, check auth configuration in Make.

- **Make error accessing Google Sheets**:
  - reconnect your Google account.
  - re-select Spreadsheet/Sheet (IDs were redacted in the exported file).

- **HTTP request fails**:
  - confirm the Flowise URL is correct.
  - confirm your Flowise instance is reachable from Make (publicly or via network/VPN).

---

## Status

- The JSON files in this repository are **safe to publish**.
- To run them, you must **configure credentials and endpoints** manually following the instructions above.
