# Camunda Agentic Insurance Claim Demo

A Camunda 8 demo showcasing how to build AI-powered agentic workflows for insurance claims processing. The processes use Camunda's Ad-hoc Subprocess pattern to give AI agents access to tools modelled directly as BPMN tasks, with human-in-the-loop oversight at every critical decision point.

---

## What This Demo Shows

### The Core Idea

Rather than building a traditional linear workflow, this demo uses **agentic Ad-hoc Subprocesses** — a Camunda 8 pattern where an LLM acts as an orchestrator, dynamically deciding which BPMN tasks (tools) to invoke and in what order based on the data it receives. The process model defines the available tools and the guard rails; the AI decides the execution path.

This demonstrates several patterns:

- **LLM-as-orchestrator**: The AI agent receives a claim and autonomously calls tools (customer lookups, policy queries, document requests, HTTP fetches) until it has enough information to present a report
- **Human-in-the-loop**: The clerk reviews the AI's report and makes the final approve/reject/more-info decision. The agent can also escalate to the clerk mid-process if it hits ambiguous cases
- **Email-driven async handoffs**: The process sends emails to customers and waits for replies (with attachments) using IMAP polling, bridging the gap between the synchronous process and asynchronous human responses
- **Document intelligence**: Uploaded or emailed documents are parsed by an LLM to extract structured identity data
- **Long-running claim monitoring**: Approved long-term injury claims transition into a second agentic loop that periodically contacts the claimant for condition reviews, with the ability to receive out-of-band case updates at any time

---

## Process Architecture

The demo is made up of three deployable processes and a set of backend service workers.

### 1. Insurance Claim Validation Agent (`InsuranceClaimValidationAgent`)

The main entry point. A claim arrives via a start form, and the process runs through two phases.

**Phase 1 — Claim Validation Agent (Ad-hoc Subprocess)**

Powered by **AWS Bedrock (Claude Haiku 4.5)**. The agent receives the submitted claim and has access to five tools:

| Tool | What it does |
|---|---|
| Get Customer Data | Looks up a customer by name or ID, returns profile and assigned DRI employee |
| Query for Insurance Policy | Searches for policies by customer, company, type, or status |
| Get Procedures and Requirements | Fetches the current validation rulebook via HTTP (from GitHub) |
| Request Information or Document | Emails the customer requesting a document or clarification; waits for an email reply with attachments |
| Ask For Clarifications | Raises a user task to the back-office clerk with a question |
| Show Final Report | Raises a user task presenting the full claim assessment in markdown; the clerk chooses to Approve, Reject, or Request More Info |

If the clerk rejects the claim (or cancels via the clarification task), an escalation event ends the process. Otherwise the process continues to Phase 2.

**Gateway — Claim Type**

After validation completes, the process splits on claim type:

- **Standard claims** (car, home, pet): Route to a `Run Payout Process` manual task and end
- **Work injury claims**: Route to the long-term monitoring path

**Summarize Claim Details**

Before the monitoring loop begins, a single-turn **OpenAI GPT-5.1** agent condenses everything the validation agent produced into a structured summary for the update agent to start from.

**Phase 2 — Claim Update Agent (Ad-hoc Subprocess)**

Also powered by **AWS Bedrock (Claude Haiku 4.5)**. This agent handles ongoing long-term injury case management. It has access to:

| Tool | What it does |
|---|---|
| Request Information or Document | Emails the claimant with periodic review questions |
| Calculate Wait time | Sets an ISO 8601 duration to wait before the next check-in |
| Show Update Report | Presents a condition update report to the clerk |
| Ask For Clarifications | Raises a question to the clerk |

After setting a wait duration the agent pauses at an event-based gateway, which can be interrupted by either a timer expiry or an inbound `CaseUpdate` message. The subprocess also listens for `InformationUpdate` messages at any time (non-interrupting), allowing external systems to push new case intelligence into the running agent context.

---

### 2. Request Document from Customer (`RequestKYCDocumentCustomer`)

A reusable subprocess called by both agent processes when they need to request or process a document.

**Flow:**

1. **Upload From Portal** start form — the customer's email address and the request body are received (pre-populated by the calling agent)
2. **Send Request for Document** — sends the request email via Gmail SMTP
3. **Event-based gateway** — waits for either:
   - An email reply in the `KYCDocumentRequest` Gmail folder (matched by `In-Reply-To` header), or
   - A 5-minute timeout (after which the process continues with a "no response" note)
