# Build Guide — Autonomous Email Monitor Agent

This guide walks through recreating the Email Monitor Agent from scratch in Power Automate. It covers every step, configuration value, and known pitfall.

**Estimated build time:** 60–75 minutes (includes Teams Delivery Flow setup)
**Prerequisites:** Microsoft 365 account (Business Basic or higher) · Azure OpenAI deployment · Teams channel

---

## Architecture Overview

This agent uses **two Power Automate flows**. Build them in order:

| Flow | Purpose |
|---|---|
| **Teams Delivery Flow** (Step 0) | Receives HTTP POST requests and posts Adaptive Cards to a Teams channel. Its trigger URL becomes `TEAMS_WEBHOOK_URL`. |
| **Email Monitor Agent** (Steps 1–9) | Triggers on new Outlook emails, calls Azure OpenAI, and POSTs to the Teams Delivery Flow for high-priority emails. |

---

## What You Will Build

A cloud flow that:
1. Triggers automatically on every new Outlook email
2. Sends the email to GPT-4.1-mini for classification
3. Posts a Teams Adaptive Card for Urgent / Action Required emails
4. Silently terminates for FYI / No Action emails

---

## Configuration Values

Replace these with your own values throughout the guide:

| Variable | What it is |
|---|---|
| `YOUR_OPENAI_ENDPOINT` | `https://YOUR-RESOURCE.openai.azure.com/openai/deployments/gpt-4.1-mini/chat/completions?api-version=2025-01-01-preview` |
| `YOUR_API_KEY` | Azure OpenAI API key from Azure Portal → AI Foundry resource → Keys |
| `TEAMS_WEBHOOK_URL` | Power Automate webhook URL (from a separate flow with HTTP trigger → Post to Teams) |

> **API version note:** Use `2025-01-01-preview` or later. Earlier versions (e.g. `2024-02-15-preview`) do not support GPT-4.1-mini and return a 404.

---

## Step 0 — Create the Teams Delivery Flow

This is a separate Power Automate flow that receives the Adaptive Card payload from the Email Monitor Agent and posts it to your Teams channel. Its HTTP trigger URL becomes `TEAMS_WEBHOOK_URL` in Step 7.

