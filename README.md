# Local AI Automation Stack & Private Orchestration Pipelines

## Executive Summary
Modern enterprise operations are increasingly throttled by the recurring subscription costs and data privacy concerns associated with third-party automation platforms (e.g., Make.com) and proprietary LLM APIs (e.g., OpenAI). This repository provides a blueprint for a fully self-hosted, local-first alternative. 

By combining the workflow orchestration of **n8n** with the local model execution of **Ollama**, this architecture provides significant commercial value:
* **Zero Data Exfiltration Risk:** Sensitive operational data, proprietary source code, and customer information never leave the local network boundary, aligning with stringent GDPR, HIPAA, and CCPA compliance requirements.
* **Cost Elimination:** Replaces variable per-token LLM pricing and workflow execution tier limits with fixed-cost local hardware utilization, enabling infinite scaling of automated routines.
* **Deterministic Execution:** Mitigates the risk of upstream API outages, rate-limiting, and deprecation of proprietary foundation models.

---

## Core Architecture Details
The stack runs locally on containerized infrastructure designed for low latency, secure external ingress, and bridge networking between automation logic and LLM runtimes.

```text
+-----------------------------------------------------------------------------------+
| LOCAL HOST                                                                        |
|                                                                                   |
|  +--------------------+      +--------------------+      +---------------------+  |
|  |     n8n Node       | ===> |     HTML Node      | ===> |    AI Agent Node    |  |
|  | (Orchestration &   |      | (DOM Parser / CSS) |      |     (LangChain)     |  |
|  |  Data Ingress)     |      +--------------------+      +---------------------+  |
|  +--------------------+                                             ||            |
|            ^                                                        || (Bridge)   |
|            || (Webhooks)                                            ||            |
|  +--------------------+                                  +---------------------+  |
|  |    ngrok Tunnel    |                                  |    Ollama Engine    |  |
|  |   (Edge Routing)   |                                  |   (Local Compute)   |  |
|  +--------------------+                                  +---------------------+  |
+------------^----------------------------------------------------------------------+
             |
     [Public Internet]

- Orchestration: Built on a Docker containerized n8n deployment. Local networking is handled via a dedicated bridge driver, with persistent volumes mapped to the host filesystem to ensure SQLite/PostgreSQL workflow data and custom scripts persist across container restarts.

- Tunneling: To receive real-time public webhooks without exposing the host firewall directly to the public internet, an edge-routing layer utilizing secure ngrok tunnels is integrated. This maps external HTTP POST/GET requests directly to the internal n8n webhook port.

- Local LLM Compute: Powered by a local Ollama daemon. The containerized n8n instances communicate with Ollama over the host-docker bridge interface at http://host.docker.internal:11434. Ollama dynamically manages model weights in VRAM/RAM, routing queries to resource-optimized local models such as tinyllama.

Project Deep Dives
Pipeline 1: Lead Enrichment & Private Synthesis
This pipeline processes incoming business leads locally without sending contact details to external databases.

1) Webhook Capture: ngrok captures incoming payloads from landing page forms and sends them to the n8n Webhook node.

3) Context Assembly: The lead data is structured into an optimized JSON payload.

2) Local Token Analysis: The payload is passed to a localized LLM chain to categorize the lead, infer industry alignment, and assign a priority score.

4) Markdown Synthesis: The output is formatted into a standardized Markdown summary document and appended to a local markdown file, ready for internal sales review.

Pipeline 2: Autonomous Web Scraper & Outreach Agent
This pipeline automates cold outreach personalization while addressing context window limitations of lightweight, local LLMs.

1) Target Ingress: Accepts target URLs manually or from a prior queue.

2) Raw Scrape: An HTTP Request node fetches the target website's raw content.

3) HTML Extraction Node: Instead of piping raw HTML (which causes VRAM/memory bottlenecks and context overflows in small models like tinyllama), an intermediate HTML Node parses the DOM using CSS selectors targeting the body tag (or specific p, h1, h2 blocks).

4) Context Reduction: It strips out headers, footers, inline CSS, script tags, and metadata, writing the output to a compact clean_text property.

5) AI Synthesis: The clean_text is injected into the AI Personalization Agent's prompt ({{ $json.clean_text }}) to draft hyper-personalized, three-sentence B2B outreach emails based strictly on the target's unique value proposition.

Setup & Installation
Prerequisite Setup
1) Clone the repository to your local system:

Bash
   git clone [https://github.com/your-username/local-ai-automation-stack.git](https://github.com/your-username/local-ai-automation-stack.git)
   cd local-ai-automation-stack

2) Start the containerized services using Docker:

Bash
   docker run -d -v ollama:/root/.ollama -p 11434:11434 --name ollama-local ollama/ollama

3) Verify Ollama is running and pull the tinyllama model:

Bash
   docker exec -it ollama-local ollama run tinyllama


Importing Workflows
All pipelines are stored in this repository as serialized JSON templates. To import a pipeline into your stack:

1) Open your browser and navigate to your local n8n instance (typically http://localhost:5678).

2) Create a new, empty workflow canvas.

3) Copy the contents of the desired workflow .json file from your repository.

4) Focus your cursor on the n8n canvas and press Ctrl+V (or Cmd+V on macOS). The nodes and connection graphs will instantiate automatically.

5) Configure your local credentials (e.g., pointing the Ollama Chat Model node to http://host.docker.internal:11434 with credential name Ollama Local) and run the workflow.
