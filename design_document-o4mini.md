# Email AI Auto‐Responder v2  
## Detailed Technical Design Document

**Version**: 1.0  
**Author**: [Your Name]  
**Date**: 2024-06-27  

---

## Table of Contents

1. Executive Summary  
2. Prerequisites  
3. Introduction to n8n Concepts  
4. High-Level Workflow Overview  
5. Node-by-Node Detailed Explanation  
   5.1 Email Trigger (IMAP)  
   5.2 HTML → Markdown Conversion  
   5.3 Email Classification  
   5.4 Conditional Routing  
   5.5 Embedding Creation  
   5.6 Knowledge Retrieval (Qdrant)  
   5.7 Prompt Assembly  
   5.8 Response Generation  
   5.9 Send Reply  
   5.10 Error Handling & Notification  
6. Connections & Data Flow  
7. Configuration & Environment Variables  
8. Error Handling, Monitoring & Alerts  
9. Extending & Adapting to New Use Cases  
10. Appendix & References  

---

## 1. Executive Summary (≈300 words)

This document describes the **Email AI Auto-Responder v2**, an n8n workflow that:

- Reads incoming emails via IMAP  
- Converts HTML content to Markdown for better LLM ingestion  
- Classifies the email as **Company Info** or **Other**  
- Retrieves relevant company knowledge from a Qdrant vector store  
- Generates a concise, professional reply (≤100 words) using OpenAI  
- Sends the reply via SMTP  
- Routes non-“Company Info” messages to an archive branch  
- Alerts the operations team on errors via Slack  

This design emphasizes **modularity**, **scalability**, and **maintainability**. Each logical stage is implemented in its own set of nodes. Configuration is driven by environment variables so the workflow can be easily adapted to new use cases (e.g., support ticket responses, meeting scheduling, or multilingual replies).

Whether you are new to n8n or an experienced automation engineer, this document will guide you through every aspect of the solution, explain each node’s purpose, and show you how to modify the workflow to suit your own requirements.

---

## 2. Prerequisites (≈300 words)

Before deploying or modifying this workflow, ensure you have:

1. **n8n installed and running**  
   - Docker, Docker-Compose, or npm installation  
   - Access to the n8n Editor UI  

2. **Credentials**  
   - **IMAP account** for reading emails  
   - **SMTP account** for sending replies  
   - **OpenAI API key** (or compatible router)  
   - **Qdrant API key** and an existing **collection** for company knowledge  
   - **Slack webhook or API token** (optional, for notifications)

3. **Environment Variables**  
   - `COLLECTION_NAME` – your Qdrant collection name  
   - (Optional) other keys for Google Drive, Twilio, etc., if extending

4. **Basic knowledge of JavaScript expressions**  
   - n8n uses `={{ ... }}` syntax to reference node outputs, environment variables, or simple JS transformations  

5. **Familiarity with LLM concepts**  
   - System versus user prompts  
   - Embeddings and vector similarity retrieval  

---

## 3. Introduction to n8n Concepts (≈500 words)

**n8n** is a low-code workflow automation tool that lets you connect apps and APIs visually. Key constructs:

- **Nodes**: Building blocks that perform tasks (e.g., HTTP Request, Email Read, AI Chain)  
- **Credentials**: Securely stored authentication details used by nodes  
- **Connections**: Wires between nodes indicating data flow  
- **Expressions**: JavaScript inside `={{ }}` to reference data from other nodes (`$node['Name'].json.property`)  
- **Workflows**: Configurations of nodes and connections that run on triggers  
- **Sub-workflows**: Called via the “Execute Workflow” node for modularity  

Nodes have **parameters**, **credentials**, and can expose **outputs** in JSON or binary.

### Best Practices

- **Name nodes descriptively** (e.g. “Email Classifier” rather than “Node 3”)  
- **Group related nodes visually**  
- **Use environment variables** for URLs, tokens, collection names  
- **Add Sticky Notes** (available in the UI) to document TODOs or reminders  
- **Error triggers** and **notification channels** for robust monitoring  