1. Go to [make.powerautomate.com](https://make.powerautomate.com)
2. Click **+ Create** → **Instant cloud flow**
3. In the dialog, name it: `Teams Delivery Flow`
4. In the trigger search box, type **"HTTP request"** → select **"When an HTTP request is received"** → Click **Create**
5. Leave the trigger block at its defaults — the HTTP POST URL is generated when you first Save

6. Click **+ New step** → search **"Teams"** → look for **"Post your own adaptive card as the Flow bot to a channel"**

   > **Finding this action:** It is a legacy Teams connector action and may not appear in the top results. If you don't see it immediately, scroll down in the Teams actions list past the newer actions (e.g. "Post message in a chat or channel"). Do **not** use "Post message in a chat or channel" (V2) — it does not accept the Adaptive Card attachment format that the Email Monitor Agent sends.

7. Configure the action:
   - **Team:** select your target team (e.g. Ahead with Anurodh)
   - **Channel:** select your target channel (e.g. Notification Trigger Agent)
   - **Message:** click the **expression** tab (not dynamic content) → enter:
     ```
     triggerBody()?['attachments']?[0]?['content']
     ```

   > **If the card renders blank or throws a type error on Save:** Some connector versions require the Message value as a JSON string rather than an object. Try this expression instead: `string(triggerBody()?['attachments']?[0]?['content'])`

8. Click **Save**
9. Click the **"When an HTTP request is received"** trigger block to expand it → copy the **HTTP POST URL** — this is your `TEAMS_WEBHOOK_URL`

> **Why a separate flow?** A single Power Automate flow can only have one trigger. The Email Monitor Agent uses an Outlook trigger; the delivery mechanism needs an HTTP trigger to receive webhook calls. Separating them also makes the delivery endpoint reusable across other agents.

---

## Step 1 — Create the Email Monitor Flow

1. Go to [make.powerautomate.com](https://make.powerautomate.com)
2. Sign in with your Microsoft 365 account
3. Click **+ Create** → **Automated cloud flow**
4. Name it: `Email Monitor Agent`
5. Search for trigger: **"When a new email arrives (V3)"** (Office 365 Outlook connector)
6. Click **Create**

---

## Step 2 — Configure the Trigger

In the **"When a new email arrives (V3)"** block:

| Field | Value |
|---|---|
| Folder | Use the **folder picker icon** to select Inbox — do NOT type it manually |
| Include Attachments | No |
| Only with Attachments | No |

> **Critical:** The Folder field must use the picker UI. It resolves to an internal Exchange folder ID (not the string "Inbox"). Typing "Inbox" as plain text causes a 404 error at runtime.

---

## Step 3 — Add HTTP Action (Azure OpenAI)

1. Click **+ New step**
2. Search for **"HTTP"** → select the **HTTP** action (standalone, not HTTP + Swagger)
3. **Do not rename this step.** Leave the default name **"HTTP"**. The condition expression in Step 6 references it as `body('HTTP')`. If you rename it (e.g. to "Azure OpenAI"), you must update all downstream expressions to match (`body('Azure_OpenAI')`).
4. Configure:

**Method:** `POST`

**URI:**
```
https://YOUR-RESOURCE.openai.azure.com/openai/deployments/gpt-4.1-mini/chat/completions?api-version=2025-01-01-preview
```

**Headers** (add two rows — type directly, do not copy from formatted docs):

| Key | Value |
|---|---|
| `Content-Type` | `application/json` |
| `api-key` | `YOUR_API_KEY` |

> **Critical:** Do not copy header names from markdown. Backtick characters (`Content-Type`) cause 401 Unauthorized. Type the header names and values directly in the field.

**Body** (paste this, then replace the dynamic content placeholders):

```json
{
  "messages": [
    {
      "role": "system",
      "content": "You are an email classifier for a business professional. Classify the email into exactly one category: Urgent, Action Required, FYI, or No Action. Respond with only a JSON object in this exact format: {\"classification\": \"CATEGORY\", \"reason\": \"one sentence explanation\", \"suggested_action\": \"what the recipient should do\"}"
    },
    {
      "role": "user",
      "content": "From: FROM_PLACEHOLDER Subject: SUBJECT_PLACEHOLDER Body: BODY_PLACEHOLDER"
    }
  ],
  "max_tokens": 200,
  "temperature": 0.1
}
```

**How to replace the placeholders with dynamic content:**

The Body field is a plain text editor. Power Automate lets you embed dynamic values by positioning your cursor in the text and selecting from the dynamic content panel. For each placeholder:

1. In the Body field, select and delete the text `FROM_PLACEHOLDER` — keep the surrounding `"` quote marks
2. With the cursor between the quotes, click **Add dynamic content** (the link below the field) → under the **"When a new email arrives"** section → click **From**. Power Automate inserts a token that visually shows as `[From]`.
3. Repeat for `SUBJECT_PLACEHOLDER` → select **Subject** under "When a new email arrives"
4. Repeat for `BODY_PLACEHOLDER` → select **Body** under "When a new email arrives" — this is the plain-text version. **Do not select "HTML Body"** — the HTML tags pass through to GPT and degrade classification quality.

> **Tip:** The dynamic content panel groups fields by step name. Always confirm you are picking from the "When a new email arrives" section, not from a different step.

> **Critical:** The Body field must contain raw JSON only — no ` ```json ` code fence markers. Including markdown formatting causes a 400 BadRequest.

---

## Step 4 — Parse JSON

1. Click **+ New step** → search for **"Parse JSON"**
2. **Do not rename this step.** Leave the default name **"Parse JSON"**. The Compose expression in Step 5 references it as `body('Parse_JSON')`.
3. Configure:
   - **Content:** dynamic content → **Body** (from the HTTP step). In the dynamic content panel, look under the **"HTTP"** section header — not the "When a new email arrives" section. Both have a field named "Body". Selecting the email body here causes a schema mismatch at runtime and a silent failure.
   - **Schema:** paste this:

```json
{
  "type": "object",
  "properties": {
    "choices": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "message": {
            "type": "object",
            "properties": {
              "content": { "type": "string" }
            }
          }
        }
      }
    }
  }
}
```

---

## Step 5 — Extract Classification (Compose)

1. Click **+ New step** → search for **"Compose"** (under Data Operations)
2. In the **Inputs** field, click the expression tab (not dynamic content) and enter:

```
json(first(body('Parse_JSON')?['choices'])?['message']?['content'])
```

3. Click the step name → **Rename** → `Extract Classification`

This converts the GPT response string into a typed JSON object with `classification`, `reason`, and `suggested_action` fields.

> **Why Steps 6 and 7 don't reference this step's output:** You will notice that the Condition (Step 6) and the card body expressions (Step 7) both bypass `outputs('Extract_Classification')` and go directly to `body('HTTP')?['choices']?[0]?['message']?['content']`. This is intentional — the Compose output was found to be unreliable at the Condition evaluation point in some flow execution paths (documented in Key Decision #3 in ARCHITECTURE.md). Keep this step in the flow: it makes the data path readable and is the correct extension point if you want to add more branches based on the classification object later.

---

## Step 6 — Condition

1. Click **+ New step** → search for **"Condition"**
2. Configure with OR logic:

| Row | Left value (expression tab) | Operator | Right value |
|---|---|---|---|
| 1 | `body('HTTP')?['choices']?[0]?['message']?['content']` | contains | `Action Required` |
| 2 | `body('HTTP')?['choices']?[0]?['message']?['content']` | contains | `Urgent` |

Change the AND connector between rows to **OR**.

> **Note on expression choice:** Using `body('HTTP')?['choices']?[0]?['message']?['content'] contains "Action Required"` (checking the extracted content string) is more reliable than checking `outputs('Extract_Classification')?['classification']` at the condition point. The HTTP body path always resolves; the Compose output can fail to evaluate in some execution paths.

---

## Step 7 — True Branch: Teams Adaptive Card

In the **If yes** branch:

1. Click **Add an action** → search for **"HTTP"**
2. Configure:

**Method:** `POST`

**URI:** `TEAMS_WEBHOOK_URL` (the HTTP POST URL you copied in Step 0)

**Headers:**

| Key | Value |
|---|---|
| `Content-Type` | `application/json` |

**Body:**

```json
{
  "type": "message",
  "attachments": [{
    "contentType": "application/vnd.microsoft.card.adaptive",
    "content": {
      "type": "AdaptiveCard",
      "$schema": "http://adaptivecards.io/schemas/adaptive-card.json",
      "version": "1.2",
      "body": [
        {
          "type": "TextBlock",
          "text": "📧 Email Alert — ACTION REQUIRED",
          "weight": "Bolder",
          "size": "Medium",
          "color": "Attention"
        },
        {
          "type": "FactSet",
          "facts": [
            { "title": "From",             "value": "FROM_DYNAMIC" },
            { "title": "Subject",          "value": "SUBJECT_DYNAMIC" },
            { "title": "Classification",   "value": "CLASSIFICATION_DYNAMIC" },
            { "title": "Reason",           "value": "REASON_DYNAMIC" },
            { "title": "Suggested Action", "value": "SUGGESTED_ACTION_DYNAMIC" }
          ]
        }
      ]
    }
  }]
}
```

**How to replace the placeholders (two different methods):**

**FROM_DYNAMIC and SUBJECT_DYNAMIC — use the dynamic content panel:**
1. In the body text, select and delete `FROM_DYNAMIC` — keep the `"` quotes on both sides
2. With the cursor between the quotes, click **Add dynamic content** → under "When a new email arrives" → click **From**
3. Repeat for `SUBJECT_DYNAMIC` → select **Subject**

