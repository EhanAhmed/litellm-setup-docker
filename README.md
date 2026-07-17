# LiteLLM Proxy with Docker Compose

A self-hosted [LiteLLM](https://docs.litellm.ai/) proxy backed by PostgreSQL. It provides an OpenAI-compatible API on port `4000`, routes the configured model aliases to OpenAI, and persists proxy data in a Docker volume.

## What is included

- LiteLLM proxy using `ghcr.io/berriai/litellm:main-latest`
- PostgreSQL 16 for LiteLLM data and usage history
- OpenAI provider credentials supplied through environment variables
- A local `model_prices.json` catalog for model metadata and pricing
- The model aliases configured in `config.yaml`

## Prerequisites

- Docker Engine with the Docker Compose plugin
- An OpenAI API key with access to the models configured in `config.yaml`
- `curl` to refresh the pricing catalog

## Quick Start

1. Clone this repository and enter it.

	```bash
	git clone <your-repository-url>
	cd litellm_setup
	```

2. Create your local environment file.

	```bash
	cp .env.example .env
	```

3. Set secure values in `.env`.

	```dotenv
	OPENAI_API_KEY=sk-your-openai-api-key
	LITELLM_MASTER_KEY=sk-your-strong-proxy-master-key
	LITELLM_DB_PASSWORD=use-a-long-random-postgres-password
	```

	`LITELLM_MASTER_KEY` protects requests to the proxy. Use a distinct, high-entropy value rather than your OpenAI key.

4. Start the services.

	```bash
	docker compose up -d
	```

5. Confirm both containers are running.

	```bash
	docker compose ps
	```

6. Send a test request through the proxy.

	```bash
	curl http://localhost:4000/v1/chat/completions \
	  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
	  -H "Content-Type: application/json" \
	  -d '{
		 "model": "gpt-4o-mini",
		 "messages": [{"role": "user", "content": "Reply with: LiteLLM is running."}]
	  }'
	```

	Export the master key first when running this command from a new shell:

	```bash
	set -a
	source .env
	set +a
	```

## Configured Models

Requests must use one of these LiteLLM model aliases:

| Alias | OpenAI model |
| --- | --- |
| `gpt-5.6-sol` | `gpt-5.6-sol` |
| `gpt-5.6-terra` | `gpt-5.6-terra` |
| `gpt-5.6-luna` | `gpt-5.6-luna` |
| `gpt-5.5` | `gpt-5.5` |
| `gpt-5.3-codex` | `gpt-5.3-codex` |
| `o3-mini` | `o3-mini` |
| `o4-mini` | `o4-mini` |
| `gpt-4o` | `gpt-4o` |
| `gpt-4o-mini` | `gpt-4o-mini` |

Add, remove, or rename aliases in `config.yaml`, then apply the change:

```bash
docker compose up -d --force-recreate litellm-proxy
```

`drop_params: true` is enabled, so LiteLLM drops unsupported request parameters instead of failing the request. This is useful when clients send parameters that are not accepted by every configured model.

## Pricing Catalog

`model_prices.json` is mounted into the LiteLLM container and configured through `model_prices_and_context_window_path`. Refresh it from LiteLLM's upstream catalog when needed:

```bash
curl -L -o model_prices.json \
  https://cdn.jsdelivr.net/gh/BerriAI/litellm@main/model_prices_and_context_window.json
docker compose up -d --force-recreate litellm-proxy
```

Review the diff before committing a catalog refresh because the upstream file is large.

## Operations

View logs:

```bash
docker compose logs -f litellm-proxy
```

Stop the stack while retaining PostgreSQL data:

```bash
docker compose down
```

Remove the stack and all persisted database data:

```bash
docker compose down -v
```