---

## 4. High-Level Workflow Overview (≈500 words)

Below is the stepwise flow:

1. **Email Trigger**  
   - Polls IMAP for unread messages → marks them read  

2. **HTML → Markdown**  
   - Converts email body to Markdown for clarity  

3. **Email Classification**  
   - Uses LangChain’s `textClassifier` to tag as “Company Info” or “Other”  

4. **Conditional Routing**  
   - If “Other” → `NoOp` (archive or manual review)  
   - If “Company Info” → proceed  

5. **Embedding Generation**  
   - Creates an embedding vector from the email text  

6. **Qdrant Retrieval**  
   - Retrieves top-k related documents from the company knowledge base  

7. **Prompt Assembly**  
   - Merges the retrieved context and original email into a structured prompt  

8. **Generate Reply**  
   - Uses `chainLlm` node with system + user messages → returns HTML reply  

9. **Send Reply**  
   - Sends the generated HTML via SMTP, preserving original “From”/“To”  

10. **Error Handling & Notification**  
    - An `errorTrigger` node catches any failures → posts to Slack  

**Diagram**  

```text
[Email Trigger] → [HTML→Markdown] → [Classifier] →| Company Info  
                                               ↳ [Archive NoOp]

→ [Embedding] → [Qdrant Retrieve] → [Merge Context] → [Generate Reply] → [Send Email]
                                                           ↳ [errorTrigger] → [Notify Ops]
```

---

## 5. Node-by-Node Detailed Explanation (≈2000 words)

Below is a breakdown of each node in the workflow. All JSON snippets are partial; in n8n you will import the full workflow JSON provided earlier.

### 5.1 Email Trigger (IMAP)

```jsonc
{
  "name": "Email Trigger (IMAP)",
  "type": "n8n-nodes-base.emailReadImap",
  "typeVersion": 2,
  "position": [-300, 0],
  "parameters": {
    "options": { "unseen": true },   // Only fetch unread
    "markSeen": true                  // Mark message read to prevent duplicates
  },
  "credentials": {
    "imap": { "id": "YOUR_IMAP_CREDENTIAL_ID" }
  }
}
```

**Purpose**  
- Poll an IMAP mailbox at defined intervals (in n8n settings you control polling frequency).  
- Only “unseen” mails are processed; after reading, they are marked “seen.”  
- Outputs JSON payload containing fields like `from`, `to`, `subject`, `textHtml`, `textPlain`, `attachments`, etc.

**Customizations**  
- Change `unseen` to `false` + `fetchAll` if you need to reprocess historical data.  
- Add filters (e.g., “from” contains certain domain).

---

### 5.2 HTML → Markdown Conversion

```jsonc
{
  "name": "HTML → Markdown",
  "type": "n8n-nodes-base.markdown",
  "typeVersion": 1,
  "position": [0, 0],
  "parameters": {
    "html": "={{ $json.textHtml }}"  // Pull HTML body from trigger
  }
}
```

**Purpose**  
- Convert the HTML email body (`textHtml`) into Markdown.  
- Markdown is generally simpler for LLM prompt reading and summarization.

**Output**  
- `parsed` property in the node’s JSON holds the Markdown text.

---

### 5.3 Email Classification

```jsonc
{
  "name": "Email Classifier",
  "type": "@n8n/n8n-nodes-langchain.textClassifier",
  "typeVersion": 1,
  "position": [300, 0],
  "parameters": {
    "inputText": "=Here is the email:\n\n{{$node['HTML → Markdown'].json.parsed}}",
    "options": {
      "fallback": "Other",
      "multiClass": false,
      "enableAutoFixing": true,
      "systemPromptTemplate": "Please classify the text provided by the user into one of the following categories: {categories}. Output only the JSON, do not explain."
    },
    "categories": {
      "categories": [
        { "category": "Company Info", "description": "General info requests" },
        { "category": "Other",        "description": "All other emails" }
      ]
    }
  }
}
```

