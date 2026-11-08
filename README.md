# Claude Code with Strata workers

The main Claude Code session runs on Claude. Delegated work (subagent tasks) runs on a local model served by Strata on a GPU machine. Claude plans, hands self-contained tasks to the local model, and checks the result.

There are two worker paths:

| Path | What runs | When |
| --- | --- | --- |
| `/sol-pi` (`sol-pi-run`) | The Pi coding agent with the SoL-Pi extension, on Strata | Default |
| `strata-claude` | Headless `claude -p` pointed at Strata | Fallback when SoL-Pi is broken |

Why a wrapper and not a normal subagent: the `model:` field of a Claude Code subagent cannot change the API base URL, so a subagent cannot be pointed at a local server. Each worker is a separate process instead.

## Requirements

- A GPU machine for Strata. Tested on Linux with an RTX 4070 (12 GB) and 64 GB RAM, serving Qwen3.8-Flash-Next IQ3_S with 256k context.
- A client machine with Claude Code, Node.js 22.19+, npm, Python 3.10+ and git.
- Both machines on one network. We use Tailscale. Cloud sessions (claude.ai/code) cannot reach a tailnet, so this only works for local Claude Code.

In the examples, replace `STRATA_HOST` with the GPU machine's address and `MODEL` with the model name the server reports (ours is `qwen3.8-flash-next-iq3_s`).

## 1. Serve the model with Strata (GPU machine)

Install Strata with its installer, then start it once with the guided setup:

```bash
strata launch
```

Or with flags. This is our autostart line:

```bash
strata launch --family qwen --model iq3_s --context 262144 \
  --access tailscale --port 8080 --api-key "$KEY" \
  --set vram-reserve-mib=1228 --yes --no-tui
```

- `--access tailscale` listens on the tailnet. `--api-key` protects `/v1/*`. Generate a long random key.
- `vram-reserve-mib` leaves room for the desktop. Tune it for your card.
- The server serves one request at a time (`--parallel 1`). Workers queue.

Start it at boot with a systemd user unit, `~/.config/systemd/user/strata.service`:

```ini
[Unit]
Description=Strata local LLM server
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/sh %h/.local/bin/strata-autostart.sh
ExecStop=%h/.local/bin/strata launch --stop
TimeoutStartSec=15min

[Install]
WantedBy=default.target
```

`~/.local/bin/strata-autostart.sh` waits for Tailscale, then runs the launch line above:

```sh
#!/bin/sh
STRATA="$HOME/.local/bin/strata"
i=0; until tailscale ip -4 >/dev/null 2>&1 || [ $i -ge 60 ]; do sleep 2; i=$((i+1)); done
KEYS=$("$STRATA" launch --show-key 2>/dev/null | tail -n 1)
exec "$STRATA" launch --family qwen --model iq3_s --context 262144 --access tailscale \
  --port 8080 --api-key "$KEYS" --set vram-reserve-mib=1228 --yes --no-tui
```

```bash
systemctl --user enable --now strata.service
loginctl enable-linger "$USER"   # keep it running without a login session
```

Check it from the client:

```bash
curl -s -m5 http://STRATA_HOST:8080/health
# {"status": "ok", "max_context": 262144, "model": "MODEL", ..., "loaded": false}
```

`loaded: false` is normal. The model loads on the first request and unloads after 10 idle minutes, so the first call after a break is slow.

Strata speaks the Anthropic Messages API (`/v1/messages`), plus `/v1/chat/completions` and `/v1/responses`. `strata connect claude-code` and `strata connect pi` print the client settings for the running server; add `--check` to test a prompt and a tool call.

## 2. Store the API key (client)

```bash
mkdir -p ~/.config/strata
printf '%s' 'YOUR_STRATA_KEY' > ~/.config/strata/api_key
chmod 600 ~/.config/strata/api_key
```

Both worker paths read the key from this file. Never put it in a repo or a settings file.

## 3. Fallback worker: `strata-claude`

`~/.local/bin/strata-claude`:

```sh
#!/bin/sh
# Headless Claude Code running on Strata.
# Usage: strata-claude [claude -p flags] -- "task"
export ANTHROPIC_BASE_URL=http://STRATA_HOST:8080
export ANTHROPIC_AUTH_TOKEN="$(cat ~/.config/strata/api_key)"
M=MODEL
export ANTHROPIC_MODEL=$M ANTHROPIC_SMALL_FAST_MODEL=$M \
  ANTHROPIC_DEFAULT_HAIKU_MODEL=$M ANTHROPIC_DEFAULT_SONNET_MODEL=$M ANTHROPIC_DEFAULT_OPUS_MODEL=$M \
  CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1
unset ANTHROPIC_API_KEY
exec claude -p --model "$M" --strict-mcp-config "$@"
```

```bash
chmod +x ~/.local/bin/strata-claude
strata-claude --allowedTools "Read,Bash" -- "List the files in /tmp and say how many there are."
```

- Every model alias maps to the local model, so nothing falls back to Claude.
- `--strict-mcp-config` with no config means the worker loads no MCP servers.
- Keep the `--`. `--allowedTools` takes several values and eats the prompt otherwise.
- Two warnings on stderr are harmless (connectors disabled, unrecognized model).
- A small task takes about 25 to 30 seconds.

## 4. Default worker: SoL-Pi on Strata