4. If a reply arrives with attachments, **Get Data from Document** runs as a multi-instance task — one instance per attachment — using **OpenAI GPT-4o** to extract structured identity data (name, ID type, ID number, expiry date) from each document
5. **Create Response Variable** consolidates the email body and any extracted document data into a single result variable returned to the calling agent

---

### 3. Request Document Call Activity (`RequestDocumentCustomer`)

A thin wrapper call activity around `RequestKYCDocumentCustomer`, used when the document request subprocess is invoked as a standalone tool call from the agent context rather than directly as a call activity.

---

## Repository Structure

```
├── Insurance Claim Validation Agent/
│   ├── Insurance Claim Validation Agent.bpmn   # Main process
│   ├── Start Form.form                          # Claim intake form
│   ├── Ask For Clarifications.form              # Clerk clarification dialog
│   └── Show Final Report.form                  # Clerk approval/rejection form
│
├── Document Request Process/
│   ├── Request Document from Customer.bpmn     # Document request subprocess
│   ├── Request Document Call Activity.bpmn     # Wrapper call activity
│   ├── Request Documents Template.json         # Element template for the call activity
│   ├── Upload Form.form                        # Portal upload start form
│   ├── Validate Extract.form                   # Human document review form
│   └── Get Data Manually.form                  # Manual data entry fallback form
│
└── Connector Templates/
    ├── manage-customer-record.json             # Customer CRUD element template
    ├── manage-insurance-policy.json            # Policy CRUD/QUERY element template
    └── match-customer-with-dri.json            # Customer-to-DRI matching template
```

---

## Requirements

### Camunda 8 Environment

- **Camunda 8 SaaS** (recommended) or a self-managed Camunda 8.8+ cluster
- The **Agentic AI connector** (`io.camunda.agenticai:aiagent-job-worker:1` and `io.camunda.agenticai:aiagent:1`) must be available in your cluster — this is included in Camunda SaaS by default from 8.7+
- The **Email connector** (`io.camunda:email:1` and `io.camunda:connector-email-inbound:1`) must be available — included in SaaS by default

### AI Providers

You will need accounts and credentials for:

- **AWS** with Bedrock access enabled in your chosen region, and model access granted for `us.anthropic.claude-haiku-4-5-20251001-v1:0` in that region
- **OpenAI** with access to `gpt-4o` and `gpt-5.1`

### Email Account