**Purpose**  
- Determines whether an incoming email is a request for company information or something else.  
- Reduces LLM API usage by filtering out irrelevant emails.

**Key Parameters**  
- `fallback`: default category if classifier is uncertain.  
- `multiClass:false`: forces a single label.  
- `enableAutoFixing`: allows minor prompt/template fixes by the node.

---

### 5.4 Conditional Routing

```jsonc
{
  "name": "Route Email",
  "type": "n8n-nodes-base.if",
  "typeVersion": 1,
  "position": [600, 0],
  "parameters": {
    "conditions": {
      "boolean": [
        { "value1": "={{ $json.category === 'Company Info' }}" }
      ]
    }
  }
}
```

**Purpose**  
- A simple “If” node that checks the classifier’s `category`.  
- **True branch** → proceed with AI reply.  
- **False branch** → route to archive or manual processing (`NoOp` node).

---

### 5.5 Archive Non-Company Emails (NoOp)

```jsonc
{
  "name": "Archive Non-Company Emails",
  "type": "n8n-nodes-base.noOp",
  "typeVersion": 1,
  "position": [900, -100]
}
```

**Purpose**  
- Placeholder for future expansion (e.g., move to Gmail label, Slack alert).  
- Keeps the false branch from falling through to the AI reply logic.

---

### 5.6 Create Embedding

```jsonc
{
  "name": "Create Embedding",
  "type": "@n8n/n8n-nodes-langchain.embeddingsOpenAi",
  "typeVersion": 1.2,
  "position": [900, 50],
  "parameters": { "options": {} },
  "credentials": {
    "openAiApi": { "id": "YOUR_OPENAI_CREDENTIAL_ID" }
  }
}
```

**Purpose**  
- Converts the Markdown email text into an embedding vector.  
- The embedding is used to retrieve similar documents from the Qdrant vector store.

**Input**  
- Defaults to using the incoming JSON data field; if you need custom text, set `text: ={{ ... }}`.

---

### 5.7 Retrieve Context (Qdrant)

```jsonc
{
  "name": "Retrieve Context",
  "type": "@n8n/n8n-nodes-langchain.vectorStoreQdrant",
  "typeVersion": 1,
  "position": [1200, 50],
  "parameters": {
    "mode": "retrieve-as-tool",
    "toolName": "company_knowledge_base",
    "toolDescription": "Fetch relevant company information.",
    "qdrantCollection": {
      "__rl": true,
      "mode": "id",
      "value": "={{ $env.COLLECTION_NAME }}"
    },
    "includeDocumentMetadata": false
  },
  "credentials": {
    "qdrantApi": { "id": "YOUR_QDRANT_CREDENTIAL_ID" }
  }
}
```

**Purpose**  
- Queries your Qdrant collection using the embedding from the prior node.  
- Returns an array of the top-k similar documents, which will form the “knowledge context.”

**Customization**  
- You can adjust `topK`, filters, or include metadata.  
- Set your collection via environment variable to keep code portable.

---

### 5.8 Assemble Prompt

```jsonc
{
  "name": "Assemble Prompt",
  "type": "n8n-nodes-base.merge",
  "typeVersion": 1,
  "position": [1500, 50],
  "parameters": {
    "mode": "mergeByIndex",
    "propertyName": "context"
  }
}
```

**Purpose**  
- Merges the two incoming streams:  
  1. Original email Markdown  
  2. Array of retrieved documents  
- Result is a single JSON object:
  ```json
  {
    "parsed": "Original Markdown text…",
    "context": ["Doc1 text", "Doc2 text", ...]
  }
  ```

---

### 5.9 Generate Reply

```jsonc
{
  "name": "Generate Reply",
  "type": "@n8n/n8n-nodes-langchain.chainLlm",
  "typeVersion": 1.5,
  "position": [1800, 50],
  "parameters": {
    "promptType": "define",
    "messages": {
      "messageValues": [
        {
          "role": "system",
          "message": "You are a concise business assistant. Respond in professional HTML, max 100 words."
        },
        {
          "role": "system",
          "message": "Context:\n\n{{ $node['Retrieve Context'].json.documents.join('\\n\\n') }}"
        },
        {
          "role": "user",
          "message": "Reply to this email:\n\n{{ $node['HTML → Markdown'].json.parsed }}"
        }
      ]
    },
    "outputFormat": "html"
  }
}
```

