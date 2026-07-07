<p align="center">
  <img width="350" height="350" alt="Solbakken" src="https://github.com/user-attachments/assets/4d6e4d09-c076-45ba-96fa-536a6b52fa27" />
</p>

<h1 align="center">Solbakken</h1>

<p align="center">
  <em>He doesn't write the code. He picks the team.</em>
</p>

<p align="center">
  <img alt="opencode" src="https://img.shields.io/badge/opencode-agent-black" />
  <img alt="ollama" src="https://img.shields.io/badge/ollama-local-white" />
  <img alt="workers" src="https://img.shields.io/badge/cloud%20tokens-planning%20only-blue" />
  <img alt="license" src="https://img.shields.io/badge/license-MIT-green" />
</p>

<p align="center">
  <b>Cloud model judgment · local model labor · you pay for the manager, not the squad</b>
</p>

<p align="center">
  Solbakken is a cost-saving orchestrator agent for <a href="https://opencode.ai">opencode</a>.
  It breaks large coding tasks into subtasks and delegates the bulk of the work to two
  local Ollama models, stepping in itself only for planning, judgment calls, and review.
  Every mechanical edit, search, and multi-file change runs on your own hardware at
  $0 per token. The expensive model touches the keyboard only when it has to.
</p>

<p align="center">
  Named after the Norwegian national football team, and written the night Norway beat
  Brazil 2-1. (Ro!) The manager stays on the touchline. Haaland scores.
</p>

## Setup

You need [Ollama](https://ollama.com) installed, plus three steps: pull the
models, let Ollama handle parallel requests, and add the agents to opencode.

### 1. Pull the models

```sh
ollama pull qwen3-coder:30b-a3b-q8_0
ollama pull qwen3:8b
```

> **Different hardware?** You can substitute smaller/larger models — just
> use the same names in the opencode config (step 3) and in the `model:`
> field of `Haaland.md`/`Nusa.md`.

### 2. Allow parallel requests

Nusa runs several instances at once, but by default Ollama queues
concurrent requests instead of processing them in parallel. Set
`OLLAMA_NUM_PARALLEL` to fix that:

```sh
# If you run `ollama serve` yourself:
OLLAMA_NUM_PARALLEL=4 ollama serve

# If Ollama runs as the macOS app:
launchctl setenv OLLAMA_NUM_PARALLEL 4
# ...then fully quit and reopen Ollama.app so it picks up the setting.

# On Linux with systemd, add an env override instead:
#   systemctl edit ollama  →  Environment="OLLAMA_NUM_PARALLEL=4"
#   sudo systemctl restart ollama
```

The env var only affects Ollama instances started *after* it's set, so a
restart of the app/service is always required.

### 3. Configure opencode

Add an `ollama` provider to your opencode config
(`~/.config/opencode/opencode.jsonc`) so opencode can reach the local
Ollama server:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "ollama": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Ollama (local)",
      "options": {
        "baseURL": "http://localhost:11434/v1"
      },
      "models": {
        "qwen3:8b": {
          "name": "Qwen3 8B (local)",
          "limit": {
            "context": 32768,
            "output": 8192
          }
        },
        "qwen3-coder:30b-a3b-q8_0": {
          "name": "Qwen3 Coder 30B A3B Q8_0 (local)",
          "limit": {
            "context": 262144,
            "output": 8192
          }
        }
      }
    }
  }
}
```

Then copy the three agent files into an opencode agent directory:

```sh
git clone https://github.com/leomcgrath/solbakken-agent.git
cd solbakken-agent

# Option A — global: available in every project
mkdir -p ~/.config/opencode/agent
cp agent/*.md ~/.config/opencode/agent/

# Option B — per-project: available only in one project
mkdir -p /path/to/project/.opencode/agent
cp agent/*.md /path/to/project/.opencode/agent/
```

That's it — restart opencode (or start a new session) and `Solbakken`,
`Haaland`, and `Nusa` will show up as available agents.

## Usage

Invoke `Solbakken` as your primary agent for a large, multi-step task.
It will plan the work with `todowrite`, delegate subtasks to `Haaland`
(sequentially) and `Nusa` (batched in parallel), review results against
`git diff`/test output, and report back what was delegated vs. done itself.

Solbakken delegates by default whenever it's cheaper than writing the
change itself — including small, single-file edits — and only keeps work
for itself when a change is trivial (a one-liner) or needs judgment no
local model can be trusted with.

## License

MIT — see [LICENSE](LICENSE).
