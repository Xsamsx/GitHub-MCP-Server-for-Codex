# Configure GitHub MCP Server for Codex (macOS + Colima)

This guide explains how to connect OpenAI Codex to the GitHub MCP Server using Docker (via Colima) and a GitHub Personal Access Token (PAT) stored in a local `.env` file. You get a clean startup/shutdown flow, keep the token off your Codex config, and avoid Docker Desktop altogether.

## Prerequisites

- macOS with Homebrew
- [Colima](https://github.com/abiosoft/colima) (Docker runtime)
- OpenAI Codex installed (CLI or IDE extension)
- A GitHub account (personal or organization)

```bash
brew install colima
colima start           # starts the Docker daemon (inside a lightweight VM)
docker ps              # sanity check (should show no error)
```

## 1. Create a GitHub Personal Access Token (PAT)

Create a fine-grained PAT with the following minimum settings:

- Resource owner: your GitHub user (e.g., `Xsamsx`)
- Repository access: All repositories (or explicitly select the ones you need)
- Minimum repository permissions (adjust as needed):
  - Metadata: Read
  - Contents: Read (or Read & write if you plan to push)
  - Issues: Read (or Read & write)
  - Pull requests: Read (or Read & write)
- Optional: add Actions, Administration, Commit statuses for advanced automation

If your organization uses SAML SSO, authorize the token for that org after creation. You will receive a token that looks like `github_pat_XXXXXXXXXXXXXXXX...`.

## 2. Store the PAT in a local env file

Create a hidden env file in your home directory and lock down permissions:

```bash
echo "GITHUB_PERSONAL_ACCESS_TOKEN=github_pat_XXXXXXXXXXXXXXXXXXXXXXXX" > ~/.github-mcp.env
chmod 600 ~/.github-mcp.env
```

Docker will read this file locally and provide the environment variable to the container when it starts. The file never leaves your machine.

To ensure the file never gets committed accidentally:

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
  "run","-i","--rm",
  "--name","github-mcp",
  "--env-file","/Users/YourMacUsername/.github-mcp.env",
  "ghcr.io/github/github-mcp-server","stdio"
]
```

Replace `YourMacUsername` with your macOS username. The `--name github-mcp` flag ensures only one container runs at a time, and `--rm` auto-cleans when Codex exits.

### Optional: GitHub Enterprise

If you connect to GitHub Enterprise Server (GHES) or GitHub Enterprise Cloud with a custom hostname, add `GITHUB_HOST`:

```toml
args = [
  "run","-i","--rm",
  "--name","github-mcp",
  "--env-file","/Users/YourMacUsername/.github-mcp.env",
  "-e","GITHUB_HOST=https://your-org.ghe.com",
  "ghcr.io/github/github-mcp-server","stdio"
]
```

## 4. Start Codex

Launch Colima and Codex—no need to export the token manually:

```bash
colima start        # ensure Docker is running
codex               # launch Codex; approve the docker run when prompted
```

Codex will show the MCP server under `/mcp`. The command it runs should look like:

```bash
docker run -i --rm --name github-mcp --env-file /Users/You/.github-mcp.env ghcr.io/github/github-mcp-server stdio
```

Choose `Yes` (or “Yes, and don’t ask again”) when Codex asks whether to run the MCP server.

## 5. Verify in Codex

Use any of the following Codex prompts to confirm everything works:

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
- Quit Codex with `Ctrl+C` (avoid using `Ctrl+Z`). The container auto-removes because of `--rm`.
- To stop the container manually: `docker stop github-mcp`

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
