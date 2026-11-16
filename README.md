# agent

Docker compose setups for various agents or harnesses. The idea is to use
them with [copier](https://copier.readthedocs.io), e.g.:

```bash
copier copy ~/path/to/this/repo agent
copier copy gh:jochenklar/agent agent
```

## Setup

Create a `.env` file with at least an `AGENT=(pi|opencode|claude|codex|tau)` variable,
which decides which agent is used.

Local paths (e.g. skills or config files) can be mounted into the containers
using a `volumes.yaml` file, which is merged with `compose.yaml` if it exists.
Since only the service for the selected `AGENT` is started, the sections for
several agents can be combined in this one file.

Mounting local paths and configuring local models differs by agent:

### Pi.dev

```yaml
# volumes.yaml
services:
  pi:
    volumes:
    - ./skills:/home/agent/.pi/agent/skills
    - ./models.json:/home/agent/.pi/agent/models.json
```

To use local models with Ollama use a `models.json` like this:

```json
{
  "providers": {
    "ollama": {
      "baseUrl": "http://host.docker.internal:11434/v1",
      "api": "openai-completions",
      "apiKey": "ollama",
      "compat": {
        "supportsDeveloperRole": false,
        "supportsReasoningEffort": false
      },
      "models": [
        {
          "id": "gemma4:e2b"
        }
      ]
    }
  }
}
```

For MLX use:

```json
{
  "providers": {
    "mlx": {
      "baseUrl": "http://host.docker.internal:4040/v1",
      "api": "openai-completions",
      "apiKey": "mlx",
      "compat": {
        "supportsDeveloperRole": false,
        "supportsReasoningEffort": false
      },
      "models": [
        { "id": "mlx-community/gemma-4-E2B-it-qat-4bit" },
        { "id": "mlx-community/gemma-4-E4B-it-qat-4bit" }
      ]
    }
  }
}
```

### OpenCode

```yaml
# volumes.yaml
services:
  opencode:
    volumes:
    - ./skills:/home/agent/.config/opencode/skills
    - ./opencode.json:/home/agent/.config/opencode/opencode.json
```

To use local models with Ollama use a `opencode.json` like this:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "ollama": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Ollama",
      "options": {
        "baseURL": "http://host.docker.internal:11434/v1",
        "apiKey": "ollama"
      },
      "models": {
        "gemma4:e2b": {
          "name": "Gemma 4 E2B"
        }
      }
    }
  },
  "model": "ollama/gemma4:e2b"
}
```

### Claude Code

```yaml
# volumes.yaml
services:
  claude:
    volumes:
    - ./skills:/home/agent/.claude/skills
```

### Codex

```yaml
# volumes.yaml
services:
  codex:
    volumes:
    - ./skills:/home/agent/.codex/skills
```

### Tau

```yaml
# volumes.yaml
services:
  tau:
    volumes:
    - ./skills:/home/agent/.tau/skills
    - ./catalog.toml:/home/agent/.tau/catalog.toml
```

To use local models with Ollama use a `catalog.toml` like this:

```toml
schema_version = 1

[[providers]]
name = "ollama"
display_name = "Ollama"
kind = "openai-compatible"
base_url = "http://host.docker.internal:11434/v1"
api_key_env = "OLLAMA_API_KEY"
credential_name = "ollama"
models = [
    "gemma4:e4b",
    "gemma4:e2b",
]
default_model = "gemma4:e4b"
docs_url = "https://example.test/ollama"
thinking_levels = [
    "low",
    "medium",
    "high"
]
```

**Important:** For this to work, add `OLLAMA_API_KEY=dummy` to `.env`.
