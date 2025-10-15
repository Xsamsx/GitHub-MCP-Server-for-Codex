# Configure GitHub MCP Server for Codex (macOS + Colima)

What: This setup lets Codex talk directly to GitHub through a secure Docker bridge so you can search repos, read files, or open issues with natural language.

## 🧠 TL;DR Quick Start

Need it fast? Copy/paste (swap in your macOS username and PAT).

```bash
# 1. Install & start the Docker runtime
brew install colima && colima start

# 2. Pull the GitHub MCP Server image (pin a release when you need stability)
docker pull ghcr.io/github/github-mcp-server:latest
# docker pull ghcr.io/github/github-mcp-server:vX.Y.Z   # pin a specific release

# 3. Create ~/.github-mcp.env and confirm Docker can read the PAT
echo "GITHUB_PERSONAL_ACCESS_TOKEN=github_pat_XXXXXXXXXXXXXXXXXXXXXXXX" > ~/.github-mcp.env
chmod 600 ~/.github-mcp.env
docker run --rm --env-file ~/.github-mcp.env alpine sh -lc 'test -n "$GITHUB_PERSONAL_ACCESS_TOKEN" && echo OK'

# 4. Add this block to ~/.codex/config.toml (update the username)
# 5. Launch Codex and approve the docker run when prompted
codex

# 6. Smoke test inside Codex
Using only the `github` MCP tools, call `get_me`.
```

```toml
[mcp_servers.github]
command = "docker"
args = [
  "run","-i","--rm","--pull=always",          # keep the image fresh (swap to :vX.Y.Z + drop --pull for pinned builds)
  "--name","github-mcp",
  "--env-file","/Users/YourMacUsername/.github-mcp.env",
  "ghcr.io/github/github-mcp-server:latest","stdio"
]
```

> ⚠️ Security tip: `~/.github-mcp.env` holds a live token. Keep the file private (`chmod 600`) and never commit it to git.

---

## 🧱 Install Prerequisites