**CLASSIFICATION_DYNAMIC, REASON_DYNAMIC, SUGGESTED_ACTION_DYNAMIC — type `@{...}` expressions directly:**
1. Select and delete `CLASSIFICATION_DYNAMIC` — keep the `"` quotes
2. With the cursor between the quotes, type the expression directly into the text field (do not use the expression tab here — `@{...}` syntax works inline inside a text body):
   ```
   @{json(body('HTTP')?['choices']?[0]?['message']?['content'])?['classification']}
   ```
3. Repeat for `REASON_DYNAMIC`:
   ```
   @{json(body('HTTP')?['choices']?[0]?['message']?['content'])?['reason']}
   ```
4. Repeat for `SUGGESTED_ACTION_DYNAMIC`:
   ```
   @{json(body('HTTP')?['choices']?[0]?['message']?['content'])?['suggested_action']}
   ```

> **Inline vs. expression tab:** The expression tab (fx button) replaces the *entire* field with an expression result. `@{...}` typed directly inside a text field embeds an expression *within* a larger string. For this JSON body, always use the inline `@{...}` approach for the GPT fields.

---

## Step 8 — False Branch: Terminate

In the **If no** branch:

1. Click **Add an action** → search for **"Terminate"**
2. Set **Status** to `Succeeded`

