# akmon-templates

YAML detection rules for [akmon](https://github.com/ClawGuard-Labs/akmon). This repository holds **only** templates; the engine lives in **akmon** (`internal/templates/`).

## Usage

| Path | Purpose |
|------|---------|
| `behavioral-templates/` | Passive detection rules (session, file, process, network subfolders). |
| `nuclei-templates/` | Nuclei v3 HTTP templates for active scans (e.g. `ai-services/`). |

**Behavioral** rules score/tag events from telemetry the agent already collects. **Nuclei** rules send HTTP requests against discovered services.

```
behavioral-templates/
├── file/
│   ├── model-load.yaml          # AI model file extensions (.pt, .gguf, .safetensors …)
│   ├── large-mmap.yaml          # Memory-mapped files >100 MB (model load via mmap)
│   ├── sensitive-path.yaml      # /etc, /proc, /sys access
│   ├── ssh-key-access.yaml      # SSH private key / authorized_keys access
│   ├── self-modify.yaml         # Process writes to its own binary
│   ├── config-access.yaml       # .json, .yaml, .env, .toml file access
│   └── file-deleted.yaml        # File deletion (unlink)
├── network/
│   ├── outbound-http.yaml       # Connections to port 80/443
│   ├── http-post.yaml           # HTTP POST requests
│   └── unusual-port.yaml        # Connections to non-standard ports
├── process/
│   ├── ai-process.yaml          # Known AI runtimes (python, ollama, vllm …)
│   └── shell-spawned-by-ai.yaml # Shell (bash/sh/zsh) spawned in AI session
└── session/
    ├── download-exec-chain.yaml  # exec within 30s of net_connect
    ├── curl-bash-chain.yaml      # curl/wget → shell pattern (RCE risk)
    ├── long-running-llm.yaml     # AI process active >5 minutes
    └── read-write-inference-loop.yaml  # Read one file, write another (inference pipeline)
```

```
nuclei-templates/ai-services/
├── qdrant-unauth.yaml        # Qdrant vector DB unauthenticated /collections
├── chromadb-unauth.yaml      # ChromaDB unauthenticated /api/v1/collections
├── ollama-api-exposed.yaml   # Ollama /api/tags exposed
├── weaviate-unauth.yaml      # Weaviate /v1/schema exposed
├── vllm-api-exposed.yaml     # vLLM /v1/models exposed
├── gradio-exposed.yaml       # Gradio ML app publicly accessible
└── localai-exposed.yaml      # LocalAI /v1/models exposed
```

## Run with a local clone

Point **`--behavioral-templates`** at this repo’s **`behavioral-templates/`** directory (not the repo root, or the loader would try to parse Nuclei YAML as behavioral rules). Example when this checkout is **`./akmon-templates`** next to the binary’s cwd:

```bash
./bin/akmon \
  --behavioral-templates ./akmon-templates/behavioral-templates \
  --nuclei-templates ./akmon-templates/nuclei-templates
```

Installed systems typically use **`/etc/akmon/behavioral-templates`** and **`/etc/akmon/nuclei-templates`**. **akmon** defaults are **`./akmon-templates/behavioral-templates`** and **`./nuclei-templates`**—adjust flags or layout as needed.

## Authoring rules

See **[AUTHORING.md](./AUTHORING.md)** for the behavioral YAML schema, matcher types, and examples.

## Contributing

Open pull requests for YAML templates and docs in this repo only; code changes belong in **akmon**.
