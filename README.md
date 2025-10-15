# Configure GitHub MCP Server for Codex (macOS + Colima)

This guide explains how to connect OpenAI Codex to the GitHub MCP Server using Docker (via Colima) and a GitHub Personal Access Token (PAT) stored in a local `.env` file. You get a clean startup/shutdown flow, keep the token out of your Codex config, and avoid Docker Desktop altogether.

## TL;DR Quick Start

Need it fast? Copy/paste the block below (replace the PAT value and your macOS username).

```bash
# 1. Install and start Colima (Docker runtime)
brew install colima
colima start

# 2. Pull the GitHub MCP Server image (pin a release when available)
docker pull ghcr.io/github/github-mcp-server:latest
# docker pull ghcr.io/github/github-mcp-server:vX.Y.Z   # pin a tag once you select a release

# 3. Create ~/.github-mcp.env with your fine-grained PAT and verify the container sees it
echo "GITHUB_PERSONAL_ACCESS_TOKEN=github_pat_XXXXXXXXXXXXXXXXXXXXXXXX" > ~/.github-mcp.env
chmod 600 ~/.github-mcp.env
docker run --rm --env-file ~/.github-mcp.env alpine sh -lc 'test -n "$GITHUB_PERSONAL_ACCESS_TOKEN" && echo OK'

# 4. Paste the snippet below into ~/.codex/config.toml (swap in your macOS username)
# 5. Launch Codex and approve the docker run when prompted
codex

# 6. Prompt Codex to exercise the server
Using only the `github` MCP tools, call `get_me`.
```

```toml
[mcp_servers.github]
command = "docker"
args = [
  "run","-i","--rm","--pull=always",
  "--name","github-mcp",
  "--env-file","/Users/YourMacUsername/.github-mcp.env",
  "ghcr.io/github/github-mcp-server:latest","stdio"
]
```

Swap `:latest` for a tagged release (`:vX.Y.Z`) when you want reproducible builds across teammates.

## Prerequisites