**Purpose**  
- Sends a multi-message chat prompt to OpenAI (or another LLM).  
- The first two messages set the system instructions (tone, formatting, word limit, context).  
- The user message contains the email content.  
- The node returns legitimate HTML text for direct email sending.

---

### 5.10 Send Reply

```jsonc
{
  "name": "Send Reply",
  "type": "n8n-nodes-base.emailSend",
  "typeVersion": 2.1,
  "position": [2100, 50],
  "parameters": {
    "toEmail": "={{ $node['Email Trigger (IMAP)'].json.from }}",
    "fromEmail": "={{ $node['Email Trigger (IMAP)'].json.to }}",
    "subject": "={{ `Re: ${$node['Email Trigger (IMAP)'].json.subject}` }}",
    "html": "={{ $node['Generate Reply'].json.text }}"
  },
  "credentials": {
    "smtp": { "id": "YOUR_SMTP_CREDENTIAL_ID" }
  }
}
```

**Purpose**  
- Uses SMTP to send the HTML reply back to the original sender.  
- Preserves original “To” and “From” addresses.  
- Pre-pends “Re:” to the subject line.

---

### 5.11 Error Handling & Notification

```jsonc
[
  {
    "name": "Error Handler",
    "type": "n8n-nodes-base.errorTrigger",
    "typeVersion": 1,
    "position": [300, 300]
  },
  {
    "name": "Notify Ops (Slack)",
    "type": "n8n-nodes-base.slack",
    "typeVersion": 1,
    "position": [600, 300],
    "parameters": {
      "text": "🚨 *Error in Email AI Auto-Responder v2:* {{$json.error.message}}",
      "channel": "#alerts"
    },
    "credentials": {
      "slackApi": { "id": "YOUR_SLACK_CREDENTIAL_ID" }
    }
  }
]
```

**Purpose**  
- When any node in the workflow errors out, the `Error Handler` catches it.  
- The Slack node posts a concise alert to your channel with the error message.  

**Customizations**  
- You can swap Slack for Email, Microsoft Teams, or other tools.  
- Add error‐specific branching if you need specialized recovery steps.

---

## 6. Connections & Data Flow (≈500 words)

The `connections` section of the workflow JSON defines how data moves between nodes. Here is an illustrative excerpt:

```jsonc
"connections": {
  "Email Trigger (IMAP)": {
    "main": [[{ "node": "HTML → Markdown", "type": "main", "index": 0 }]]
  },
  "HTML → Markdown": {
    "main": [[{ "node": "Email Classifier", "type": "main", "index": 0 }]]
  },
  "Email Classifier": {
    "main": [[{ "node": "Route Email", "type": "main", "index": 0 }]]
  },
  "Route Email": {
    "main": [
      [{ "node": "Create Embedding", "type": "main", "index": 0 }],
      [{ "node": "Archive Non-Company Emails", "type": "main", "index": 1 }]
    ]
  },
  "Create Embedding": {
    "main": [[{ "node": "Retrieve Context", "type": "main", "index": 0 }]]
  },
  "Retrieve Context": {
    "main": [[{ "node": "Assemble Prompt", "type": "main", "index": 0 }]]
  },
  "Assemble Prompt": {
    "main": [[{ "node": "Generate Reply", "type": "main", "index": 0 }]]
  },
  "Generate Reply": {
    "main": [[{ "node": "Send Reply", "type": "main", "index": 0 }]]
  },
  "Error Handler": {
    "main": [[{ "node": "Notify Ops (Slack)", "type": "main", "index": 0 }]]
  }
}
```

### Key Points

- **Branching** at the “Route Email” node:  
  - Index 0 (true) → AI reply flow  
  - Index 1 (false) → archive branch
