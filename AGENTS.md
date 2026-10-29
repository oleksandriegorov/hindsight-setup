# AGENTS.md — install Hindsight memory for Claude Code (macOS, Linux, Windows with WSL2)

You are an agent. This file is the procedure. Follow the steps from top to bottom. Run each command exactly as written. [README.md](README.md) gives the reasons; read it only when a step is not clear.

The author tested this procedure on macOS (Apple Silicon). The Linux and Windows branches use the same steps and are not tested. When a step is different for a platform, it has a **macOS**, **Linux** or **Windows** line. Windows uses the Linux branch inside WSL2.

Paths such as `files/…` are relative to this folder (the folder that holds `AGENTS.md`). If your shell does not keep the folder between commands, start each command with `cd <this folder> &&`.

Each command has an **Expected** line. If the output is different, stop. Show the output to the human and wait for an answer. Replace only the `<…>` placeholders. Do not change a command, a path, a port or a version to make it work. A command without an **Expected** line is correct when it gives no error.

## Rules

- Never print, copy or move the content of `auth.json`, `hindsight.env`, an API key or a token. Never ask the human to paste a key in chat.
- The human completes every login in the browser. You only start the login command (step 5).
- Never use `~/.codex` (the Codex CLI login) for Hindsight. Use `~/.hindsight/codex` only.
- Never delete the Docker volume `hindsight-data`. It holds the memory.
- Never overwrite `~/.claude/settings.json`. The installer in step 8 merges the hooks into it.
- Never run `codex` or `claude` without arguments. They open an interactive program and block you. Never run `codex` without `CODEX_HOME`: that logs in to `~/.codex`.
- Never run `sudo` without the human's approval. Show the command and the reason first.
- At a `FAIL` in step 2, do the action in the table. If the table says "Stop", or the human cannot do the action, stop. After each fix, run the check again. Never work around a gate.
- Some fixes need a new login or a new shell. Then tell the human to start you again and to say "continue at step 2". Step 1 saves the answers for this case.
- Change nothing before step 4. Steps 0–3 only ask and read; the only exceptions are the fixes from the table in step 2 and the answer file in step 1.

## 0. Find the platform

```bash
uname -s; uname -m; grep -qi microsoft /proc/version 2>/dev/null && echo WSL2 || true
```

| Output | Platform | Branch |
|---|---|---|
| `Darwin` | macOS | macOS |
| `Linux` without `WSL2` | Linux | Linux |
| `Linux` and `WSL2` | Windows, inside WSL2 | Linux, plus the **Windows** lines |

**Windows, not in WSL2** (for example PowerShell): stop. Tell the human to do these steps, then to start you again in the WSL2 shell:

1. In PowerShell as administrator, run `wsl --install`, then restart the computer.
2. Install Docker Desktop. In **Settings → Resources → WSL integration**, turn on the Linux distribution.
3. Open the WSL2 shell (for example "Ubuntu") and install Claude Code there.

Expected architecture: `x86_64`, `aarch64` or `arm64`. Any other value: stop. The image has no build for this CPU.

## 1. Ask the human

If `~/.hindsight/install-answers.txt` exists and the human says "continue at step 2", read the answers from that file and go to step 2. Otherwise ask the four questions in one message. Give each option with its example. If the human gives no preference for a question, use its default.

**Q1. LLM provider.** "Which account pays for the memory work?"

- **A** (default): a ChatGPT Plus or Pro subscription, provider `openai-codex`, no API key. For personal use only.
- **B**: an Anthropic Console API key (`sk-ant-api03-…`) with credit, model `claude-haiku-4-5`. For example, a work machine where a subscription is not allowed.

**Q2. Shared bank for "No folder" sessions.** "Send all Claude desktop sessions without a project folder to one bank, `coding-agent::scratch`?"

- **Yes** (default): a quick question from last week is still in memory today.
- **No**: each "No folder" session gets its own bank, and later sessions do not read it.

**Q3. Git backup of `~/.claude`.** "Does a tool such as dotclaude-sync back up `~/.claude` to git?"

