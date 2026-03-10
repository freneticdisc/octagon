# Octagon — Runbook

Operational guide for developers and operators. Covers authentication, environment setup, build, and troubleshooting.

---

## 1. Prerequisites

| Tool | Version | Install |
|---|---|---|
| Python | 3.12+ | [python.org](https://python.org) |
| uv | latest | `curl -LsSf https://astral.sh/uv/install.sh \| sh` |
| Ollama | latest (optional) | [ollama.com](https://ollama.com) — only for local models |
| OCI SDK | latest (optional) | `uv add oci` — only for Oracle OCI provider |
| gcloud CLI | latest (optional) | [cloud.google.com/sdk](https://cloud.google.com/sdk) — only for Vertex AI |

---

## 2. Authentication

**Rule: secrets never go in `config.yaml`.** All credentials are set as environment variables. LiteLLM reads them automatically based on the `provider` field in each participant config.

### Setup

```bash
cp .env.example .env
# Edit .env — fill in only the providers you use
source .env   # or use direnv / your shell's dotenv loader
```

Octagon validates required keys **before** the graph starts. If a key is missing for a configured provider, it exits immediately:

```
EnvironmentError: Missing ANTHROPIC_API_KEY — required for participant 'Security Architect' (anthropic)
```

---

## 3. Provider Authentication Reference

### OpenAI
**Config:** `provider: openai` · **Model prefix:** `gpt-4o`, `o3-mini`, etc.

```bash
OPENAI_API_KEY=sk-...
```
Get your key at [platform.openai.com/api-keys](https://platform.openai.com/api-keys).

---

### Anthropic
**Config:** `provider: anthropic` · **Model prefix:** `claude-opus-4-...`, `claude-sonnet-...`

```bash
ANTHROPIC_API_KEY=sk-ant-...
```
Get your key at [console.anthropic.com/settings/keys](https://console.anthropic.com/settings/keys).

---

### Ollama (Local)
**Config:** `provider: ollama` · **Model prefix:** `ollama/<model-name>`

```bash
OLLAMA_BASE_URL=http://localhost:11434   # default; change if running remotely
```

No API key needed. Pull models before running:
```bash
ollama serve          # start the daemon
ollama pull llama3    # download a model
ollama list           # verify it's available
```

---

### Microsoft Azure OpenAI
**Config:** `provider: azure` · **Model prefix:** `azure/<your-deployment-name>`

```bash
AZURE_API_KEY=...
AZURE_API_BASE=https://your-resource.openai.azure.com
AZURE_API_VERSION=2024-08-01-preview
```

**Important:** `model_name` in `config.yaml` must match the **deployment name** you assigned in Azure AI Studio — not the underlying model name. For example, if you deployed GPT-4o under the name `my-gpt4o-prod`, use `azure/my-gpt4o-prod`.

Get your key and endpoint at [portal.azure.com](https://portal.azure.com) → Azure OpenAI → Keys and Endpoint.

---

### AWS Bedrock
**Config:** `provider: bedrock` · **Model prefix:** `bedrock/<model-id>`

```bash
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
AWS_REGION_NAME=us-east-1
```

**Setup steps:**
1. In AWS Console → Bedrock → Model access, enable the specific models you want to use
2. Your IAM user/role needs `AmazonBedrockFullAccess` or a scoped equivalent
3. Get credentials at [console.aws.amazon.com/iam](https://console.aws.amazon.com/iam) → Users → Security credentials

**Example model IDs:**
- `bedrock/anthropic.claude-3-5-sonnet-20241022-v2:0`
- `bedrock/meta.llama3-70b-instruct-v1:0`
- `bedrock/amazon.nova-pro-v1:0`

---

### Google Vertex AI
**Config:** `provider: vertex_ai` · **Model prefix:** `vertex_ai/<model-id>`

```bash
VERTEXAI_PROJECT=your-gcp-project-id
VERTEXAI_LOCATION=us-central1
```

**Two auth options:**

**Option A — Developer (recommended for local):**
```bash
gcloud auth application-default login
# No GOOGLE_APPLICATION_CREDENTIALS needed
```

**Option B — Service account (recommended for CI/production):**
```bash
# Create a service account with Vertex AI User role, download JSON key
GOOGLE_APPLICATION_CREDENTIALS=/path/to/service-account.json
```

**Setup steps:**
1. Enable the Vertex AI API in your GCP project
2. Ensure the project has billing enabled
3. Authenticate with one of the two options above

**Example model IDs:**
- `vertex_ai/gemini-1.5-pro`
- `vertex_ai/gemini-2.0-flash`
- `vertex_ai/claude-sonnet-4@20250514` (Anthropic on Vertex)

---

### Oracle OCI Generative AI
**Config:** `provider: oci` · **Model prefix:** `oci/<model-id>`

```bash
OCI_USER=ocid1.user.oc1..xxxxx
OCI_FINGERPRINT=xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx
OCI_TENANCY=ocid1.tenancy.oc1..xxxxx
OCI_REGION=us-chicago-1
OCI_KEY_FILE=~/.oci/oci_api_key.pem
OCI_COMPARTMENT_ID=ocid1.compartment.oc1..xxxxx
```

**Setup steps:**
1. Install the OCI SDK: `uv add oci`
2. In OCI Console → Identity → Users → your user → API Keys, generate a signing key pair
3. Download the private key PEM and note the fingerprint
4. Set `OCI_KEY_FILE` to the path of your downloaded PEM file
5. Enable OCI Generative AI in your tenancy (Chicago or Frankfurt regions)
6. Get your compartment OCID from OCI Console → Identity → Compartments

Full guide: [docs.oracle.com — API Signing Key](https://docs.oracle.com/en-us/iaas/Content/API/Concepts/apisigningkey.htm)

**Example model IDs:**
- `oci/meta.llama-3.3-70b-instruct`
- `oci/xai.grok-4`
- `oci/cohere.command-a-03-2025`

**Note on OCI serving modes:** OCI supports `ON_DEMAND` (default) and `DEDICATED` (for dedicated AI clusters). If using a dedicated endpoint, add `oci_serving_mode` and `oci_endpoint_id` to the participant's extra config. See ADR 004 if this is needed.

---

## 4. Provider Summary Table

| Provider | `provider` value | Key env vars | Model prefix | Extra dep |
|---|---|---|---|---|
| OpenAI | `openai` | `OPENAI_API_KEY` | `gpt-4o`, `o3-mini` | — |
| Anthropic | `anthropic` | `ANTHROPIC_API_KEY` | `claude-opus-4-...` | — |
| Ollama | `ollama` | `OLLAMA_BASE_URL` | `ollama/<model>` | Ollama daemon |
| Azure OpenAI | `azure` | `AZURE_API_KEY`, `AZURE_API_BASE`, `AZURE_API_VERSION` | `azure/<deployment>` | — |
| AWS Bedrock | `bedrock` | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION_NAME` | `bedrock/<model-id>` | — |
| Google Vertex AI | `vertex_ai` | `VERTEXAI_PROJECT`, `VERTEXAI_LOCATION` | `vertex_ai/<model>` | gcloud CLI or service account |
| Oracle OCI | `oci` | `OCI_USER`, `OCI_FINGERPRINT`, `OCI_TENANCY`, `OCI_REGION`, `OCI_KEY_FILE`, `OCI_COMPARTMENT_ID` | `oci/<model-id>` | `uv add oci` |

---

## 5. Installation

```bash
git clone <repo-url> && cd octagon
cp .env.example .env   # fill in your provider keys
uv sync
uv run octagon --version
```

---

## 6. Running Octagon

```bash
# Inline prompt
uv run octagon "Design a rate-limiting architecture for a public API"

# From a markdown file
uv run octagon < requirements.md

# No install required (uvx)
uvx octagon "..."

# Validate config and auth without starting a debate
uv run octagon --dry-run
```

---

## 7. Development Commands

```bash
uv run pytest                          # run all tests
uv run pytest tests/test_eviction.py -v  # single file
uv add <package>                       # add runtime dependency
uv add --dev <package>                 # add dev dependency
uv lock                                # regenerate lockfile
```

---

## 8. Output Files

| File | Written when | Gitignored |
|---|---|---|
| `octagon_result.md` | Consensus, max rounds, or budget hit | Yes |
| `state_dump.md` | Human input timeout, exception, or Ctrl+C | Yes |

Both files are overwritten on each run. Archive them manually to preserve results.

---

## 9. Troubleshooting

**`EnvironmentError: Missing X`**  
→ Fill in the variable in `.env` and `source .env`.

**`ConnectionError` for Ollama**  
→ Run `ollama serve`. Check `OLLAMA_BASE_URL` matches the running host and port.

**AWS Bedrock: `AccessDeniedException`**  
→ Enable the specific model in AWS Console → Bedrock → Model access. Check IAM permissions.

**Vertex AI: `Permission denied` or `Project not found`**  
→ Confirm `VERTEXAI_PROJECT` is correct and the Vertex AI API is enabled. Re-run `gcloud auth application-default login`.

**OCI: `NotAuthenticated` or `InvalidParameter`**  
→ Check PEM file path in `OCI_KEY_FILE`. Confirm the fingerprint matches the key uploaded to OCI Console. Ensure the region in `OCI_REGION` has OCI GenAI available (Chicago or Frankfurt).

**Debate terminates immediately on round 1**  
→ `max_session_cost_usd` may be set too low, or a participant's `token_limit` is too tight.

**`state_dump.md` written but no `octagon_result.md`**  
→ Session ended early. Check the last lines of `state_dump.md` for the exit reason.

**Rich output garbled**  
→ Run `export TERM=xterm-256color` or pass `--no-colour` flag.

---

## 10. Adding a New Provider

1. Add the required env vars to `.env.example` (empty values, with comments)
2. Add the provider → env var mapping in `octagon/core/config.py` (`REQUIRED_ENV` dict)
3. Add a commented-out participant example in `config.yaml`
4. Document it in this Runbook under sections 3 and 4
5. Create an ADR in `docs/decisions/` if the provider requires non-standard LiteLLM config
