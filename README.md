# Deploy and Host Opik on Railway

[![Deploy on Railway](https://railway.com/button.svg)](https://railway.com/new/template/opik-llm?referralCode=sLjDXb&utm_medium=integration&utm_source=button&utm_campaign=opik-llm)

[Opik](https://www.comet.com/docs/opik/) is Comet's open-source (Apache-2.0) platform for tracing, evaluating and monitoring LLM applications and agents. It records every prompt, model call, retrieval and tool step as a trace, scores traces with LLM-as-a-judge metrics, and keeps the datasets, experiments and prompt versions you test against. This template runs the official Opik images with ClickHouse, MySQL, Redis and MinIO, and puts the whole site behind basic auth, because self-hosted Opik has no login of its own.

## About Hosting Opik

The stack is six services. Only **Opik** has a public domain; everything else talks over Railway's private network.

- **Opik**: the official `opik-frontend` image (nginx and the React UI) plus the server block upstream's compose mounts from disk. It proxies `/api` to the backend and enforces basic auth on everything except `/health`.
- **Backend**: the official Java API, pinned to 2.2.96. Every boot runs the MySQL and ClickHouse migrations first, and Railway only switches traffic once `/health-check` passes, so upgrading is bumping the image tag.
- **ClickHouse**: stores traces, spans and feedback scores. Opik's schema uses replicated tables and cluster-wide DDL, which need a coordination service even on one node. Upstream runs a separate ZooKeeper for this; here ClickHouse Keeper runs embedded in the ClickHouse server, which saves a service and a 512 MB JVM. ClickHouse logs to the console, its own metric tables are off, and its allocator does not use transparent huge pages (with them, idle ClickHouse on Railway grew past 1.9 GB within half an hour).
- **MySQL 8.4**: projects, datasets, prompts, experiments and settings.
- **Redis 8**: locks, caches and the online-evaluation streams, with an append-only file on a volume.
- **MinIO** (the maintained `pgsty/silo` build): attachments, including large images and payloads the backend strips out of traces. The bucket is created on boot, and uploads and downloads go through the backend, so object storage never needs a public URL.

Every password, plus Opik's encryption key for saved LLM provider keys, is generated per deployment. Upstream's defaults for these are public.

## Common Use Cases

- Debugging RAG pipelines and agents: see which chunks were retrieved, how the prompt was built and which tool call went wrong
- Prompt and model regression testing: run a dataset through a new prompt or model and compare scores before shipping
- Online evaluation: LLM-as-a-judge rules that score live production traces for hallucination, relevance or your own criteria
- Cost, latency and token dashboards per project, on traces that stay in your own infrastructure

## Dependencies for Opik Hosting

- ClickHouse 26.3 with embedded Keeper (included, private network only)
- MySQL 8.4 (included, private network only)
- Redis 8 (included, private network only)
- MinIO S3-compatible storage (included, private network only)

### Deployment Dependencies

- [Opik self-hosting documentation](https://www.comet.com/docs/opik/self-host/overview)
- [Opik on GitHub](https://github.com/comet-ml/opik)
- [Opik Python SDK](https://pypi.org/project/opik/)
- [Template source on GitHub](https://github.com/nomideusz/opik-railway)

### Implementation Details

**Authentication.** Open-source Opik has no users, roles or login. Anyone who can reach it can read every trace and use the LLM provider keys saved in it. This template therefore puts HTTP basic auth in front of the UI and the API. The username is `OPIK_AUTH_USER` (default `opik`), and the password is the generated `OPIK_AUTH_PASSWORD`, both on the Opik service. Your browser asks for them once. Only clear the password if the service has no public domain.

**Sending traces from the SDK.** The Opik SDK sends `OPIK_API_KEY` as the `Authorization` header, and the gate accepts the bare password there:

```bash
pip install opik
export OPIK_URL_OVERRIDE=https://<your-opik-domain>/api
export OPIK_API_KEY=<OPIK_AUTH_PASSWORD>
export OPIK_WORKSPACE=default
```

Then decorate a function with `@opik.track` (or use any of the OpenAI, Anthropic, LangChain, LlamaIndex or LiteLLM integrations), and traces show up in the `Default Project` within seconds. For curl and other HTTP clients, send `Authorization: <password>` or use basic auth.

**LLM-as-a-judge and the playground.** Add a provider key (OpenAI, Anthropic, Gemini, OpenRouter, Ollama or any OpenAI-compatible endpoint) under Configuration → AI Providers. Keys are encrypted with `OPIK_ENCRYPTION_KEY`. Do not change that variable after saving keys.

**What is left out.** Upstream's `python-backend` runs user-written Python code metrics, the Optimization Studio and the demo-data generator. It is not part of this template, to keep the stack smaller, and those features are switched off with the `TOGGLE_*` variables on the Backend. Tracing, datasets, experiments, prompts, dashboards, alerts and LLM-as-a-judge rules all work without it, and evaluations you run from the SDK (`opik.evaluate`) run on your own machine. The GPU-sized guardrails service and the OpenTelemetry collector are also left out.

**Resources.** Measured on Railway after a test deploy settled: about 1.7 GB for the whole stack at idle (ClickHouse 0.5 to 0.7 GB, backend 650 MB, MySQL 285 MB, nginx 105 MB, MinIO 90 MB, Redis 18 MB). The Java backend's heap is capped at 768 MB with `-Xmx` in `JAVA_OPTS` (raise it for heavy ingestion). It needs the Hobby plan or higher; it will not fit the Trial plan's 512 MB per service.

**Back up the volumes.** ClickHouse (`/var/lib/clickhouse`, traces and the Keeper state) and MySQL (`/var/lib/mysql`, projects, datasets and prompts) hold the data that matters. MinIO holds attachments.

## Why Deploy Opik on Railway?

Railway is a singular platform to deploy your infrastructure stack. Railway will host your infrastructure so you don't have to deal with configuration, while allowing you to vertically and horizontally scale it.

By deploying Opik on Railway, you are one step closer to supporting a complete full-stack application with minimal burden. Host your servers, databases, AI agents, and more on Railway.