- **Yes**: step 8 adds the installed skill to `~/.claude/.gitignore`, because its example tokens trigger the secret scan.
- **No** (default): nothing to do.

**Q4. Import of past sessions.** "Import your past Claude Code sessions for one project?"

- **No** (default): memory starts with the next session.
- **Yes, for `<project folder>`**, for example `~/code/my-project`: Hindsight reads those transcripts one time. This uses model quota. If the human says Yes without a folder, ask for the folder.

Also tell the human: "Hindsight sends each conversation to the provider that you select, for fact extraction. Do not use it for sessions with secrets that must not leave the machine."

Save the answers, because a fix in step 2 can need a restart. Replace `A yes no none` with the answers, for example `B no yes /home/me/code/my-project`:

```bash
mkdir -p -m 700 ~/.hindsight && printf 'provider=%s\nscratch_bank=%s\ngitignore=%s\nimport_folder=%s\n' A yes no none > ~/.hindsight/install-answers.txt && cat ~/.hindsight/install-answers.txt
```

Expected: the four lines with the answers. The steps below refer to them as Q1–Q4.

## 2. Examine the requirements (gates)

Run this check. It needs only Python 3 and changes nothing. If `python3` is missing: **macOS:** run `xcode-select --install` and wait for the dialog to finish. **Linux:** ask the human to install the `python3` package.

```bash
python3 - <<'EOF'
import os, platform, shutil, socket, subprocess
def out(cmd):
    try: return subprocess.check_output(cmd, stderr=subprocess.DEVNULL, text=True).strip()
    except Exception: return ""
gb = lambda b: round(b / 1e9, 1)
ram = int(out(["sysctl", "-n", "hw.memsize"]) or 0) if platform.system() == "Darwin" else os.sysconf("SC_PAGE_SIZE") * os.sysconf("SC_PHYS_PAGES")
dmem = int(out(["docker", "info", "--format", "{{.MemTotal}}"]) or 0)
disk = shutil.disk_usage(os.path.expanduser("~")).free
node = out(["node", "-v"]).lstrip("v").split(".")[0]
rows = [
    ("arch 64-bit", platform.machine(), platform.machine() in ("x86_64", "aarch64", "arm64"), True),
    ("cpu cores >= 4 (8 rec.)", os.cpu_count(), os.cpu_count() >= 4, os.cpu_count() >= 8),
    ("host RAM GB >= 8 (16 rec.)", gb(ram), gb(ram) >= 8, gb(ram) >= 16),
    ("docker answers", "yes" if dmem else "no", dmem > 0, True),
    ("docker memory GB >= 4 (6 rec.)", gb(dmem), gb(dmem) >= 4, gb(dmem) >= 6),
    ("docker compose v2", out(["docker", "compose", "version", "--short"]) or "missing", bool(out(["docker", "compose", "version", "--short"])), True),
    ("free disk GB >= 10 (20 rec.)", gb(disk), gb(disk) >= 10, gb(disk) >= 20),
    ("node >= 20", node or "missing", node.isdigit() and int(node) >= 20, True),
    ("curl", shutil.which("curl") or "missing", bool(shutil.which("curl")), True),
    ("claude", shutil.which("claude") or "missing", bool(shutil.which("claude")), True),
    ("codex (option A)", shutil.which("codex") or "missing", True, bool(shutil.which("codex"))),
]
for port in (8888, 9999):
    used = socket.socket().connect_ex(("127.0.0.1", port)) == 0
    rows.append((f"port {port} free", "in use" if used else "free", not used, True))
for name, value, ok, rec in rows:
    print(f"{'FAIL' if not ok else 'WARN' if not rec else 'OK  '}  {name}: {value}")
EOF
```

Expected: `OK` on each line. Then act on each `FAIL` and `WARN`:

