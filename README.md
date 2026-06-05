# Local AI Automation Stack

A self-hosted, local-first automation stack: **n8n** for workflow orchestration
plus a **local LLM via Ollama**, all running in Docker. The point is to run
automation pipelines (lead enrichment, scraping, outreach drafting) without
sending data to third-party automation platforms or paid LLM APIs.

This is a personal build / learning project, not a turnkey enterprise product —
but the pipelines below are real and runnable on a single machine.

## Why local

- **Data stays on the machine.** Lead details, scraped content, and prompts
  never leave the local network. Useful when you don't want to push customer
  data through a hosted SaaS.
- **No per-token / per-execution billing.** Once the hardware is there, runs are
  free. Tradeoff: you're limited by your own hardware, not infinite scale.
- **No dependency on a hosted API's uptime or rate limits** for the LLM steps.

---

## Architecture

```text
+---------------------------------------------------------------+
| LOCAL HOST (Docker)                                           |
|                                                               |
|  +--------------+   +--------------+   +-------------------+   |
|  |   n8n Node   |-->|  HTML Node   |-->|   AI Agent Node   |   |
|  | (orchestrate)|   | (DOM parse)  |   |   (LangChain)     |   |
|  +--------------+   +--------------+   +-------------------+   |
|        ^                                       |              |
|        | webhooks                              | http         |
|  +--------------+                      +-------------------+   |
|  | ngrok tunnel |                      |   Ollama engine   |   |
|  | (public URL) |                      |  (local models)   |   |
|  +--------------+                      +-------------------+   |
+--------^------------------------------------------------------+
         |
   [ public internet ]
```

- **Orchestration:** n8n in a Docker container, with a persistent volume so
  workflows and credentials survive container restarts.
- **Local LLM:** Ollama running as a container, serving models on port `11434`.
  n8n's Ollama node talks to it over the shared Docker network.
- **Webhooks:** to receive webhooks from the public internet (e.g. a landing-page
  form), an **ngrok** tunnel forwards a public URL to the local n8n port. Note:
  ngrok *exposes* your local n8n endpoint to the internet via a tunnel — it's a
  way to receive external requests without configuring port-forwarding on your
  router, not a security/firewall layer. Use ngrok auth + n8n auth to lock it down.

---

## Setup

Requires Docker.

### 1. Start the stack

```bash
git clone https://github.com/Tusharvijaypatil/local-ai-automation-stack.git
cd local-ai-automation-stack
docker compose up -d
```

`docker-compose.yml`:

```yaml
services:
  n8n:
    image: n8nio/n8n
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=localhost
      - N8N_PORT=5678
      # set this to your ngrok URL when using public webhooks:
      # - WEBHOOK_URL=https://<your-id>.ngrok-free.app
    volumes:
      - n8n_data:/home/node/.n8n
    networks:
      - ai-stack

  ollama:
    image: ollama/ollama
    ports:
      - "11434:11434"
    volumes:
      - ollama_data:/root/.ollama
    networks:
      - ai-stack

volumes:
  n8n_data:
  ollama_data:

networks:
  ai-stack:
    driver: bridge
```

### 2. Pull a model

```bash
docker exec -it $(docker ps -qf name=ollama) ollama pull tinyllama
```

`tinyllama` (1.1B) is a small model — fine for quick local testing on modest
hardware, but light on quality. For real summarization / outreach-drafting
results, pull a larger model if your hardware allows, e.g.:

```bash
docker exec -it $(docker ps -qf name=ollama) ollama pull llama3.1:8b
```

### 3. Point n8n at Ollama

In the n8n Ollama Chat Model node, set the base URL to:

- `http://ollama:11434` — if Ollama runs in this compose stack (recommended), or
- `http://host.docker.internal:11434` — if Ollama runs directly on the host
  rather than as a container.

### 4. (Optional) Expose webhooks with ngrok

```bash
ngrok http 5678
```

Then set `WEBHOOK_URL` in the n8n service to the ngrok URL and recreate the
container.

---

## Importing the workflows

Workflows are stored as JSON in this repo. To import one:

1. Open n8n at `http://localhost:5678`.
2. Create a new empty workflow.
3. Copy the contents of the desired `.json` file.
4. Click on the canvas and paste (Ctrl+V / Cmd+V). The nodes will appear.
5. Set your local credentials (Ollama node → base URL above) and run.

---

## Pipelines

### Pipeline 1 — Lead enrichment & local synthesis

Processes incoming leads locally, no external lookups.

1. **Webhook capture:** ngrok forwards a landing-page form POST to the n8n
   Webhook node.
2. **Context assembly:** lead data is structured into a clean JSON payload.
3. **Local analysis:** the payload goes to a local LLM chain that categorizes
   the lead, infers industry, and assigns a priority score.
4. **Markdown output:** the result is written as a Markdown summary appended to
   a local file for internal review.

### Pipeline 2 — Web scraper & outreach drafting

Drafts cold-outreach copy while working around the small context window of
lightweight local models.

1. **Target input:** accepts target URLs manually or from a queue.
2. **Raw scrape:** an HTTP Request node fetches the page HTML.
3. **HTML extraction:** instead of feeding raw HTML to the model (which blows up
   a small model's context), an HTML node parses the DOM with CSS selectors and
   pulls `body` / `p` / `h1` / `h2` text.
4. **Context reduction:** strips headers, footers, inline CSS, scripts, and
   metadata into a compact `clean_text` field.
5. **Drafting:** `clean_text` is injected into the prompt
   (`{{ $json.clean_text }}`) to draft a short, target-specific outreach email.

---

## Limitations / notes

- Quality is bounded by the local model. `tinyllama` is for testing; expect to
  use a larger model for usable output.
- ngrok free URLs rotate; for anything persistent you'd want a reserved domain
  or a different ingress.
- This is a single-machine setup, not a horizontally scaled deployment.