- macOS with Homebrew  
- [Colima](https://github.com/abiosoft/colima) (lightweight Docker runtime)  
- OpenAI Codex (CLI or VS Code extension)  
- A GitHub account (personal or organization)

```bash
brew install colima
colima start           # launches the Docker daemon inside a VM
docker ps              # quick sanity check
```

**Windows / Linux?** Use Docker Desktop (or WSL2 + Docker) on Windows, and plain Docker Engine on Linux. The Codex `args` list is identical—just change the `--env-file` path to match your platform (for example, `C:\\Users\\you\\.github-mcp.env` on Windows, `/home/you/.github-mcp.env` on Linux).

## 🔑 Create Your GitHub Token

Create a personal access token (the new fine-grained type) with these minimum settings:

- Resource owner: your GitHub user (for example, `Xsamsx`)  
- Repository access: all repositories, or pick the repos you need  
- Repository permissions (tweak as required):
  - Metadata: Read
  - Contents: Read (or Read & write if you’ll push changes)
  - Issues: Read (or Read & write)
  - Pull requests: Read (or Read & write)
- Optional: add Actions, Administration, Commit statuses, etc., for automation

Authorize the token for any org that enforces SAML SSO. GitHub gives you a value like `github_pat_XXXXXXXXXXXXXXXX...`.

### Scope Quick Reference

| Action | Minimum fine-grained scopes | Notes |
| --- | --- | --- |
| Read repositories & files | Metadata: Read, Contents: Read | Required for repo discovery, default branch lookup, `get_file_contents` |
| Search issues & pull requests | Issues: Read, Pull requests: Read | Add write if you plan to edit or create issues/PRs |
| Create issues | Issues: Read & write | Select the target repositories and re-authorize for SAML orgs |
| Push / update files | Contents: Read & write | Enables `create_or_update_file` and commit operations |
| See org repos | Metadata: Read + org-level access | Add `read:org` (classic) or the equivalent fine-grained org scope |

## 🔒 Store the Token Locally

Create a hidden env file in your home directory and lock down permissions:

```bash
echo "GITHUB_PERSONAL_ACCESS_TOKEN=github_pat_XXXXXXXXXXXXXXXXXXXXXXXX" > ~/.github-mcp.env
chmod 600 ~/.github-mcp.env
```

Sanity-check that Docker can read the token (if “OK” does not print, fix the env file before opening Codex):

```bash
docker run --rm --env-file ~/.github-mcp.env alpine sh -lc 'test -n "$GITHUB_PERSONAL_ACCESS_TOKEN" && echo OK'
```

Optional guardrail so the token never lands in git history:

```bash
echo ".github-mcp.env" >> ~/.gitignore_global
git config --global core.excludesfile ~/.gitignore_global
```

### Optional: Easiest auth via GitHub CLI (skip the env file)

Already logged in with `gh`? You can run the server via the official MCP extension—no PAT file needed:

```bash
gh extension install shuymn/gh-mcp
gh mcp run    # starts github-mcp-server in Docker using your gh auth
```

Codex sees the same MCP endpoint; just point your config at the command that `gh mcp run` prints (or let Codex launch it manually by wiring the extension command into `[mcp_servers.github]`).

## ⚙️ Configure Codex

Edit `~/.codex/config.toml` and paste the block from the quick start (update the username). `--pull=always` keeps the container up to date; when you pin a tag such as `ghcr.io/github/github-mcp-server:vX.Y.Z`, remove `--pull=always` for reproducible builds.

### Narrow toolsets (keep context lean)

The server exposes multiple toolsets. Trim them to improve Codex’s tool selection and reduce context:

```toml
args = [
  "run","-i","--rm","--pull=always",
  "--name","github-mcp",
  "--env-file","/Users/YourMacUsername/.github-mcp.env",
  "ghcr.io/github/github-mcp-server:latest",
  "--toolsets","repositories,issues","stdio"
]
```

When running the binary directly, the flag is identical:

```bash
github-mcp-server --toolsets=issues,contents stdio
```

Check `github-mcp-server --help` for the current toolset list (`repositories`, `issues`, `pull_requests`, `workflows`, etc.).

### GitHub Enterprise (GHES or custom hosts)

Add your hostname through `GITHUB_HOST`:

```toml
args = [
  "run","-i","--rm","--pull=always",
  "--name","github-mcp",
  "--env-file","/Users/YourMacUsername/.github-mcp.env",
  "-e","GITHUB_HOST=https://your-org.ghe.com",
  "ghcr.io/github/github-mcp-server:latest","stdio"
]
```

## 🚀 Launch & Verify

```bash
colima start        # ensure Docker is running
codex               # launch Codex; approve the docker run when prompted
docker ps | grep github-mcp   # should list one running container
```

In Codex, open the MCP view (or run `/mcp`) and confirm `github` is listed.

### Prompt Pack (copy & paste)

Use any of these prompts to prove the server works—each stays scoped to the GitHub MCP tools.

- **List my repos (table):**
  ```
  Using only the github MCP tools, call get_me. Then call search_repositories with q="user:{login}" and per_page=100. Return a table with columns: full_name | visibility | description. If over 100, paginate and combine.
  ```
- **Open a repo and preview README:**
  ```
  With the github MCP tools, get the default branch of <owner>/<repo>, then fetch README.md via get_file_contents. Show the first 30 lines with line numbers.
  ```
- **Find issues I created last 7 days:**
  ```
  Using only github MCP tools, query issues with q="author:{login} is:issue created:>=7d"; return repository | number | title | state | created_at.
  ```
- **Create a test issue (requires write scopes):**
  ```
  Using github MCP tools, create an issue in <owner>/<repo> titled "MCP test" with body "hello from Codex." Return the issue URL.
  ```
- **Search code in my org for a string:**
  ```
  With github MCP, run a code search q="org:<org> <term> in:file"; list repo | path | html_url for the first 50 matches.
  ```

## 🔄 Day-2 Ops & Diagnostics

- Start / stop routine:
  - Start Docker: `colima start`
  - Launch Codex: `codex`
  - Quit with `Ctrl+C` (avoid `Ctrl+Z`); the container vanishes thanks to `--rm`
  - Manual stop: `docker stop github-mcp`
  - Clean stray containers:  
    ```bash
    docker ps -q --filter ancestor=ghcr.io/github/github-mcp-server | xargs -r docker stop
    ```
- Refresh the image periodically: `docker pull ghcr.io/github/github-mcp-server:latest` (or bump your pinned tag).
- Rotate PATs on a schedule—replace the value in `~/.github-mcp.env` and restart Codex.
- Logs and errors: check `~/.codex/logs/latest.log` if Codex fails to start the server. You can tune timeouts via `startup_timeout_sec` or `tool_timeout_sec` under `[mcp_servers.github]` if large requests time out.

## 🛠️ Troubleshooting

- **401 Bad credentials**  
  `~/.github-mcp.env` is missing or the token expired. Re-run the sanity check command and generate a new PAT if needed.
- **0 repositories found**  
  The PAT lacks repository access. Edit the token and add Metadata / Contents scopes plus the specific repos or “All repositories”.
- **Org repositories missing**  
  Grant organization scopes (`read:org` or the fine-grained equivalent) and approve SAML SSO for the token.
- **Docker command rejected (`unrecognized flag`)**  
  Update the image (`docker pull ...`) to match the flags you’re using, or remove optional flags like `--toolsets`.
- **Codex never lists `/mcp/github`**  
  Run `/mcp` to refresh; if still missing, inspect `~/.codex/logs/latest.log` for the docker stderr and confirm Colima is running.

### Advanced: HTTP mode for quick curls

Prefer HTTP over stdio? You can run the server in HTTP mode for lightweight debugging:

```bash
docker run --rm -p 3000:3000 --env-file ~/.github-mcp.env ghcr.io/github/github-mcp-server:latest http --listen 0.0.0.0:3000
curl -s http://localhost:3000/tools/list | jq
```

Swap ports or flags as needed—see `github-mcp-server http --help` for full options. Codex still expects stdio, so reserve HTTP for manual testing.

## 🛡️ Security Notes

- The PAT stays on disk only in `~/.github-mcp.env`.  
- File permissions `600` keep other users on the machine out.  
- The token never ends up in Docker images, Codex configs, or git history (if you follow the global ignore tip).  
- Rotate the PAT periodically and update the env file.  
- If you adopt the GitHub CLI extension, the token remains inside your `gh` keychain instead of the env file.

## 🪛 Dockerless Fallback (build from source)

No Docker? Build the server directly with Go 1.21+:

```bash
git clone https://github.com/github/github-mcp-server.git
cd github-mcp-server
go build ./cmd/github-mcp-server
./github-mcp-server stdio
```

Point Codex at the resulting binary (for example, `/Users/you/github-mcp-server/github-mcp-server`) instead of `docker`.

## 📝 Appendix

- Show hidden files (dotfiles): `ls -la ~`  
- Edit the env file: `nano ~/.github-mcp.env`  
- Confirm the token with the REST API (requires `jq`):  
  ```bash
  curl -s -H "Authorization: Bearer $(cut -d= -f2 ~/.github-mcp.env)" https://api.github.com/user | jq .login
  ```

PRs welcome for GHES tweaks, multi-org setups, additional automation tips, or new prompt ideas.