[SoL-Pi](https://github.com/NVlabs/SoL-Pi) is an extension for the Pi coding agent. It ships `sol-pi-run` and a Claude Code plugin that adds `/sol-pi`. Tested with SoL-Pi commit `2195680` and its pinned Pi 1.3.0.

### 4.1 Tell Pi about Strata

`~/.pi/agent/models.json`:

```json
{
  "$schema": "https://pi.dev/schemas/models.schema.json",
  "providers": {
    "strata": {
      "name": "Strata",
      "baseUrl": "http://STRATA_HOST:8080",
      "api": "anthropic-messages",
      "apiKey": "!cat ~/.config/strata/api_key",
      "authHeader": true,
      "compat": { "supportsEagerToolInputStreaming": false, "supportsLongCacheRetention": false },
      "models": [
        {
          "id": "MODEL",
          "name": "Qwen Strata 256k",
          "input": ["text", "image"],
          "contextWindow": 262144,
          "maxTokens": 32768,
          "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 }
        }
      ]
    }
  }
}
```

`"apiKey": "!cat ..."` makes Pi run the command and read the key at request time. Drop `"image"` if your model has no vision.

`~/.pi/agent/settings.json`:

```json
{ "defaultProvider": "strata", "defaultModel": "MODEL" }
```

### 4.2 Install SoL-Pi

```bash
git clone https://github.com/NVlabs/SoL-Pi.git ~/src/SoL-Pi
cd ~/src/SoL-Pi
mkdir -p ~/.local/bin
ln -sf "$PWD/examples/sol-pi-skill/bin/sol-pi-run" ~/.local/bin/sol-pi-run
ln -sf "$PWD/examples/sol-pi-skill/bin/sol-pi-context" ~/.local/bin/sol-pi-context

sol-pi-run setup --reducer strata/MODEL
sol-pi-run doctor --live   # every line [ok]
```

- Pass `--reducer`. The built-in reducer route uses a cloud model you probably have no login for.
- Run `setup` in your own terminal (or with `!` in Claude Code). Claude Code's auto mode may block it because it runs external code.
- Everything lands in `~/.sol-pi` (pinned Pi, agent dir, runs). Your global Pi config is linked, not changed.
- The first `doctor --live` can take about 3 minutes if the model is cold.

### 4.3 Add the Claude Code plugin

```bash
claude plugin marketplace add "$PWD/examples/sol-pi-skill/claude-plugin"
claude plugin install sol-pi@sol-pi-local --scope user
```

Restart Claude Code. A personal skill named `sol-pi` in `~/.claude/skills` would shadow the plugin; remove it.

How a hand-off works: a prompt hook sees `/sol-pi`, starts the run in the background and blocks the prompt, so Claude spends no tokens on it. A second hook waits for the run and wakes the session with the result, the diff and the cost. Each run is kept in `~/.sol-pi/runs/<id>/` (`prompt.md`, `events.jsonl`, `stderr.log`, `result.json`).

SoL-Pi edits files and runs commands with your permissions. It has no sandbox.

### 4.4 Optional add-ons for the Pi worker

These go in SoL-Pi's agent dir `~/.sol-pi/agent`, not `~/.pi/agent`.

Pi packages:

```bash
PI=~/.sol-pi/pi/1.3.0/node_modules/.bin/pi
PI_CODING_AGENT_DIR=~/.sol-pi/agent $PI install npm:cc-safety-net@2.7.4 --approve    # blocks rm -rf outside cwd and similar
PI_CODING_AGENT_DIR=~/.sol-pi/agent $PI install npm:pi-web-access@0.47.0 --approve   # web search
```

Share your Claude Code skills by adding their folders to `"skills"` in `~/.sol-pi/agent/settings.json`. Keep the `-.../sol-pi` entry that `setup` wrote; it stops the worker from delegating to itself.

MCP servers go in `~/.sol-pi/agent/mcp.json`:

```json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": ["-y", "@playwright/mcp@0.0.84", "--headless"],
      "description": "Headless browser automation"
    }
  }
}
```

## 5. Tell Claude to use the workers

Add this to `~/.claude/CLAUDE.md`:

```markdown
# Subagents run on Strata

Always run subagent work on Strata, not with the Agent tool. Strata = local model on STRATA_HOST. The main session stays on Claude.

- Prefer `/sol-pi <task>` (shell: `sol-pi-run start|status|result`). Fallback only: `strata-claude --allowedTools "Read,Bash,Edit,Write,Grep,Glob" -- "<task>"`. Keep the `--`. Grant only the tools the task needs.
- Use Bash `run_in_background` for long tasks. Strata serves one request at a time, so run workers one after another, not in parallel.
- The worker starts cold: give it a self-contained prompt with paths, goal and expected output. Verify its result before relying on it; it is a weaker model.
- Check it is up first if unsure: `curl -s -m5 http://STRATA_HOST:8080/health`. If Strata is down, tell the user and ask; do not silently fall back to the Agent tool.
- Use the Agent tool only when the user explicitly asks for a Claude subagent.
```

## Usage

```text
/sol-pi add a --json flag to scripts/report.py and a test for it
/sol-pi -t medium summarise the errors in logs/app.log
```

From a shell, or from Claude through Bash:

```bash
sol-pi-run start -- "task"     # prints a run id
sol-pi-run status <id>
sol-pi-run result <id>
sol-pi-run list
```

Fallback:

```bash
strata-claude --allowedTools "Read,Grep,Glob" -- "Find where the config file is parsed in ./src and give file:line."
```

## Troubleshooting

| Symptom | Fix |
| --- | --- |
| Worker hangs at the start | The model is loading after idle. Wait, or check `strata status` on the GPU machine. |
| A second worker waits | Expected. One request at a time. |
| `strata-claude` runs with no prompt | You left out `--` after `--allowedTools`. |
| `doctor` fails on the reducer | Rerun `sol-pi-run setup --reducer strata/MODEL`. |
| 401 from Strata | Key file wrong, or the server was relaunched with a new key. |
| Works locally, not on claude.ai/code | Cloud sessions cannot reach your tailnet. |
