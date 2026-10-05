# Setup

How to get a fresh clone working with Claude Code and the GitHub MCP server.

`.mcp.json` is committed and reads the token from the `GITHUB_PAT_PERSONAL` environment variable. `direnv` loads that variable from a local `.envrc` file whenever you `cd` into the repo, and `gh` picks up the same token through `GH_TOKEN`. The setup has three parts:

1. **Create a token** (once per token lifetime).
2. **Install the tools** (once per machine, OS-specific).
3. **Configure the clone** (once per clone, the same on every OS).

## 1. Create a token

On GitHub, go to **Settings → Developer settings → Personal access tokens → Fine-grained tokens → Generate new token**.

- **Resource owner:** your personal account
- **Repository access:** Only select repositories → this repo
- **Expiration:** your choice; 90 days is a reasonable balance

Repository permissions:

| Permission    | Access         | Used for                                                     |
| ------------- | -------------- | ------------------------------------------------------------ |
| Contents      | Read and write | Branches, commits, file reads                                |
| Pull requests | Read and write | Opening, commenting on, and reviewing PRs                    |
| Issues        | Read and write | Creating, editing, and commenting on issues                  |
| Metadata      | Read-only      | Required (selected automatically)                            |
| Actions       | Read-only      | _Optional:_ checking CI runs                                 |
| Workflows     | Read and write | _Optional:_ only if Claude should edit `.github/workflows/`  |

Leave all account permissions off.

Fine-grained tokens only cover repos owned by the resource owner you select. If the repo belongs to someone else and you are a collaborator, use a classic token instead.

## 2. Install the tools

Each OS needs `direnv` and `gh`, plus a direnv hook in your shell config. Open a new terminal after adding the hook so it takes effect.

### macOS

```bash
brew install direnv gh
echo 'eval "$(direnv hook zsh)"' >> ~/.zshrc
```

zsh is the default shell on macOS. If you use bash, run `echo 'eval "$(direnv hook bash)"' >> ~/.bash_profile` instead.

### Linux

```bash
# Debian / Ubuntu
sudo apt install direnv gh

# Fedora
sudo dnf install direnv gh

# Arch
sudo pacman -S direnv github-cli
```

```bash
echo 'eval "$(direnv hook bash)"' >> ~/.bashrc
```

If you use zsh, run `echo 'eval "$(direnv hook zsh)"' >> ~/.zshrc` instead. On older Debian/Ubuntu releases where `apt` has no `gh` package, add GitHub's apt repository by following the instructions at [cli.github.com](https://cli.github.com).

### Windows

Use **Git Bash** (included with [Git for Windows](https://gitforwindows.org)). Claude Code also runs on it, and the rest of this guide works there unchanged. If you develop inside WSL, follow the Linux section instead.

From PowerShell or Command Prompt:

```powershell
winget install --id GitHub.cli
winget install --id direnv.direnv
```

Then in Git Bash:

```bash
echo 'eval "$(direnv hook bash)"' >> ~/.bashrc
```

## 3. Configure the clone

These steps are the same on every OS (on Windows, run them in Git Bash).

```bash
git clone <repo> && cd <repo>
cp .envrc.example .envrc
```

Open `.envrc`, replace `github_pat_xxx` with your token, then run:

```bash
direnv allow
```

`.envrc` is listed in `.gitignore`, so the token is never committed. `.envrc.example` is committed and contains only the placeholder.

## Verify

```bash
echo $GITHUB_PAT_PERSONAL | cut -c1-12   # token is loaded
gh auth status                           # shows your personal account
claude                                   # start Claude Code from inside the repo
```

Inside Claude Code, run `/mcp` and check that the `github` server is connected.

## Troubleshooting

| Symptom                              | Cause and fix                                                                                              |
| ------------------------------------ | ---------------------------------------------------------------------------------------------------------- |
| `GITHUB_PAT_PERSONAL` is empty       | The direnv hook isn't in your shell config, the terminal wasn't reopened, or `direnv allow` was skipped.     |
| `gh` shows a different account       | `GH_TOKEN` isn't set. Check `.envrc`.                                                                       |
| The MCP server returns 403           | The token is missing a permission. Compare it against the table in [Create a token](#1-create-a-token).    |
| The MCP server doesn't connect       | `claude` was started before direnv loaded the variables. Restart it from inside the repo.                 |
| Auth fails on Windows                | `.envrc` was saved with CRLF line endings, so the token ends in `\r`. Re-save it with LF line endings.      |
