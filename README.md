# agent

Docker compose setups for various coding agents or harnesses
([Pi](https://pi.dev), [OpenCode](https://opencode.ai),
[Claude Code](https://claude.com/claude-code),
[Codex](https://github.com/openai/codex) and Tau). The idea is to use them with
[copier](https://copier.readthedocs.io), e.g.:

```bash
copier copy ~/path/to/this/repo agent
copier copy gh:jochenklar/agent agent
```

Copier asks which agent should be used (`pi`, `opencode`, `claude`, `codex` or
`tau`) and renders a ready-to-use project for it.

## Usage

```bash
make run     # start the agent (default target)
make bash    # open a bash shell in the agent's container
make build   # (re)build the images
make clean   # remove the agent's container and its home volume
make mlx     # start an MLX server on the host at port 4040 (needs mlx_lm)
```

The agent runs in a container based on Alpine with `bash`, `git`, `gh`, `curl`,
`jq`, `yq`, `ripgrep`, `fd`, `make`, Node.js/npm and `uv` (with a Python
3.14 virtualenv in `/opt/env`). The GNU versions of `coreutils`, `findutils`,
`sed`, `grep`, `awk`, `diff` and `ps` replace the BusyBox tools, since agents
usually expect their flags. The workspace is mounted at
`/home/agent/workspace`. The home directory of the container is a named docker
volume per agent, so logins and sessions persist between runs until `make
clean` is used.

The containers run with all capabilities dropped and `no-new-privileges`, and
the agents are started in their auto-approve modes (`claude
--dangerously-skip-permissions`, `codex --yolo`, `opencode --auto`), since the
container is the sandbox. The host is reachable as `host.docker.internal`.

On Linux, the container user gets the UID/GID of the current user, so files in
the workspace have the right owner. On macOS (or as root), `1000:1000` is used.

## Setup

The `.env` file needs at least an `AGENT=(pi|opencode|claude|codex|tau)`
variable, which decides which agent is used. When using `copier copy`, it is
created with the selected agent already set. Other variables in `.env` are
passed to the container (e.g. API keys). The following variables are used by
the `Makefile`:

| Variable         | Default       | Description                                    |
| ---------------- | ------------- | ---------------------------------------------- |
| `AGENT`          | `pi`          | The agent to use.                              |
| `WORKSPACE_PATH` | `./workspace` | Local path mounted as `/home/agent/workspace`. |

Local paths (e.g. skills or config files) can be mounted into the containers
using a `volumes.yaml` file, which is merged with `compose.yaml` if it exists.
Since only the service for the selected `AGENT` is started, the sections for
several agents can be combined in this one file.

When running `copier copy`, `templates/<agent>/volumes.yaml` for the selected
agent is rendered as `volumes.yaml` into the new project. The agent specific
config files (`templates/pi/models.json`, `templates/opencode/opencode.json`,
`templates/tau/catalog.toml`) are rendered into the root of the new project,
but only for the agent which uses them. The `templates` directory itself is not
copied.

Rules which should always be used go into `AGENTS.md`. It is copied for
every agent and mounted as the agent's global instruction file.

Skills are not mounted by default. To use local skills, add a mount for the
`skills` directory (its content is ignored by git) to `volumes.yaml`, e.g. for
Pi `/home/agent/.pi/agent/skills`.

Mounting local paths and configuring local models differs by agent:

### Pi

```yaml
# volumes.yaml
services:
  pi:
    volumes:
    - ./AGENTS.md:/home/agent/.pi/agent/AGENTS.md
    - ./models.json:/home/agent/.pi/agent/models.json
```

The `models.json` from the template configures local models with Ollama and
MLX (use `make mlx` to start the MLX server):

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
        { "id": "gemma4:e4b" },
        { "id": "gemma4:e2b" }
      ]
    },
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
    - ./AGENTS.md:/home/agent/.config/opencode/AGENTS.md
    - ./opencode.json:/home/agent/.config/opencode/opencode.json
```

The `opencode.json` from the template configures local models with Ollama:

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
        "gemma4:e4b": {
          "name": "Gemma 4 E4B"
        },
        "gemma4:e2b": {
          "name": "Gemma 4 E2B"
        }
      }
    }
  },
  "model": "ollama/gemma4:e4b"
}
```

### Claude Code

```yaml
# volumes.yaml
services:
  claude:
    volumes:
    - ./AGENTS.md:/home/agent/.claude/CLAUDE.md
```

### Codex

```yaml
# volumes.yaml
services:
  codex:
    volumes:
    - ./AGENTS.md:/home/agent/.codex/AGENTS.md
```

### Tau

```yaml
# volumes.yaml
services:
  tau:
    volumes:
    - ./AGENTS.md:/home/agent/.tau/AGENTS.md
    - ./catalog.toml:/home/agent/.tau/catalog.toml
```

The `catalog.toml` from the template configures local models with Ollama:

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
