# Multiagent_n8n_InsuranceBot_CIRA
RoadSure CIRA is an n8n-powered AI auto-insurance assistant that answers FAQs via Postgres/pgvector, collects quote info, returns 3 tiered options, supports human escalation, enforces &lt;3s SLA with “please wait” acks, and logs every turn to Postgres for a full audit trail.


# RoadSure CIRA – AI Auto-Insurance Intake & Quote Agent

## 1. Overview

**Project name:** RoadSure CIRA  
**Stack:** n8n (chat workflow), OpenAI (chat + embeddings), Postgres + pgvector, email via n8n  

RoadSure CIRA is an AI assistant for a fictional auto-insurance company. It:

- Answers policy & product questions from a vector knowledge base  
- Collects industry-standard data to generate multi-option quotes  
- Lets users select a quote and notifies stakeholders via email  
- Supports human escalation at any time  
- Enforces a < 3 second response SLA with an instant acknowledgement path  
- Logs every user and agent turn into Postgres for analytics and auditing  

All of this is implemented in a single n8n workflow using built-in nodes, OpenAI tools, and a Postgres pgvector store.

---

## 2. Problem & Goals

Typical insurance chatbots either:

- Only handle FAQs, or  
- Hard-code rigid forms with no logging or SLA awareness.

The hackathon brief asked for an AI agent that:

- Responds in under 3 seconds per turn, with fallback “please wait” if tools are slow  
- Answers basic insurance questions grounded in company content  
- Generates multi-option quotes using realistic input fields  
- Lets the user choose a quote and notifies the business  
- Allows human escalation at any time  
- Never pretends to be human and stays on-topic  

RoadSure CIRA aims to satisfy all of these while also demonstrating:

- Production-style logging (conversation + quotes)  
- Clear separation between routing, tools, and UI responses  
- A workflow that a real insurance team could extend further  

---

## 3. User Journey

1. User says “hi” or asks a question.  
2. **Chat Trigger → Normalize → Initial-Agent (OpenAI chat).**  
3. Agent classifies the turn into **Fast | Quote | Human | All**.

### Fast (FAQ / simple in-scope)

- If only a quick KB answer is needed, the agent either:
  - Answers directly via vector search, or  
  - Uses the fallback agent for richer explanation.
- Response is short, no role-play, and always acknowledges it’s AI.

### Quote flow

When the user asks for a quote (“give me a quote”, “price my car”, etc.), the agent:

- Collects structured fields: name, email, date of birth, VIN, license issue year, desired deductible, incidents count, annual mileage, etc.  
- Stores data in Postgres (`cust_info_full`, `mini_cust_info`, etc.).  
- Computes 3 quote bands (Bronze / Silver / Gold style) with premiums, coverage, and deductibles.

When the user selects a quote:

- Selection is stored into `quote_register`.  
- An email is sent to stakeholders with session id, customer info, and chosen quote.

### Human escalation

At any time the user can request a human (“talk to a human”, “agent”, “call me”, etc.). The workflow:

1. Checks DB to see if the customer already exists.  
2. If not, asks for minimal contact fields.  
3. Writes an escalation row to DB and sends an email to the business.  
4. Shows the user a confirmation message.

### SLA & “please wait” acknowledgements

For any path that hits pgvector or long DB work, the workflow calls a **Respond to Chat** node *before* heavier tools:

> “I’m looking that up in our knowledge base, one moment while I fetch the details.”

This guarantees something reaches the user in < 3 seconds even if the vector store or downstream nodes are slow.

### Logging & analytics

Every turn is logged into `roadsure_final.interaction_log` with:

- `session_id`  
- `turn_index`  
- `actor` (`user` / `agent`)  
- `agent_phase` (e.g., `Initial-Agent:input`, `Initial-Agent:reply`, `fall_agent`)  
- `message`  
- `category` (Fast / Quote / Human / All)  
- `confidence`  
- `needs_escalation`  
- `created_at`  

Additional tables store customer info and quotes.

---

## 4. Architecture & Workflow

### Core components