This produces clean run history for low-priority emails — no warnings, no empty branches.

---

## Step 9 — Save and Test

1. Click **Save**
2. Click **Test** → **Manually** → **Test**

   > **What "Test → Manually → Test" does:** This puts the flow into a listening state for its next real trigger. It does not open a manual input form — you trigger the flow by sending an actual email. The Run History page will refresh automatically when the flow fires.

3. Send a test email **to the monitored inbox** from a different email address

   > **Which inbox is monitored:** The flow monitors the Outlook inbox of the Microsoft 365 account you used to sign in and configure the trigger. Send the test email to that address from a different account (e.g. a personal Gmail). Sending from and to the same account may not trigger the flow in all Power Automate configurations.

4. Use this subject line: `URGENT: Contract review needed by EOD`
5. Wait up to 30 seconds → check **Run History** for a green Succeeded entry
6. Check your Teams channel (Notification Trigger Agent) for the Adaptive Card

---

## Troubleshooting

| Symptom | Root cause | Fix |
|---|---|---|
| Can't find "Post your own adaptive card" action in Step 0 | It is a legacy action, hidden below newer Teams actions | Scroll down past "Post message in a chat or channel" in the Teams action list; search "adaptive card" to filter |
| Teams card renders blank / type error in Step 0 | Message field expects a JSON string, not an object | Change expression to `string(triggerBody()?['attachments']?[0]?['content'])` |
| 404 on Outlook trigger | Folder typed as "Inbox" string | Delete and re-select using the folder picker icon |
| 404 on HTTP step | `api-version=2024-02-15-preview` | Update URI to use `api-version=2025-01-01-preview` |
| 401 Unauthorized on HTTP step | Backtick characters in header names/values | Delete header rows, re-enter by typing directly |
| 400 BadRequest "Unexpected character: `" | Body field starts with ` ```json ` markdown fence | Remove markdown code fence, keep raw JSON only |
| Parse JSON fails with schema mismatch | Wrong "Body" selected in Step 4 — email body instead of HTTP response body | In the dynamic content panel, select Body under the **"HTTP"** section, not "When a new email arrives" |
| Condition always goes to False | Expression `outputs('Extract_Classification')?['classification']` not resolving | Switch condition left value to `body('HTTP')?['choices']?[0]?['message']?['content']` |
| Teams card shows literal expression text | Dynamic values not set as expressions | Re-enter classification/reason/suggested_action values using `@{...}` typed inline in the body text; do not use the expression tab for these |
| Flow runs but no Teams message | `TEAMS_WEBHOOK_URL` expired or wrong | Open the Teams Delivery Flow (Step 0), click the HTTP trigger block, copy the new HTTP POST URL, update the URI in Step 7 |
| Flow fires on test but Teams card never arrives | Step 0 Teams Delivery Flow is turned off | Go to make.powerautomate.com → My flows → Teams Delivery Flow → verify Status is On |
