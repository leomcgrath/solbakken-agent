<p align="center">
  <img width="350" height="350" alt="image" src="https://github.com/user-attachments/assets/4d6e4d09-c076-45ba-96fa-536a6b52fa27" />
</p>


# Solbakken

A cost-saving orchestrator agent for [opencode](https://opencode.ai) that
breaks large coding tasks into subtasks and delegates the bulk of the work to
two local Ollama models, stepping in itself only for planning, judgment
calls, and review.

- **Solbakken** (`primary`, runs on your configured paid/cloud model) — plans
  the work, splits it into subtasks, delegates, and reviews results.
- **Haaland** (`subagent`, `ollama/ornith:35b`) — handles the majority of
  real coding subtasks: implementing changes, multi-file edits, debugging.
  Only one instance runs at a time.
- **Nusa** (`subagent`, `ollama/qwen2.5-coder:7b`) — handles small,
  mechanical, parallelizable subtasks: focused reads/searches, running a
  command, single small edits. Light enough to run several instances
  concurrently.

This exists to save cost, not time: local models are slower than a strong
cloud model, but free. Solbakken only orchestrates when a task is high-volume
enough that the tokens saved by not writing it yourself outweigh the
delegation overhead — see `Solbakken.md` for its full triage logic.

## Prerequisites

- [Ollama](https://ollama.com) installed and running locally.
- Pull the two models Haaland and Nusa run on:

  ```sh
  ollama pull ornith:35b
  ollama pull qwen2.5-coder:7b
  ```

- An `ollama` provider configured in your opencode config
  (`opencode.jsonc`/`opencode.json`) so opencode can reach the local Ollama
  server:

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
          "qwen2.5-coder:7b": {
            "name": "Qwen2.5 Coder 7B (local)",
            "limit": {
              "context": 32768,
              "output": 8192
            }
          },
          "ornith:35b": {
            "name": "Ornith 35B (local)",
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

  Adjust `models` to whatever you've actually pulled/named locally — the
  names must match the `model:` field in `Haaland.md`/`Nusa.md`
  (`ollama/ornith:35b`, `ollama/qwen2.5-coder:7b`).

## Install

Copy the three agent files into your opencode agent directory:

```sh
# Global, available in all projects:
cp Haaland.md Nusa.md Solbakken.md ~/.config/opencode/agent/

# Or per-project:
cp Haaland.md Nusa.md Solbakken.md /path/to/project/.opencode/agent/
```

`Solbakken.md` sets `model: github-copilot/claude-opus-4.8` — change this to
whatever primary model you want driving the orchestration; it doesn't need
to match your Ollama setup.

## Usage

Invoke `Solbakken` as your primary agent for a large, multi-step task.
It will plan the work with `todowrite`, delegate subtasks to `Haaland`
(sequentially) and `Nusa` (batched in parallel), review results against
`git diff`/test output, and report back what was delegated vs. done itself.

For small or single-file tasks, Solbakken is designed to skip delegation
entirely and just do the work directly — orchestration overhead only pays
off on high-volume, decomposable work.

## License

MIT — see [LICENSE](LICENSE).