- **Linear chain** from embedding → retrieval → prompt → reply → send  
- **Dedicated error path** at the bottom to capture any exceptions  

---

## 7. Configuration & Environment Variables (≈400 words)

To keep your workflow portable:

1. **n8n.env file** or UI Settings → **Environment Variables**:

   ```
   COLLECTION_NAME=company_info_v1
   ```

2. In the **Retrieve Context** Qdrant node:

   ```jsonc
   "qdrantCollection": {
     "__rl": true,
     "mode": "id",
     "value": "={{ $env.COLLECTION_NAME }}"
   }
   ```

3. **Credentials** are defined centrally in the n8n UI under “Credentials.” Use the UI to create:

   - IMAP  
   - SMTP  
   - OpenAI API  
   - Qdrant API  
   - Slack API  

4. **Polling Interval** for your IMAP trigger: adjust in the node’s “Settings” tab (e.g., every 1 minute).

---

## 8. Error Handling, Monitoring & Alerts (≈400 words)

**Error Trigger Node**  
- Catches exceptions from any downstream node.  
- Can be scoped (only certain branches) or global.

**Notification**  
- Posts to Slack channel `#alerts`.  
- Message: “🚨 *Error in Email AI Auto-Responder v2:* <error message>”

**Recommendation**  
- Use **n8n’s built-in logs** for debugging.  
- Combine with **external monitoring** (Datadog, Prometheus) if at scale.  
- Add a **dead-letter queue** or database table for failed emails, to retry later.

---

## 9. Extending & Adapting to New Scenarios (≈600 words)

This modular design makes it easy to repurpose:

1. **Change Classifier Categories**  
   - Support tickets vs. sales inquiries vs. personal  
   - Update the `categories` block in the “Email Classifier” node  

2. **Alternate Knowledge Sources**  
   - MongoDB, Elasticsearch, Airtable instead of Qdrant  
   - Swap the retrieval node accordingly  

3. **Multilingual Support**  
   - Add a `Language Detection` node (e.g. Google Translate detect)  
   - Route to different LLM models/prompts per language  

4. **Attachment Processing**  
   - Insert a parallel branch after the IMAP node  
   - Use `SplitInBatches` + `httpRequest` or specialized parsers to extract PDF text  

5. **Alternative Reply Styles**  
   - For technical inquiries, use a different system prompt (e.g., “You are a technical support engineer.”)  
   - Dynamically select prompt via an `IF` node or a small JS Function  

6. **Scheduling & Rate Limits**  
   - Add `Wait` nodes to throttle sending  
   - Enforce business hours only—use `Cron` node + `IF`  

7. **Logging & Auditing**  
   - Insert a database or Google Sheets node after “Send Reply” to log:  
     - Email metadata  
     - Prompt + response  
     - Timestamp  

8. **Human-in-the-Loop**  
   - Replace “Send Reply” with “Create Slack message + Approve”  
   - Use `Slack node → Button interaction → Webhook` to capture approval, then send  

By reorganizing your workflow into sub-workflows and naming each node descriptively, you can easily duplicate, modify, or remove sections without affecting the rest of the pipeline.

---

## 10. Appendix & References (≈500 words)

### Full JSON Import

- Refer back to the complete JSON block provided earlier in this document.  
- Import via **Settings → Workflows → Import from clipboard** in the n8n Editor.

### Useful Links

- n8n Documentation:  
  https://docs.n8n.io  
- LangChain Nodes in n8n:  
  https://n8n-community.github.io  
- Qdrant Documentation:  
  https://qdrant.tech/documentation  
- OpenAI API Reference:  
  https://beta.openai.com/docs  

### Contact & Support

For questions or issues:

- Create an issue in your team’s GitHub repo  
- Drop a message in your internal Slack #n8n-automation channel  

---

**Congratulations!** You now have a clear, modular, and production-ready design for an AI-driven email auto-responder in n8n. Use this guide to customize prompts, switch out integrations, and scale your automation to new domains. 🚀