A Gmail account is required for sending and receiving documents by email. An **App Password** must be generated (not the account's login password) — see the Gmail setup section below.

### Backend Service Workers

Three custom job worker types must be running and connected to your Camunda cluster. These handle the custom connector tasks:

| Job Type | Description |
|---|---|
| `match-customer-with-dri` | Looks up customers by name or ID and returns their assigned DRI employee |
| `manage-insurance-policy` | CRUD and QUERY operations on insurance policies |
| `manage-customer-record` | CRUD operations on customer records |

The element templates in `Connector Templates/` describe the expected input/output contract for each worker.

---

## Setup and Configuration

### Step 1 — Deploy the Processes

Deploy all three BPMN files and their associated form/template files to your Camunda cluster using the Camunda Modeler or the CLI. The connector templates in `Connector Templates/` should be imported into your Camunda Web Modeler organisation so they appear in the element templates panel.

Deploy in this order to avoid missing process references:

1. `Document Request Process/Request Document from Customer.bpmn`
2. `Document Request Process/Request Document Call Activity.bpmn`
3. `Insurance Claim Validation Agent/Insurance Claim Validation Agent.bpmn`

### Step 2 — Set Up Gmail

The email connector is configured to use Gmail with IMAP and SMTP. Standard Gmail passwords do not work with third-party SMTP/IMAP clients — you must use an **App Password**.

1. Enable 2-Step Verification on the Gmail account if not already enabled
2. Go to **Google Account → Security → 2-Step Verification → App passwords**
3. Generate an app password (select "Mail" as the app)
4. Store the 16-character password — this is your `EMAIL_PASSWORD_DMN` secret

The IMAP connector listens on a specific Gmail label/folder. You must create this folder in Gmail:

1. In Gmail, create a new label called **`KYCDocumentRequest`**
2. Optionally set up a filter to route relevant emails to this label automatically

The "from" address for outgoing emails is hardcoded as `shearsmithcamunda@gmail.com` in the BPMN. If you are using a different Gmail account, update the `data.smtpAction.from` input in the `Send Request for Document` service task in `Request Document from Customer.bpmn` before deploying.

### Step 3 — Enable AWS Bedrock Model Access

1. In the AWS Console, navigate to **Amazon Bedrock → Model access** in your target region
2. Request access to **Anthropic Claude Haiku** (specifically `claude-haiku-4-5-20251001`)
3. Note: the model ID used in the BPMN is `us.anthropic.claude-haiku-4-5-20251001-v1:0` — this is a cross-region inference profile. If your cluster's outbound traffic originates from a region without this profile available, switch to the direct regional model ID for your region

### Step 4 — Configure Secrets in Camunda

All secrets are managed via Camunda's built-in secrets store. In Camunda SaaS, go to **Console → Cluster → Connector Secrets**.

Create the following secrets exactly as named (the names are case-sensitive and must match precisely):

| Secret Name | Value | Notes |
|---|---|---|
| `AWS_REGION` | e.g. `us-east-1` | The AWS region where your Bedrock model access is enabled |
| `AWS_ACCESS_KEY` | Your IAM access key ID | Must be named exactly `AWS_ACCESS_KEY` — note there is a trailing-space typo in the BPMN (`AWS_ACCESS_KEY `) which you may also need to create as a secret named with a trailing space to work around |
| `AWS_SECRET_KEY` | Your IAM secret access key | |
| `OpenAI` | Your OpenAI API key | Starts with `sk-` |
| `EMAIL_USER_NAME_DMN` | The full Gmail address | e.g. `yourname@gmail.com` |
| `EMAIL_PASSWORD_DMN` | The Gmail App Password | The 16-character app password generated in Step 2 |

> **Note on `AWS_ACCESS_KEY`:** There is a trailing space inside the secret reference in the BPMN (`{{secrets.AWS_ACCESS_KEY }}`). Camunda resolves secrets by exact name match, so you either need to create a secret named `AWS_ACCESS_KEY ` (with a trailing space) to match the current BPMN, or fix the typo in the BPMN and redeploy. The simplest fix is to create both `AWS_ACCESS_KEY` and `AWS_ACCESS_KEY ` pointing to the same value.

### Step 5 — AWS IAM Permissions

The IAM user or role whose credentials you supply needs the following Bedrock permissions in the configured region:

```json
{
  "Effect": "Allow",
  "Action": [
    "bedrock:InvokeModel",
    "bedrock:InvokeModelWithResponseStream"
  ],
  "Resource": "arn:aws:bedrock:*::foundation-model/anthropic.claude-haiku-4-5-20251001-v1:0"
}
```

### Step 6 — Start the Backend Service Workers

The three custom job workers (`match-customer-with-dri`, `manage-insurance-policy`, `manage-customer-record`) must be connected to your cluster. These workers subscribe to their respective job types and handle the business logic for customer and policy data. Refer to your worker implementation for connection configuration (typically a Camunda client ID/secret and cluster address).

---

## Running the Demo

Once everything is deployed and configured:

1. Open **Camunda Tasklist** and start a new instance of **Insurance Claim Validation Agent**
2. Fill in the start form:
   - **Full Name** — the claimant's name (used by the agent to look up the customer record)
   - **Email** — the claimant's email address (used to send document requests)
   - **Claim Type** — select one of Car, Pet, Home, or Work Injury
   - **Details of the Claim** — free text description of the claim
3. The Claim Validation Agent will begin running. Depending on what it finds, it may:
   - Create user tasks in Tasklist asking for clerk clarifications
   - Send emails to the claimant requesting documents
   - Present a final report user task for the clerk to approve or reject
4. For **Work Injury** claims that are approved, the process continues into the long-term monitoring loop, which will periodically contact the claimant via email and surface update reports to the clerk

---

## Agentic Pattern Notes

### Ad-hoc Subprocess as Agent Container

The agent behaviour is modelled using Camunda's **Ad-hoc Subprocess** with `io.camunda.agenticai:aiagent-job-worker:1`. The subprocess contains all the tool tasks but does not define sequence flows between them — the agent job worker manages execution order by telling Camunda which tool to activate next based on the LLM's response.

### `fromAi()` Input Bindings

Inside the ad-hoc subprocess, tool task inputs use the `fromAi()` FEEL function. This is how the agent job worker injects the LLM-chosen parameter values into each tool invocation at runtime. The string descriptions inside `fromAi()` calls are the parameter descriptions sent to the model as part of the tool schema.

### Tool Call Results

Each tool task maps its output to `toolCallResult`. The ad-hoc subprocess collects all tool results into `toolCallResults` (an array), which is fed back to the LLM on the next iteration so it can reason about what it has learned so far.

### Context Window

Both agent subprocesses are configured with a context window of 20 messages and a maximum of 10 model calls per activation. These are configurable in the BPMN `ioMapping` inputs (`data.memory.contextWindowSize` and `data.limits.maxModelCalls`).
