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

## Quick Reference

| Task | Command |
|------|---------|
| Launch Kimi | `kimi` |
| Open browser UI | `kimi web` |
| Setup / login | `/login` (inside shell) |
| Get help | `/help` (inside shell) |
| Upgrade Kimi | `uv tool upgrade kimi-cli --no-cache` |
| Uninstall Kimi | `uv tool uninstall kimi-cli` |

---

**Requirements**: Python 3.12–3.14 (3.13 recommended).  
**Docs**: [moonshotai.github.io/kimi-cli](https://moonshotai.github.io/kimi-cli/)