| Line | Result | Action |
|---|---|---|
| any | `WARN` | Tell the human the value and the recommended value. Continue. |
| `arch` | `FAIL` | Stop. The image has no build for this CPU. |
| `cpu cores`, `host RAM`, `free disk` | `FAIL` | Stop. Tell the human the value and the minimum. |
| `docker answers` | `FAIL` | **macOS / Windows:** ask the human to start Docker Desktop, then run the check again. **Linux:** run `docker info`. On `permission denied`, ask the human to approve `sudo usermod -aG docker $USER`, then to log out and in again. Tell the human that the `docker` group gives root-level access to the machine. After the new login, the human starts you again (see the rules). If Docker is not installed, ask the human to install Docker Engine with the Compose plugin (<https://docs.docker.com/engine/install/>). |
| `docker memory` | `FAIL` | If `docker answers` is also `FAIL`, ignore this line: the value is 0 because Docker did not answer. Fix Docker access first. Otherwise: **macOS:** ask the human to set **Docker Desktop → Settings → Resources → Memory** to 6 GB. **Windows:** ask the human to set `memory=8GB` under `[wsl2]` in `%UserProfile%\.wslconfig`, then to run `wsl --shutdown` in PowerShell. **Linux:** Docker uses all host RAM, so this is a host RAM problem. |
| `docker compose v2` | `FAIL` | **macOS / Windows:** update Docker Desktop. **Linux:** ask the human to install the `docker-compose-plugin` package. |
| `node` | `FAIL` | **macOS:** `brew install node` (if `brew` is missing, ask the human to install Homebrew from <https://brew.sh>). **Linux / Windows:** ask the human to install `nvm` with the install command on <https://github.com/nvm-sh/nvm>, then to run `nvm install 22 && nvm alias default 22` in a new shell. With `nvm`, `npm install -g` needs no `sudo`. The hooks use the `node` that Claude Code finds. So the human must close Claude Code, open a new shell, examine that `node -v` shows 22, and start you again there (see the rules). |
| `curl` | `FAIL` | **macOS:** stop, `curl` is part of macOS; the machine is not standard. **Linux / Windows:** ask the human to install the `curl` package. |
| `claude` | `FAIL` | Stop. Ask the human to install Claude Code in this shell. |
| `codex` | `WARN` | For Q1 = A only: `npm install -g @openai/codex`. For Q1 = B: ignore. |
| `port` | `FAIL` | Run `docker ps --filter name=^hindsight$ --format '{{.Names}}'`. If it prints `hindsight`, a Hindsight container already exists: this is correct, step 3 handles it. If not, stop and tell the human that another program uses the port. |

**Q1 = A only.** Examine that the account has the two models. This reads a cache file of the Codex CLI; it never runs `codex`:

```bash
test -f ~/.codex/models_cache.json && grep -oE '"(gpt-5\.6-luna|gpt-6-luna)"' ~/.codex/models_cache.json | sort -u || echo "NO CACHE"
```

Expected: `"gpt-5.6-luna"` and `"gpt-6-luna"`. Other results:

- Only `"gpt-5.6-luna"`: keep the note "no gpt-6-luna" for step 6, which changes the model.
- `NO CACHE`: the Codex CLI never ran on this machine. Continue with both models. Step 7 examines the log.
- No output: the account has neither model. Stop and tell the human.

## 3. Detect an existing install

```bash
echo "container: $(docker ps -a --filter name=^hindsight$ --format '{{.Names}} {{.Status}}')"; echo "volume: $(docker volume ls -q --filter name=^hindsight-data$)"; echo "files: $(ls ~/.hindsight 2>/dev/null | tr '\n' ' ')"; echo "hooks: $(grep -c 'hindsight/coding-agents' ~/.claude/settings.json 2>/dev/null)"; echo "connector: $(claude mcp get hindsight 2>/dev/null | grep Status)"
```

Expected: five labeled lines. The `hooks:` number is the **hook count** in the steps below.

| Output | What to do |
|---|---|
| `container: hindsight Up …` | A server runs. Tell the human that it keeps its current provider, so Q1 does not apply. Do step 4. Skip steps 5, 6 and 7. In step 8, run the installer only if the hook count is below 3 or the `connector:` line lacks `Connected`. Do the Q2, Q3 and Q4 parts of step 8 as answered. Then do steps 9 and 10. |
| `container: hindsight Exited …` | A server exists but does not run. Do the same as for `Up`, but in step 7 run only `docker start hindsight` and the health check. |
| `container:` empty, `volume: hindsight-data` | The memory exists without a server. Keep the volume. Do all steps. Step 4 backs up old files in `~/.hindsight`, and step 6 replaces them. Step 7 does not make the volume again. |
| `container:` empty, `volume:` empty | New machine. Do all steps. Old files in `~/.hindsight` do not change this. |

## 4. Make the folders and back up

```bash
mkdir -p -m 700 ~/.hindsight/codex ~/.hindsight/backup && TS=$(date +%Y%m%d-%H%M%S) && for f in ~/.claude/settings.json ~/.claude/.gitignore ~/.hindsight/docker-compose.yml ~/.hindsight/hindsight.env ~/.hindsight/coding-agent.json; do [ -f "$f" ] && cp -p "$f" ~/.hindsight/backup/$(basename "$f").$TS; done; ls ~/.hindsight/backup; true
```

Expected: one copy of each file that existed. An empty list is correct on a new machine.

## 5. Codex login (Q1 = A only; the human completes it)

Start this command in a terminal that the human can see, for example a terminal tab of your app. If you have no such terminal, give the human the command to run. If the human says that the machine has no browser, add `--device-auth` to the command.

**Windows:** if the browser does not return to the terminal after the login, add `--device-auth` to the command and run it again.

```bash
CODEX_HOME=~/.hindsight/codex codex login
```

Tell the human: "Log in to ChatGPT in the browser and approve. Tell me when the terminal shows `Successfully logged in`." Wait for the answer. Then verify without a read of the file:

```bash
test -s ~/.hindsight/codex/auth.json && echo ok
```

Expected: `ok`. If there is no output, the login is not complete. Ask the human again. Do not continue. If the login fails, show the human the error from the terminal and wait.

**macOS:** Docker Desktop maps the file owner. Do nothing more.

**Linux / Windows:** the container runs as UID 1000 and must write this folder, because the tokens rotate. Examine your UID:

```bash
id -u
```

Expected: `1000`. If the output is different, ask the human to approve this command. It gives the folder to UID 1000, and keeps access for the human through the human's group:

```bash
sudo chown -R 1000:$(id -g) ~/.hindsight/codex && sudo chmod -R u+rwX,g+rwX ~/.hindsight/codex && echo ok
```

Expected: `ok`.

NOTE: A later `codex login` for Hindsight needs this command again, because the container makes new token files with UID 1000.

## 6. Install the configuration

**Q1 = A:**

```bash
cp files/docker-compose.yml ~/.hindsight/docker-compose.yml && cp files/hindsight.env.example ~/.hindsight/hindsight.env && chmod 600 ~/.hindsight/hindsight.env && echo ok
```

Expected: `ok`.

Only if step 2 gave the note "no gpt-6-luna" (not for `NO CACHE`), change both model values:

```bash
python3 -c "import os;p=os.path.expanduser('~/.hindsight/docker-compose.yml');s=open(p).read().replace('gpt-6-luna','gpt-5.6-luna');open(p,'w').write(s)" && grep -c "gpt-6-luna" ~/.hindsight/docker-compose.yml
```

Expected: `0`.

**Q1 = B:**

```bash
cp files/docker-compose.anthropic.yml ~/.hindsight/docker-compose.yml && cp files/hindsight.env.anthropic.example ~/.hindsight/hindsight.env && chmod 600 ~/.hindsight/hindsight.env && echo ok
```

Expected: `ok`.

Tell the human: "Open `~/.hindsight/hindsight.env` in an editor (**macOS:** `open -e ~/.hindsight/hindsight.env`; **Linux / Windows:** `nano ~/.hindsight/hindsight.env`). Paste your Anthropic key directly after `HINDSIGHT_API_LLM_API_KEY=`, without quotes or spaces. Save the file. Tell me when it is done." Wait for the answer. Then verify without a read of the key:

```bash
grep -c '^HINDSIGHT_API_LLM_API_KEY=sk-ant-api03-' ~/.hindsight/hindsight.env
```

Expected: `1`. If the output is `0`, ask the human to examine the line again. Do not continue. If the human has only a subscription token (`sk-ant-oat01-…`), it does not work: offer option A.

## 7. Start the server

```bash
docker volume inspect hindsight-data >/dev/null 2>&1 || docker volume create hindsight-data
```

```bash
(cd ~/.hindsight && docker compose up -d)
```

The first start downloads about 3.5 GB. Wait for the health check. This command can run for 10 minutes: run it with your longest timeout, or in the background.

```bash
ok=; for i in $(seq 1 120); do curl -sf http://127.0.0.1:8888/health && { ok=1; break; }; sleep 5; done; [ -n "$ok" ] || echo "TIMEOUT: run docker logs hindsight"
```

Expected: `{"status":"healthy","database":"connected",...}`. On `TIMEOUT`, show the human the last 50 lines of `docker logs hindsight` and wait.

The server verifies the models up to 2 minutes after the health check. Run this command until the expected lines appear, with a wait of 30 seconds between runs, for at most 3 minutes after the health check:

```bash
docker logs hindsight 2>&1 | grep -E "Codex LLM verified|Anthropic connection verified|LLM \("
```

Expected for Q1 = A: `Codex LLM verified: gpt-5.6-luna`, `Codex LLM verified: gpt-6-luna`, and `LLM (reflect)` and `LLM (consolidation)` lines with `provider=openai-codex`. More `LLM (` lines and repeated lines are correct. If step 6 changed the model, only `gpt-5.6-luna` appears.

Expected for Q1 = B: `Anthropic connection verified successfully`.

If step 2 gave `NO CACHE`, `Codex LLM verified: gpt-5.6-luna` is present, and `Codex LLM verified: gpt-6-luna` is absent after 3 minutes, the account has no `gpt-6-luna`. Run the `python3` command from step 6, then `(cd ~/.hindsight && docker compose up -d --force-recreate)`, and examine the log again.

If the log shows a permission error on `/home/hindsight/.codex`: **macOS:** run `chmod 700 ~/.hindsight/codex && chmod 600 ~/.hindsight/codex/auth.json`. **Linux / Windows:** do the `chown` command of step 5 again (it needs `sudo` and the human's approval). Then run `(cd ~/.hindsight && docker compose up -d --force-recreate)`, and do the health check and the log check again.

## 8. Connect Claude Code

Run this from a folder that is not a project:

```bash
cd "${TMPDIR:-/tmp}" && npx -y @vectorize-io/hindsight-coding-agents@0.8.2 install claude-code --server self-hosted --api-url http://127.0.0.1:8888
```

Expected: `hooks merged into ~/.claude/settings.json`, `skill installed at ~/.claude/skills/hindsight-coding-agent`, `MCP server registered`. The installer keeps the other hooks. It also makes `~/.hindsight/coding-agent.json` and the runtime in `~/.hindsight/coding-agents/`.

NOTE: The runtime updates itself once a day (`autoUpdate`), so the version can be newer than `0.8.2` later. To stop this, the human can add `"autoUpdate": false` to `~/.hindsight/coding-agent.json`. Do not add it unless the human asks.

Do the Q4 part before the Q2 part, because the Q4 command runs the installer again.

**Q4 = Yes.** Run this in the project folder that the human named. Use the absolute path, for example `/Users/<name>/code/my-project`, because `~` does not expand inside the quotes:

```bash
cd "<project folder>" && npx -y @vectorize-io/hindsight-coding-agents@0.8.2 install claude-code --server self-hosted --api-url http://127.0.0.1:8888 --import-conversations
```

Expected: the same three lines as above, and lines about the imported conversations.

**Q2 = Yes.** The "No folder" sessions of the Claude desktop app start in a workspace folder. The mapping sends that folder to one bank.

**Linux / Windows:** a CLI session always starts in a folder that the human selects, so there is no default path. Skip the Q2 part, unless the human gives you a folder for "No folder" sessions. Then use that path in the `python3` command below in place of the default.

**macOS:** examine the default workspace path:

```bash
ls -d ~/Library/Application\ Support/Claude/scratch-workspaces
```

Expected: the path. If the output is `No such file or directory`, ask the human for the "No folder" workspace path of the Claude desktop app. Use that path in the next command in place of the default.

Merge the mapping. The command keeps all other keys:

```bash
python3 -c "import json,os;p=os.path.expanduser('~/.hindsight/coding-agent.json');d=json.load(open(p));d.setdefault('mapPathToBank',{})[os.path.expanduser('~/Library/Application Support/Claude/scratch-workspaces')]='coding-agent::scratch';json.dump(d,open(p,'w'),indent=2);print(d['mapPathToBank'])"
```

Expected: a dictionary with the path and `coding-agent::scratch`. The file then has the keys of [files/coding-agent.json](files/coding-agent.json); more keys from the installer are correct.

**Q3 = Yes:**

```bash
grep -qx '/skills/hindsight-coding-agent/' ~/.claude/.gitignore || echo '/skills/hindsight-coding-agent/' >> ~/.claude/.gitignore; grep -c '/skills/hindsight-coding-agent/' ~/.claude/.gitignore
```

Expected: `1`.

## 9. Verify

```bash
echo "health: $(curl -s http://127.0.0.1:8888/health | head -c 60)"; echo "hooks: $(grep -c 'hindsight/coding-agents' ~/.claude/settings.json)"; echo "connector: $(claude mcp get hindsight | grep Status)"; echo "shm: $(docker inspect hindsight --format '{{.HostConfig.ShmSize}}')"; docker port hindsight
```

Expected: `health:` with `"status":"healthy"`, `hooks:` with 3 or more, `connector:` with `Connected`, `shm: 1073741824`, and two port lines that end with `127.0.0.1:8888` and `127.0.0.1:9999`. If a port line has `0.0.0.0`, stop: the server is open to the network. If `shm:` is `67108864`, the compose file of the container has no `shm_size: 1g`. Add it and recreate the container, or new banks fail. Any other `shm:` value: stop and tell the human.

Smoke-test the recall hook. The folder name `hs-smoke` makes the test bank `coding-agent::hs-smoke`. The last command deletes only that bank.

```bash
D="${TMPDIR:-/tmp}/hs-smoke" && mkdir -p "$D" && echo '{"session_id":"smoke","hook_event_name":"UserPromptSubmit","prompt":"test","cwd":"'"$D"'","transcript_path":"/dev/null"}' | node ~/.hindsight/coding-agents/dist/claude-hook.js | head -c 120; echo; curl -s -o /dev/null -w "%{http_code}\n" -X DELETE "http://127.0.0.1:8888/v1/default/banks/coding-agent::hs-smoke"; rm -rf "$D"
```

Expected: JSON that starts with `{"systemMessage":`, then `200`. A `404` in place of `200` is also correct: the hook made no bank for the test folder.

## 10. Report to the human

Report in a short list:

- the platform (step 0) and the `WARN` lines of step 2, if any;
- the server health, and the model and effort level for retain, consolidation and reflect. Read them from the `environment:` block of `~/.hindsight/docker-compose.yml`. For Q1 = B: `claude-haiku-4-5`, no effort level;
- the web UI URL: <http://127.0.0.1:9999>;
- the hook count and the connector status;
- the answers to Q1–Q4, the optional steps that you did, and the steps that you skipped;
- the backups in `~/.hindsight/backup/`;
- the bank rule: each session uses the bank `coding-agent::<folder name>` of the folder where it starts. "No folder" sessions use `coding-agent::scratch` (Q2 = Yes) or one bank each (Q2 = No);
- that a **new** Claude Code session is necessary, because the current session does not load the new hooks.
