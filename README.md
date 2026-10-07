
# 🚀 Automated API Edge-Case Fuzzer & QA Pipeline

An enterprise-grade, event-driven API QA fuzzer and automated alerting pipeline built with **n8n**, **Node.js/Express**, **MongoDB**, **ngrok**, **Discord Webhooks**, and the **GitHub REST API**.

This repository demonstrates an end-to-end DevOps testing pipeline that automatically mutates API parameters, executes fuzzing payloads against local REST endpoints via secure tunneling, evaluates response statuses, auto-generates GitHub bug tickets for `500` crashes, and posts consolidated execution summaries to Discord.

---

## 📐 System Architecture

```text
               ┌─────────────────────────────────────────────────────────────┐
               │                         n8n Cloud                           │
               └──────────────────────────────┬──────────────────────────────┘
                                              │
                                   1. Generate Test Payloads
                                              │
                                              ▼
                                 ┌─────────────────────────┐
                                 │ HTTP Request (POST API) │
                                 └────────────┬────────────┘
                                              │
                                2. Secure ngrok Public Tunnel
                                              │
                                              ▼
                                  ┌───────────────────────┐
                                  │ Express REST API      │ (Port 5000)
                                  │ MongoDB (employee_db) │
                                  └───────────┬───────────┘
                                              │
                                     3. Status Code Return
                                              │
                     ┌────────────────────────┴────────────────────────┐
                     │                                                 │
                     ▼                                                 ▼
          [Status == 500 Crash]                            [Status == 200/201/400]
                     │                                                 │
          ┌──────────┴──────────┐                          ┌───────────┴───────────┐
          │                     │                          │ Switch Node Routing   │
          ▼                     ▼                          └───────────┬───────────┘
   ┌──────────────┐    ┌─────────────────┐                             │
   │ Discord      │    │ GitHub REST API │                   ┌─────────┼─────────┐
   │ Crash Alert  │    │ Create Issue    │                   ▼         ▼         ▼
   └──────────────┘    └─────────────────┘                (200/201)  (400)   (Fallback)
                                                          Success   Handled   Alert
                                                          
                     └────────────────────────┬────────────────────────┘
                                              │
                                    4. Parallel Aggregation
                                              │
                                              ▼
                                ┌───────────────────────────┐
                                │ Test Suite Summary Generator│
                                └─────────────┬─────────────┘
                                              │
                                              ▼
                                 ┌─────────────────────────┐
                                 │ Discord Webhook Summary │
                                 └─────────────────────────┘

```

---

## ✨ Features

* **Dynamic Payload Mutation:** Generates happy path, type mismatch, and null/boundary fuzzer payloads programmatically with timestamped parameters to prevent duplicate database key conflicts.
* **Edge Tunnel Integration:** Exposes local microservices to cloud orchestration engines using ngrok edge tunneling.
* **Intelligent Error Triage:**
* **`200 / 201 Created`**: Evaluated as successful execution.
* **`400 / 422 Bad Request`**: Evaluated as expected schema/validation handling.
* **`500 Internal Server Error`**: Identified as an unhandled application crash.
* **Fallback Handling**: Routes unexpected status codes (`401`, `404`, `502`) to warning channels.


* **Automated GitHub Issue Generation:** Automatically files a detailed bug ticket on GitHub for `500` crashes, including the mutated JSON payload and backend stack trace.
* **Consolidated Discord Summary:** Compiles test suite metrics into a single digest report sent via Webhook upon completion.

---

## 🛠 Tech Stack

* **Backend:** Node.js, Express.js, Mongoose, MongoDB (`employee_db`)
* **Orchestration & Testing:** n8n (Workflow Automation Engine)
* **Tunneling:** ngrok Tunnel Client
* **Integrations:** GitHub REST API v3, Discord Webhooks API

---

## 📋 API Specification

### POST `/api/employees`

Creates a new employee record.

**Request Headers:**

```http
Content-Type: application/json

```

**Payload Schema:**

```json
{
  "name": "String (Required)",
  "email": "String (Required, Unique)",
  "age": "Number",
  "department": "String",
  "salary": "Number"
}

```

**Known Test Vulnerability:**
The endpoint includes an unhandled `.trim()` method call on the `name` parameter without checking for null values:

```javascript
// Intentional flaw triggered by { "name": null }
const formattedName = req.body.name.trim(); 

```

Passing `{ "name": null }` triggers an unhandled `TypeError: Cannot read properties of null (reading 'trim')`, returning an HTTP `500 Internal Server Error`.

---

## 🚀 Getting Started (Developer Setup)

### Prerequisites

* Node.js (v16+) and npm installed
* MongoDB installed locally and running on default port `27017`
* An active ngrok account
* An n8n instance (Self-hosted or Cloud)

---

### 1. Backend Application Setup

Clone the repository and install dependencies:

```bash
git clone [https://github.com/sreevarshineb-afk/Employee_dashboard.git](https://github.com/sreevarshineb-afk/Employee_dashboard.git)
cd Employee_dashboard
npm install

```

Ensure local MongoDB is active:

```bash
mongosh --eval "db.version()"

```

Start the Express backend server:

```bash
node server.js
# Expected output: Server running on port 5000 | MongoDB Connected: employee_db

```

---

### 2. ngrok Edge Tunnel Setup

Expose your local Express server (port 5000) to the internet:

```bash
ngrok http 5000

```

Copy the generated forwarding URL (e.g., `https://garlic-retreat-provided.ngrok-free.dev`).

---

### 3. n8n Automation Workflow Setup

1. Open your n8n workspace.
2. Click **Import from File** and select `workflows/qa-fuzzer-workflow.json`.
3. Update environment variables and credentials inside the nodes:
* **HTTP Request (Target Executor):** Set URL to `<YOUR_NGROK_URL>/api/employees`.
* **GitHub Node:** Select your GitHub Credential (requires Classic PAT with `repo` scope or Fine-Grained token with `Issues: Read and Write`). Set Owner to `sreevarshineb-afk` and Repo to `Employee_dashboard`.
* **Discord Webhook Nodes:** Set URL to your Discord Channel Webhook.



---

## 🧪 Fuzzer Payload Test Matrix

The `Code in JavaScript` generator node fires three distinct test cases per run:

| Test Case | Payload Modification | Expected HTTP Status | Expected Workflow Action |
| --- | --- | --- | --- |
| **Happy Path** | Valid schema payload with unique email | `201 Created` | Routed through Switch Output 1 (Passed quietly) |
| **Type Mismatch** | `"age": "thirty"` (String instead of Number) | `400 Bad Request` | Routed through Switch Output 2 (Validation caught) |
| **Null Name Fuzzer** | `"name": null` (Triggers backend `.trim()` bug) | `500 Internal Error` | Triggers Discord Crash Alert & Auto-creates GitHub Issue |

---

## 📊 Summary Execution Report Example

Upon completion, the parallel aggregator posts the following summary digest to Discord:

```text
📊 QA Test Suite Execution Summary
----------------------------------------
• Total Tests Executed: 3
• Passed (Happy Path): 1
• Graceful Validations Handled: 1
• Unhandled Server Crashes (500): 1
• Unexpected Status Codes: 0
• Suite Result: ⚠️ FAILED (CRASHES DETECTED)

```
