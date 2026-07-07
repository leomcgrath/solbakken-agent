<p align="center">
  <img width="350" height="350" alt="image" src="https://github.com/user-attachments/assets/4d6e4d09-c076-45ba-96fa-536a6b52fa27" />
</p>

<p align="center">
 <h1>Solbakken</h1>
</p>

A cost-saving orchestrator agent for [opencode](https://opencode.ai) that
breaks large coding tasks into subtasks and delegates the bulk of the work to
two local Ollama models, stepping in itself only for planning, judgment
calls, and review.

- **Solbakken** (`primary`, runs on your configured paid/cloud model) — plans
  the work, splits it into subtasks, delegates, and reviews results.
- **Haaland** (`subagent`, `ollama/qwen3-coder:30b-a3b-q8_0`) — handles the majority of
  real coding subtasks: implementing changes, multi-file edits, debugging.
  Only one instance runs at a time.
- **Nusa** (`subagent`, `ollama/qwen3:8b`) — handles small,
  mechanical, parallelizable subtasks: focused reads/searches, running a
  command, single small edits. Light enough to run several instances
  concurrently.

This exists to save cost, not time: local models are slower than a strong
cloud model, but free. Solbakken delegates by default whenever the change
is cheap enough to hand off — see `Solbakken.md` for its full triage
logic.

## Prerequisites

- [Ollama](https://ollama.com) installed and running locally.
- Pull the two models Haaland and Nusa run on:

  ```sh
  ollama pull qwen3-coder:30b-a3b-q8_0
  ollama pull qwen3:8b
  ```

- Nusa runs several instances concurrently, so start the Ollama server with
  enough parallel request slots to actually process them at once instead of
  queueing:

  ```sh
  OLLAMA_NUM_PARALLEL=4 ollama serve
  ```

  (If Ollama already runs as a background service, set `OLLAMA_NUM_PARALLEL=4`
  in its environment — e.g. via `launchctl setenv OLLAMA_NUM_PARALLEL 4` on
  macOS or a systemd unit override on Linux — and restart it.)

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

  Adjust `models` to whatever you've actually pulled/named locally — the
  names must match the `model:` field in `Haaland.md`/`Nusa.md`
  (`ollama/qwen3-coder:30b-a3b-q8_0`, `ollama/qwen3:8b`).

## Install

1. **Get the files.** Clone this repo (or just download the three `.md`
   files from it):

   ```sh
   git clone https://github.com/leomcgrath/solbakken-agent.git
   cd solbakken-agent
   ```

2. **Copy the three agent files** into an opencode agent directory. Pick
   one:

   ```sh
   # Option A — global: available in every project
   mkdir -p ~/.config/opencode/agent
   cp agent/Haaland.md agent/Nusa.md agent/Solbakken.md ~/.config/opencode/agent/

   # Option B — per-project: available only in one project
   mkdir -p /path/to/project/.opencode/agent
   cp agent/Haaland.md agent/Nusa.md agent/Solbakken.md /path/to/project/.opencode/agent/
   ```

3. **Set Solbakken's model.** Open the copy of `Solbakken.md` you just
   placed and change the `model:` field in its frontmatter (it defaults to
   `github-copilot/claude-opus-4.8`) to whatever paid/cloud model you want
   driving the orchestration. This does not need to match your Ollama
   setup — only `Haaland.md`/`Nusa.md` need to match Ollama.

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