- macOS with Homebrew
- [Colima](https://github.com/abiosoft/colima) (Docker runtime)
- OpenAI Codex installed (CLI or IDE extension)
- A GitHub account (personal or organization)

```bash
brew install colima
colima start           # starts the Docker daemon inside a lightweight VM
docker ps              # sanity check (should show no error)
```

## Before you start: Pull the server image

Avoid first-run surprises by pulling the container ahead of time:

```bash
docker pull ghcr.io/github/github-mcp-server:latest
# docker pull ghcr.io/github/github-mcp-server:vX.Y.Z   # pin once you pick a release
```

The README examples use `:latest` together with `--pull=always` so you automatically receive updates. For production workflows, pin a specific tag and remove `--pull=always`.

## 1. Create a GitHub Personal Access Token (PAT)

Create a fine-grained PAT with the following minimum settings:

- Resource owner: your GitHub user (e.g., `Xsamsx`)
- Repository access: All repositories (or explicitly select the ones you need)
- Minimum repository permissions (adjust as needed):
  - Metadata: Read
  - Contents: Read (or Read & write if you plan to push)
  - Issues: Read (or Read & write)
  - Pull requests: Read (or Read & write)
- Optional: add Actions, Administration, Commit statuses for deeper automation

If your organization uses SAML SSO, authorize the token for that org after creation. You will receive a token shaped like `github_pat_XXXXXXXXXXXXXXXX...`.

### Scope Reference

| Action | Fine-grained scopes | Notes |
| --- | --- | --- |
| Read repositories and files | Metadata: Read, Contents: Read | Required for repo discovery, default branch lookup, `get_file_contents` |
| Search issues and pull requests | Issues: Read, Pull requests: Read | Add write for editing/creation workflows |
| Create new issues | Issues: Read & write | Select target repositories and re-authorize for SAML orgs |
| Push or edit files | Contents: Read & write | Enables `create_or_update_file` and commit operations |
| List org repositories | Metadata: Read, Organization access (as needed) | Add `read:org` (classic) or equivalent fine-grained org permission |

## 2. Store the PAT in a local env file

Create a hidden env file in your home directory and lock down permissions:

```bash
echo "GITHUB_PERSONAL_ACCESS_TOKEN=github_pat_XXXXXXXXXXXXXXXXXXXXXXXX" > ~/.github-mcp.env
chmod 600 ~/.github-mcp.env
```

Sanity-check that Docker can read the token before touching Codex:

```bash
docker run --rm --env-file ~/.github-mcp.env alpine sh -lc 'test -n "$GITHUB_PERSONAL_ACCESS_TOKEN" && echo OK'
```

Docker reads this file locally and injects the environment variable when the container starts. The file never leaves your machine.

Optional guardrail so the token never lands in git history:

```bash
echo ".github-mcp.env" >> ~/.gitignore_global
git config --global core.excludesfile ~/.gitignore_global
```

## 3. Configure Codex to launch the MCP server

Edit `~/.codex/config.toml` and add the GitHub MCP server entry:

```toml
[mcp_servers.github]
command = "docker"
args = [
  "run","-i","--rm","--pull=always",
  "--name","github-mcp",
  "--env-file","/Users/YourMacUsername/.github-mcp.env",
  "ghcr.io/github/github-mcp-server:latest","stdio"
]
```

Replace `YourMacUsername` with your macOS username. `--pull=always` keeps the container fresh; if you pin a tag (recommended for teams), change the image reference to `ghcr.io/github/github-mcp-server:vX.Y.Z` and optionally drop `--pull=always`. The `--name github-mcp` flag ensures only one container runs at a time, and `--rm` auto-cleans when Codex exits.

### Optional: Narrow toolsets

The server exposes multiple toolsets. To keep Codex contexts tight, append `--toolsets` with a comma-separated list right before `stdio`:

```toml
args = [
  "run","-i","--rm","--pull=always",
  "--name","github-mcp",
  "--env-file","/Users/YourMacUsername/.github-mcp.env",
  "ghcr.io/github/github-mcp-server:latest",
  "--toolsets","repositories,issues","stdio"
]
```

Available values include `repositories`, `issues`, `pull_requests`, `workflows`, and more—check `github-mcp-server --help` for the current list.

### Optional: GitHub Enterprise

If you connect to GitHub Enterprise Server (GHES) or GitHub Enterprise Cloud with a custom hostname, add `GITHUB_HOST`:

```toml
args = [
  "run","-i","--rm","--pull=always",
  "--name","github-mcp",
  "--env-file","/Users/YourMacUsername/.github-mcp.env",
  "-e","GITHUB_HOST=https://your-org.ghe.com",
  "ghcr.io/github/github-mcp-server:latest","stdio"
]
```

## 4. Start Codex

Launch Colima and Codex—no need to export the token manually:

```bash
colima start        # ensure Docker is running
codex               # launch Codex; approve the docker run when prompted
```

The first time Codex connects, approve the docker command. You can select “Yes, and don’t ask again” once you trust it.

## 5. Verify in Codex

- Confirm the server is registered: in Codex, run `/mcp` (or view the MCP pane) and look for `github`.
- Basic info:
  ```
  Using only the `github` MCP tools, call `get_me`. Then call `search_repositories` with q="user:{login}" and per_page=100. Return a table: full_name | visibility | description.
  ```
- Read a repository file:
  ```
  Using the github MCP tools only, get the default branch for <owner>/<repo> and then fetch README.md via get_file_contents. Print the first 20 lines.
  ```
- Issues/PRs (requires write scopes):
  ```
  Create an issue titled "MCP test" in <owner>/<repo> with body "hello from Codex".
  ```

## 6. Day-to-day workflow

- Start Docker: `colima start`
- Launch Codex: `codex`
- Quit Codex with `Ctrl+C` (avoid `Ctrl+Z`). The container auto-removes because of `--rm`.
- Stop manually if needed: `docker stop github-mcp`
- Refresh the image periodically: `docker pull ghcr.io/github/github-mcp-server:latest` (or rely on `--pull=always`)

## Diagnose issues quickly

- If Codex reports it cannot start the server, re-run the token sanity check and confirm Colima is running.
- A missing `/mcp/github` entry usually means the docker command failed; inspect `~/.codex/logs/latest.log` for the stderr output.
- If you see `unrecognized flag` errors, update the container (`docker pull ...`) or remove optional flags like `--toolsets`.

## Troubleshooting

- **401 Bad credentials**  
  The container cannot read a valid token. Verify `~/.github-mcp.env` contains a single line `GITHUB_PERSONAL_ACCESS_TOKEN=...` with a valid token. Test outside Codex:
  ```bash
  docker run --rm --env-file ~/.github-mcp.env alpine sh -lc 'test -n "$GITHUB_PERSONAL_ACCESS_TOKEN" && echo OK'
  ```

- **0 repositories found**  
  The fine-grained PAT lacks repository access. Update the token permissions to include at least Metadata: Read and Contents: Read, and select the repositories (or “All repositories”).

- **Org repositories missing**  
  Add `read:org` (classic) or the equivalent org permissions for fine-grained PATs. Authorize the token for SAML SSO in your organization if required.

- **Multiple containers running**  
  Stop them all:
  ```bash
  docker ps -q --filter ancestor=ghcr.io/github/github-mcp-server | xargs -r docker stop
  ```
  The `--name github-mcp` flag prevents duplicates on future runs.

## Security notes

- The PAT lives only in `~/.github-mcp.env` on your machine.
- File permissions `600` restrict access to your user.
- The token never ends up in images, configs, or source control.
- Rotate the PAT periodically and update `~/.github-mcp.env` with the new value.

## Dockerless fallback (build from source)

If you cannot run Docker, clone the server and build it with Go 1.21+:

```bash
git clone https://github.com/github/github-mcp-server.git
cd github-mcp-server
go build ./cmd/github-mcp-server
./github-mcp-server stdio
```

Point Codex at the resulting binary (for example, replace the `docker` command with `/Users/you/github-mcp-server/github-mcp-server` in your config).

## Appendix: Quick reference

- Show hidden files (dotfiles):
  ```bash
  ls -la ~
  ```
- Edit the env file:
  ```bash
  nano ~/.github-mcp.env
  # GITHUB_PERSONAL_ACCESS_TOKEN=github_pat_...
  ```
- Confirm the token works (host side). Requires `jq`: `brew install jq`
  ```bash
  curl -s -H "Authorization: Bearer $(cut -d= -f2 ~/.github-mcp.env)" https://api.github.com/user | jq .login
  ```

PRs welcome for tweaks, GHES specifics, multi-org setups, or additional automation notes.