- **n8n Chat Trigger** – entry point for each message (Response Mode: “Using response nodes”)  
- **Normalize-1 / Normalize-2** – clean inputs and parse agent JSON output  
- **Initial-Agent (OpenAI Chat Model)**  
  - System prompt enforces:
    - AI identity & scope (RoadSure + auto insurance only)  
    - Output schema: `answer`, `category`, `confidence`, `needs_escalation`  
    - SLA behavior & categories:
      - `Fast`: answer immediately  
      - `Quote`: start quote workflow  
      - `Human`: escalate  
      - `All`: send to fallback agent  

- **Postgres + pgvector + OpenAI embeddings** – stores RoadSure policy handbook content for RAG.  
- **Fallback Agent (“All” path)** – higher-fidelity answerer with more detailed, cited explanations.  
- **Quote path**  
  - Forms / questions to collect rating fields  
  - Writes into Postgres tables: `cust_info_full`, `mini_cust_info`, `quote_register`, etc.  
- **Email nodes** – one for quote selection notification, one for human escalation notification.  
- **Respond to Chat nodes**  
  - A simple instant ACK used before heavy vector/DB work  
  - Final response nodes for Fast / Quote / Human / All paths.

### SLA enforcement pattern

```text
Chat Trigger → Normalize → (optional) Respond to Chat ACK → tools (pgvector, DB, etc.) → Normalize-2 → Respond to Chat (final answer)


If tools exceed 3 seconds, the user still gets the ACK message within SLA.

---

## 5. Data Model (Postgres)

**Schema:** `roadsure_final`

**Key tables:**

- `cust_info_full` – full customer details for quoting (name, email, phone, DOB, VIN, license year, deductible, incidents, annual mileage, etc.).  
- `mini_cust_info` – minimal subset for quick human escalations.  
- `quote_register` – one row per generated quote selection (session id, chosen tier, premium, coverage summary, timestamp).  
- `interaction_log` – full conversation log:

```sql
CREATE TABLE roadsure_final.interaction_log (
  id               uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  session_id       text    NOT NULL,
  turn_index       integer NOT NULL,
  actor            text    NOT NULL, -- 'user' | 'agent'
  agent_phase      text    NOT NULL,
  message          text    NOT NULL,
  category         text,
  confidence       numeric,
  needs_escalation boolean,
  created_at       timestamptz NOT NULL DEFAULT now()
);

Additional helper tables like session_logger_table and total_schema_table support aggregations and reporting.


6. How to Run (Local Dev)
Prerequisites

Docker & Docker Compose

OpenAI API key

n8n workflow JSON (Main.json in this repo)

6.1 Run Postgres with pgvector
docker run --name pg \
  -e POSTGRES_USER=n8n \
  -e POSTGRES_PASSWORD=pass \
  -e POSTGRES_DB=roadsure \
  -p 5432:5432 \
  pgvector/pgvector:pg16


docker run -it --name n8n \
  -p 5678:5678 \
  -e N8N_HOST=localhost \
  -e N8N_PORT=5678 \
  -e N8N_PROTOCOL=http \
  -e OPENAI_API_KEY=YOUR_KEY \
  n8nio/n8n:1.122.5



6.3 Configure Postgres credentials in n8n

Host: host.docker.internal (Mac) or container name pg on a shared Docker network

Port: 5432

DB: roadsure

User: n8n

Password: n8n_pw

SSL: disabled for local dev

6.4 Run DB migration

Use an n8n Postgres → Execute a SQL Query node (or psql) to create:

cust_info_full

mini_cust_info

quote_register

interaction_log

any helper tables

(Full SQL is included in this repo in a separate .sql file.)

6.5 Import the workflow

In n8n UI → Workflows → Import from file → select Main.json.

Update:

OpenAI credentials

Postgres credential references

Email node configuration (SMTP / test inbox)

6.6 Test

Try these in the chat UI:

hi → greeting + Fast answer

tell me about your company → FAQ + fallback agent

give me a quote for my car → quote flow

connect me to a human → escalation email

Check Postgres tables (interaction_log, cust_info_full, quote_register) to see stored rows.

