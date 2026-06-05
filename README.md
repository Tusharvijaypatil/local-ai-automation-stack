# Local AI Automation Stack & Enterprise RAG Gateway

A production-ready, self-hosted AI automation stack engineered using containerized **n8n** and a decoupled **Python FastAPI microservice backend**. This repository demonstrates how to build end-to-end autonomous business logic, context-optimized web harvesting, and high-security, non-hallucinatory Retrieval-Augmented Generation (RAG) with programmatic human-in-the-loop fallback structures—all running with zero third-party API execution costs or data leaks.

---

## 🏗️ System Architecture & Grid Layout

```text
       [Inbound Trigger Webhook]                 [Target Web URL]
                   │                                     │
                   ▼                                     ▼
 ┌───────────────────────────────────┐ ┌───────────────────────────────────┐
 │ Pipeline 1: Lead Enrichment Agent │ │ Pipeline 2: Outreach Web Scraper  │
 │ (Local TinyLlama Parsing Engine)  │ │ (DOM Element Extraction & Filter) │
 └───────────────────────────────────┘ └───────────────────────────────────┘
                   │                                     │
                   └─────────────────┬───────────────────┘
                                     │ (Operational Stack Expansion)
                                     ▼
 ┌─────────────────────────────────────────────────────────────────────────┐
 │               Pipeline 3: Enterprise Ingress/Egress Gateway              │
 │                (n8n Orchestrator Canvas Edge Gateway)                   │
 └─────────────────────────────────────────────────────────────────────────┘
                                     │
                        (POST Request via REST API)
                                     ▼
 ┌─────────────────────────────────────────────────────────────────────────┐
 │                  BookLeaf Core Microservice Backend                     │
 │      (Python FastAPI Server running on Local Host Network, Port 8000)   │
 └─────────────────────────────────────────────────────────────────────────┘
         │                                   │
         ▼ (Semantic Vector Match)           ▼ (Exact Schema Match)
 ┌───────────────────────────────────┐ ┌───────────────────────────────────┐
 │    FAISS Local Vector Database    │ │    Supabase PostgreSQL Records    │
 │ (knowledge_base.md Ground Truth)  │ │   (Mock Author Database/Logs)     │
 └───────────────────────────────────┘ └───────────────────────────────────┘
         │                                   │
         └─────────────────┬─────────────────┘
                           ▼
             [Confidence Matrix Evaluation]
          Score = 0.4*LLM_Conf + 0.6*Grounding
                           │
             ┌─────────────┴─────────────┐
             │                           │
  (Score >= 0.80 / escalate=false) (Score < 0.80 / escalate=true)
             │                           │
             ▼                           ▼
 ┌───────────────────────────────┐ ┌───────────────────────────────┐
 │   n8n Automated AI Reply      │ │    n8n Human Escalation       │
 │   (Instant Delivery Gate)     │ │   (Status Labeled: QUEUED)    │
 └───────────────────────────────┘ └───────────────────────────────┘

Automation Pipelines & Engineering Scope

1. Pipeline 1: Lead Enrichment Engine

- Core Logic: Captures incoming payload data through a webhook entry gate and forwards the unstructured contact payload to a locally hosted, lightweight tinyllama model via Ollama.

- Formatting: Uses structured system prompts to parse variables out of the message context and return a clean, highly standardized Markdown output format for database matching.

2. Pipeline 2: Outreach Scraper & Context Optimizer

-Core Logic: Automates web harvesting by targeting specific remote endpoints using native HTTP nodes.

-Token Reduction: To bypass large language model context window bottlenecks and memory errors, raw binary HTML outputs are filtered through an intermediate HTML Node utilizing specific body CSS selectors. This sanitizes the raw content, extracting text and passing only clean data to the generative agent.

3. Pipeline 3: Hybrid Enterprise RAG Gateway

- Core Logic: Exposes a complex, pre-existing local RAG bot (BookLeaf) to an n8n automation network. n8n acts as the ingestion edge gate, translating webhook triggers and making an outbound REST call to a local FastAPI server running on the host network.

- Defensive Governance: The Python engine executes programmatic lookups against a local FAISS vector database (knowledge_base.md) and a remote Supabase PostgreSQL cluster. It runs a weighted multi-signal calibration formula ($0.4 \times \text{LLM\_conf} + 0.6 \times \text{Grounding\_conf}$) to verify truth accuracy.

- Human-in-the-Loop Routing: If the Python engine determines a confidence rating under $0.80$, it flags the JSON response with escalate: true. The n8n If Node captures this boolean parameter dynamically, locks down automated execution paths, logs the interaction payload, and assigns a high-priority "escalation_status": "QUEUED" label for review.

Infrastructure Setup & Orchestration
The foundational architecture layers are entirely Dockerized, enabling you to launch your workspace with a single console command.

Environment Requirements
Ensure your docker configuration profile or a local .env file contains your connection records:

- LLM_API_KEY (Gemini API Authentication Vector)

- SUPABASE_URL & SUPABASE_KEY (Remote PostgreSQL Cluster Vectors)

Deployment Commands
1) Spin up the Automation Container Network:

Bash
docker-compose up -d

2) Launch the Host Microservice Engine:
Navigate into your local bookleaf-query-bot directory and initiate the server:

Bash
python main.py

The server initializes an active listener on your local network context: http://0.0.0.0:8000

Importing Workflows to n8n
Every pipeline configuration is preserved as an independent JSON canvas template located directly in the root directory of this repository:

📄 pipeline-1-lead-enrichment.json

📄 pipeline-2-scraper-outreach.json

📄 project3_rag_gateway.json

Step-by-Step Restoration

1) Access your local running n8n instance canvas (http://localhost:5678).

2) Create a clean, blank workflow dashboard.

3) Open the corresponding .json blueprint file in this repository using any raw text viewer and copy its entire text string.

4) Click anywhere directly onto your empty n8n canvas background and press Ctrl + V (or Cmd + V on Mac).

5) The full visual node schema, configuration settings, and connection parameters will map onto your grid instantly.
