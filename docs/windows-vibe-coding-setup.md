# Windows Vibe-Coding Setup Guide

A practical guide to installing the latest PowerShell and configuring Kimi Code CLI for vibe-coding on Windows.

---

## 1. Install the Latest PowerShell

Windows ships with **Windows PowerShell 5.1** by default. You want **PowerShell 7** (the modern, cross-platform version) for better performance and compatibility.

### Install via winget (recommended)

Open a terminal (Windows Terminal, CMD, or the existing PowerShell) and run:

```powershell
winget install Microsoft.PowerShell
```

### Alternative methods

- **Microsoft Store**: Search for "PowerShell" and install the official app by Microsoft.
- **GitHub Releases**: Download the `.msi` installer from [github.com/PowerShell/PowerShell/releases](https://github.com/PowerShell/PowerShell/releases).

### Verify

```powershell
pwsh --version
```

> **Tip**: Set PowerShell 7 as your default shell in Windows Terminal for the best experience.

---

## 2. Install Kimi Code CLI

Kimi Code CLI is the terminal agent that enables "vibe coding" — you describe what you want in natural language, and it reads, edits, and executes code for you.

### One-line installer (run in PowerShell 7)

```powershell
Invoke-RestMethod https://code.kimi.com/install.ps1 | Invoke-Expression
```

This script automatically installs `uv` (a fast Python package manager) and then installs the `kimi` CLI tool.

### If you already have `uv` installed

```powershell
uv tool install --python 3.13 kimi-cli
```

### Verify

```powershell
kimi --version
```

### Upgrade later

```powershell
uv tool upgrade kimi-cli --no-cache
```

---

## 3. Register an Account & Configure Kimi

### Where to register

Go to **[kimi.com](https://kimi.com)** (or **[kimi.moonshot.cn](https://kimi.moonshot.cn)** for China region) and sign up for a Moonshot AI account. You can register with email, phone, or OAuth (Google, etc.).

### Get your API key

Once logged in:

1. Navigate to your **Developer / API settings**.
2. Create a new API key.
3. Copy the key (starts with `sk-...`).

### Configure Kimi CLI

Navigate to your project folder and launch Kimi:

```powershell
cd D:\YourProject
kimi
```

On first run, type the setup command inside Kimi:

```
/login
```

Follow the wizard:

1. **Select platform**: Choose **Kimi Code** (recommended — supports search and web fetching) or **Moonshot AI Open Platform**.
2. **Enter your API key**: Paste the key you copied.
3. **Select a model**: Pick from the available list (e.g., `kimi-k2`).

Your settings are saved to `~/.kimi/config.toml` and reloaded automatically.

### Start vibe coding

Now just describe what you want:

```
Create a React component that fetches and displays a user profile
```

Or let Kimi analyze your project first:

```
/init
```

---

## 4. Install Git for GitLab

GitLab uses standard Git for version control. You need Git for Windows to clone repositories, commit changes, and push code.

### Install Git

```powershell
winget install Git.Git
```

Or download the installer from [git-scm.com/download/win](https://git-scm.com/download/win).

### Verify

```powershell
git --version
```

### Configure your identity

```powershell
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
```

### Clone a GitLab repository

```powershell
git clone https://gitlab.com/your-group/your-project.git
cd your-project
```

> **Tip**: If your GitLab instance uses SSH, set up an SSH key first:
> ```powershell
> ssh-keygen -t ed25519 -C "your.email@example.com"
> ```
> Then add the public key (`~/.ssh/id_ed25519.pub`) to your GitLab profile under **Preferences → SSH Keys**.

### Share files via GitLab

After making changes in your project:

```powershell
# Stage your changes
git add .

# Commit with a message
git commit -m "Describe your changes"

# Push to the remote repository
git push origin main
```

For larger changes, use **Merge Requests** (MRs):

1. Create a feature branch:
   ```powershell
   git checkout -b feature/my-new-feature
   ```
2. Make changes, commit, and push the branch:
   ```powershell
   git push -u origin feature/my-new-feature
   ```
3. Open your GitLab project in a browser and click **Create merge request**.
4. Fill in the title and description, then submit for review.

---

## 5. Install and Start BMAD-Org

BMAD (Build-Measure-Analyze-Deploy) is an AI agent framework that adds structured skills and workflows to your project. It is installed per project via `npx`.

### Prerequisites

- **Node.js** (LTS recommended). Install via winget:
  ```powershell
  winget install OpenJS.NodeJS
  ```

### Install BMAD in your project

Navigate to your project root and run:

```powershell
cd D:\YourProject
npx bmad-method install --modules bmm --tools kimi-code --yes
```

Common `--tools` values for different IDEs:

| IDE / Agent | Tool ID |
|-------------|---------|
| Kimi Code | `kimi-code` |
| Claude Code | `claude-code` |
| Cursor | `cursor` |
| GitHub Copilot | `github-copilot` |

To install multiple tools at once:

```powershell
npx bmad-method install --modules bmm --tools "claude-code,codex,kimi-code" --yes
```

### What gets created

The installer creates a `_bmad/` folder in your project containing:

- `config.toml` – team-wide agent and module configuration
- `config.user.toml` – your personal preferences (name, skill level)
- `core/`, `bmm/`, `custom/` – modules and skills

### Check BMAD status

```powershell
npx bmad-method status
```

### Update BMAD later

```powershell
npx bmad-method install --action update --yes
```

### Uninstall BMAD from a project

```powershell
npx bmad-method uninstall
```

---

## 6. MCP Servers Setup

> **Important**: MCP servers are **installed per machine**, not via the repository. The repo may contain a configuration file (e.g., `~/.kimi/mcp.json`) that tells Kimi how to connect to each server, but the actual server binaries or packages must be present on every developer's workstation.

The following MCP servers are commonly used with this stack. Install only the ones you need.

### GCP (Google Cloud Platform)

Install the Google Cloud MCP server globally:

```powershell
npm install -g @google-cloud/gcloud-mcp
```

Verify gcloud is also installed (the MCP server wraps it):

```powershell
winget install Google.CloudSDK
```

Authenticate once:

```powershell
gcloud auth login
```

### PostgreSQL

Install the PostgreSQL MCP server globally:

```powershell
npm install -g @modelcontextprotocol/server-postgres
```

Or use the community alternative (used in some configurations):

```powershell
npm install -g @henkey/postgres-mcp-server
```

> **Note**: You still need a running PostgreSQL instance (local or remote) and a valid connection string. The MCP server only provides the bridge; it does not install PostgreSQL itself.

### HashiCorp Vault

The Vault MCP server is typically a standalone binary (not available on npm). You need to:

1. **Download the binary** from your organization's internal source or build it from the upstream Vault MCP server repository.
2. **Place it in a folder on your PATH**, for example:
   ```powershell
   C:\Users\<you>\bin\vault-mcp-server.exe
   ```
3. Ensure the environment variables `VAULT_ADDR` and `VAULT_TOKEN` are set (either in your shell profile or in the MCP config).

### Playwright

Install the official Playwright MCP server globally:

```powershell
npm install -g @playwright/mcp
```

Playwright browsers will be downloaded automatically on first use. If you need to install them manually:

```powershell
npx playwright install
```

### Configuring MCP servers in Kimi

After installation, edit Kimi's MCP config file at `~/.kimi/mcp.json` to point to the installed servers. Here is an example structure:

```json
{
  "mcpServers": {
    "gcp": {
      "command": "cmd",
      "args": ["/c", "npx", "-y", "@google-cloud/gcloud-mcp"],
      "env": {}
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "POSTGRES_CONNECTION_STRING": "postgresql://user:pass@host:port/db"
      }
    },
    "vault": {
      "command": "C:\\Users\\<you>\\bin\\vault-mcp-server.exe",
      "args": ["stdio"],
      "env": {
        "VAULT_ADDR": "https://your-vault-instance.com",
        "VAULT_TOKEN": "your-token"
      }
    },
    "playwright": {
      "command": "npx",
      "args": ["-y", "@playwright/mcp"],
      "env": {}
    }
  }
}
```

> **Security tip**: Keep tokens and connection strings out of committed files. Use environment variables or a local, gitignored config file where possible.

---

## Quick Reference

| Task | Command |
|------|---------|
| Launch Kimi | `kimi` |
| Open browser UI | `kimi web` |
| Setup / login | `/login` (inside shell) |
| Get help | `/help` (inside shell) |
| Upgrade Kimi | `uv tool upgrade kimi-cli --no-cache` |
| Uninstall Kimi | `uv tool uninstall kimi-cli` |
| Check BMAD status | `npx bmad-method status` |
| Update BMAD | `npx bmad-method install --action update --yes` |

---

**Requirements**: Python 3.12–3.14 (3.13 recommended), Node.js LTS.  
**Docs**: [moonshotai.github.io/kimi-cli](https://moonshotai.github.io/kimi-cli/)
