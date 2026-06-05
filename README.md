# Local AI Automation Stack

A self-hosted, local-first n8n + Ollama automation experiment. This repository contains workflow templates designed to run LLM-powered pipelines locally on your own hardware, removing dependencies on third-party cloud orchestrators and external proprietary LLM APIs.

---

## 1. Core Architecture

The stack runs locally on containerized infrastructure using Docker Compose to bridge n8n's workflow orchestration with Ollama's local model execution.

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
|  |   (Public Ingress) |                                  |   (Local Compute)   |  |
|  +--------------------+                                  +---------------------+  |
+------------^----------------------------------------------------------------------+
             |
     [Public Internet]
```

- **Orchestration:** n8n runs in a Docker container. Workflows and local database states are persisted using Docker volumes mapped to n8n's internal filesystem.
- **Tunneling:** An ngrok tunnel is used to expose the local n8n webhook port to the public internet. This allows the local instance to receive webhook payloads from public landing pages or forms without requiring router port-forwarding or firewall modifications.
- **Local LLM Compute:** Ollama runs alongside n8n on the same Docker bridge network (`local-ai-network`). The n8n instances connect directly to Ollama at `http://ollama:11434`.

---

## 2. Workflows & Pipelines

### Pipeline 1: Lead Enrichment & Local Synthesis
File: [`project1_lead_scorer.json`](./project1_lead_scorer.json)

Processes incoming business leads and generates a markdown summary locally:
1. **Webhook Capture:** ngrok receives an HTTP POST payload from a landing page form and routes it to n8n's Webhook node.
2. **Local Token Analysis:** The payload is sent to Ollama, where a local LLM categorizes the lead and evaluates priority based on the message content and budget.
3. **Context Assembly:** n8n structures the model's analysis into a clean JSON payload.
4. **Markdown Synthesis:** The output is appended to a local markdown file on the host filesystem for review.

### Pipeline 2: Web Scraper & Outreach Personalization
File: [`project2_web_scraper.json`](./project2_web_scraper.json)

Scrapes a target website and drafts a personalized email:
1. **Target Ingress:** Accepts target URLs manually or from an input queue.
2. **Raw Scrape:** Fetches the raw HTML content of the target website using n8n's HTTP Request node.
3. **HTML Extraction:** An HTML parser node isolates the `body` tag and removes script/style tags. This reduces the token volume to avoid context overflows and performance bottlenecks in lightweight models.
4. **Context Reduction:** The extracted clean text is assigned to a structured workflow variable.
5. **AI Synthesis:** The clean text is injected into the prompt of a local LLM agent to draft a personalized three-sentence outreach email.

---

## 3. Setup & Installation

### Prerequisites
- Docker and Docker Compose installed.
- An ngrok account (if you want to receive public webhooks).

### 1. Clone the repository
```bash
git clone https://github.com/Tusharvijaypatil/local-ai-automation-stack.git
cd local-ai-automation-stack
```

### 2. Start the Stack
Create a `docker-compose.yml` file or use the one provided:
```bash
docker compose up -d
```
This spins up:
- **Ollama** at `http://localhost:11434`
- **n8n** at `http://localhost:5678`

### 3. Configure the Local LLM
By default, the workflow templates reference `tinyllama:latest` (a lightweight 1.1B model). This model is a low-resource demo default chosen for fast execution on standard laptops, but its reasoning and instruction-following capabilities are highly limited.

To pull the default model:
```bash
docker exec -it ollama-local ollama run tinyllama
```

For higher-quality output, swap in a larger model (e.g., Llama 3 8B or Gemma 2 9B) if your hardware has sufficient VRAM/RAM:
```bash
docker exec -it ollama-local ollama run llama3
```
After pulling a larger model, update the **Model** field in n8n's Ollama Chat Model node to match the model name (e.g., `llama3:latest`).

### 4. Importing Workflows to n8n
1. Open n8n in your browser at `http://localhost:5678`.
2. Create a new workflow.
3. Open [`project1_lead_scorer.json`](./project1_lead_scorer.json) or [`project2_web_scraper.json`](./project2_web_scraper.json) in a text editor, copy the entire JSON content, and paste it (Ctrl+V) directly onto the n8n workflow canvas.
4. When configuring the Ollama model node, ensure the Base URL is set to `http://ollama:11434` (the container DNS name inside the Docker bridge network).
