# OpenClaw Setup Guide: Sandboxed VM on macOS (M2 Pro)

A step-by-step guide to install OpenClaw in an isolated macOS VM so it **cannot touch your host files, folders, or local network**. Includes Telegram setup and AI model configuration (Claude / OpenRouter).

---

## Table of Contents

1. [What You Need Before Starting](#1-what-you-need-before-starting)
2. [Install Lume (The VM Manager)](#2-install-lume-the-vm-manager)
3. [Create the macOS VM](#3-create-the-macos-vm)
4. [Set Up the VM (First Boot)](#4-set-up-the-vm-first-boot)
5. [Network Isolation (Sandboxing)](#5-network-isolation-sandboxing)
6. [Install OpenClaw Inside the VM](#6-install-openclaw-inside-the-vm)
7. [Set Up Telegram](#7-set-up-telegram)
8. [Configure Your AI Model](#8-configure-your-ai-model)
9. [Lock Down Security](#9-lock-down-security)
10. [Run OpenClaw Headlessly (Background Mode)](#10-run-openclaw-headlessly-background-mode)
11. [Backups & Snapshots](#11-backups--snapshots)
12. [Troubleshooting](#12-troubleshooting)
13. [Tearing It All Down (Safe & Secure Deletion)](#13-tearing-it-all-down-safe--secure-deletion)

---

## 1. What You Need Before Starting

| Requirement | Details |
|---|---|
| **Mac** | Apple Silicon (M1/M2/M3/M4) — your M2 Pro is perfect |
| **macOS version** | Sequoia or later (on your host Mac) |
| **Free disk space** | ~60 GB for the VM |
| **Time** | ~20-30 minutes |
| **Telegram account** | You'll need one to create a bot |
| **AI API key** | An Anthropic (Claude), Google (Gemini), or OpenRouter API key |

### Where to get API keys

- **Anthropic (Claude):** Go to [console.anthropic.com](https://console.anthropic.com), create an account, and generate an API key. Keys start with `sk-ant-`. This is pay-per-use.
- **Google (Gemini):** Go to [aistudio.google.com/apikey](https://aistudio.google.com/apikey), sign in with your Google account, and create an API key. Gemini offers a generous free tier.
- **OpenRouter:** Go to [openrouter.ai](https://openrouter.ai), create an account, and generate an API key. Keys start with `sk-or-`. OpenRouter gives you access to 300+ models (including Claude) through one API key — handy if you want to experiment.

---

## 2. Install Lume (The VM Manager)

Lume is a lightweight tool that uses Apple's built-in virtualization to run VMs natively on Apple Silicon. Think of it like "VirtualBox but actually fast on M-series Macs."

Open **Terminal** on your host Mac (the real one, not a VM) and run:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/trycua/cua/main/libs/lume/scripts/install.sh)"
```

Then add it to your PATH so you can use the `lume` command from anywhere:

```bash
echo 'export PATH="$PATH:$HOME/.local/bin"' >> ~/.zshrc && source ~/.zshrc
```

Verify it worked:

```bash
lume --version
```

If you see a version number, you're good.

---

## 3. Create the macOS VM

This downloads a fresh copy of macOS and creates a virtual machine from it. It's a big download (~13 GB), so grab a coffee.

```bash
lume create openclaw --os macos --ipsw latest
```

A VNC window will pop up automatically showing the VM booting up. Don't close it — you need it for the next step.

---

## 4. Set Up the VM (First Boot)

In the VNC window that popped up, you'll see the macOS Setup Assistant (just like when you buy a new Mac). Walk through it:

1. **Language & Region** — pick whatever you want
2. **Apple ID** — **Skip this.** You don't need it for OpenClaw (unless you want iMessage integration later)
3. **Create a user account** — pick a username and password. **Write these down.** You'll need them to SSH in later
4. **All the optional stuff** (Siri, analytics, etc.) — skip everything

### Enable SSH (Remote Login)

Once you're at the desktop inside the VM:

1. Open **System Settings** → **General** → **Sharing**
2. Turn on **"Remote Login"** (this is SSH)

This lets you connect to the VM from your host Mac's Terminal without needing the VNC window.

### Get the VM's IP Address

Back on your **host Mac's Terminal** (not inside the VM), run:

```bash
lume get openclaw
```

Look for the IP address — it'll be something like `192.168.64.X`. Write it down.

### Test SSH Connection

```bash
ssh youruser@192.168.64.X
```

Replace `youruser` with the username you created, and `192.168.64.X` with the actual IP. Type `yes` when asked about the fingerprint, then enter your password.

If you get a command prompt inside the VM, SSH is working.

---

## 5. Network Isolation (Sandboxing)

This is the important part. By default, Lume uses **NAT networking**, which means:

- The VM **can** access the internet (needed for Telegram and AI APIs)
- The VM **cannot** be directly reached from other devices on your network
- But the VM **could theoretically** access other devices on your local network

To lock this down properly, you have **two layers of protection**:

### Layer 1: macOS Firewall Inside the VM

SSH into the VM and enable the firewall:

```bash
ssh youruser@192.168.64.X
```

Then inside the VM:

```bash
# Enable the macOS firewall
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setglobalstate on

# Explicitly allow SSH so you don't get locked out
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --add /usr/sbin/sshd
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --unblockapp /usr/sbin/sshd
```

> **Known issue:** Using `--setblockall on` will block SSH access to the VM, even if you explicitly allow `/usr/sbin/sshd`. This is a macOS application firewall quirk — it blocks system daemons launched via `launchctl` regardless of per-app exceptions. If you lose SSH access after enabling the firewall, open the VM via VNC (`lume run openclaw`) and disable the firewall with `sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setglobalstate off`. The host-side PF rules (Layer 2) and NAT isolation already prevent other devices on your network from reaching the VM, so the in-VM firewall is an optional extra layer, not a critical one.

### Layer 2: Block the VM from Accessing Your Local Network (pf firewall on Host)

This is the key step. On your **host Mac** (not the VM), you'll add a firewall rule that lets the VM reach the internet but **blocks it from reaching anything on your local network**.

First, find your local network range. Run this on your host Mac:

```bash
ifconfig | grep "inet " | grep -v 127.0.0.1
```

You'll see something like `192.168.1.X` or `10.0.0.X`. Your local network is likely `192.168.1.0/24` or `10.0.0.0/24`.

Now create a pf (packet filter) anchor file:

```bash
sudo nano /etc/pf.anchors/openclaw-isolate
```

Paste this (adjust the network ranges to match yours):

```
# Block VM from reaching the host's local network
# Replace 192.168.1.0/24 with YOUR actual local network range
# The VM subnet (192.168.64.0/24) is Lume's NAT range — keep it allowed for SSH

# Allow VM to reach the internet and its own NAT subnet
pass out on vmnet0 from 192.168.64.0/24 to 192.168.64.0/24
pass out on vmnet0 from 192.168.64.0/24 to ! 192.168.1.0/24

# Block VM from reaching your local LAN
block out on vmnet0 from 192.168.64.0/24 to 192.168.1.0/24
```

Save (`Ctrl+O`, Enter, `Ctrl+X`).

Now add this anchor to the main pf config:

```bash
sudo nano /etc/pf.conf
```

Add these two lines **after** all existing rules (at the end of the file):

```
anchor "openclaw-isolate"
load anchor "openclaw-isolate" from "/etc/pf.anchors/openclaw-isolate"
```

> **Important:** These lines must go **after** the `scrub-anchor`, `nat-anchor`, `rdr-anchor`, and `dummynet-anchor` lines. PF requires rules in a specific order: options → normalization (`scrub`) → translation (`nat`/`rdr`) → filtering (`anchor`/`block`/`pass`). Putting filter rules before normalization rules will cause a syntax error.

Enable pf:

```bash
sudo pfctl -ef /etc/pf.conf
```

> **What did this do?** The VM can still reach the internet (for Telegram and AI APIs), and you can still SSH into it, but it **cannot** see or access any other device on your home network — your files, NAS drives, other computers, printers, etc. are all invisible to it.

### Layer 3: OpenClaw's Built-in Sandbox Mode

We'll configure this in [Step 9](#9-lock-down-security) after OpenClaw is installed.

---

## 6. Install OpenClaw Inside the VM

SSH into the VM:

```bash
ssh youruser@192.168.64.X
```

Install Node.js (OpenClaw needs it):

```bash
# Install Homebrew first (the macOS package manager)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Add Homebrew to PATH
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zshrc && source ~/.zshrc

# Install Node.js
brew install node
```

Now install OpenClaw:

```bash
npm install -g openclaw@latest
```

Run the onboarding wizard:

```bash
openclaw onboard
```

This will ask you questions about your setup. For now, you can pick defaults — we'll configure the AI model and Telegram in the next steps.

---

## 7. Set Up Telegram

### Step 7a: Create a Telegram Bot

1. Open Telegram on your phone or desktop
2. Search for **@BotFather** (look for the blue checkmark — this is Telegram's official bot creator)
3. Send `/newbot` to BotFather
4. BotFather will ask you for:
   - A **name** for your bot (e.g., "My OpenClaw Assistant")
   - A **username** for your bot (must end in `bot`, e.g., `myclaw_bot`)
5. BotFather will give you a **bot token** — it looks like `123456789:ABCdef-something-long`. **Copy this and save it somewhere safe.** This is basically the password to your bot.

### Step 7b: Configure Telegram in OpenClaw

SSH into the VM and edit the config file:

```bash
nano ~/.openclaw/openclaw.json
```

> **Note:** The config file may use JSON5 format (allows comments and trailing commas). If the file already exists with content, merge these settings in.

Add or update the `channels` section:

```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "YOUR_BOT_TOKEN_FROM_BOTFATHER",
      dmPolicy: "pairing",
      groups: {
        "*": {
          requireMention: true,
        },
      },
    },
  },
}
```

**What do these settings mean?**

| Setting | What It Does |
|---|---|
| `enabled: true` | Turns on the Telegram channel |
| `botToken` | The token BotFather gave you |
| `dmPolicy: "pairing"` | When someone messages your bot, they need a pairing code that YOU approve. This prevents strangers from using your bot |
| `requireMention: true` | In group chats, the bot only responds when you @mention it |

### Step 7c: Start OpenClaw and Pair Your Account

Start the OpenClaw gateway:

```bash
openclaw gateway
```

Now on your phone/desktop, open Telegram and **send any message to your bot** (search for the username you chose, e.g., `@myclaw_bot`).

The bot will give you a **pairing code**. Back in the VM terminal, approve it:

```bash
# List pending pairing requests
openclaw pairing list telegram

# Approve the code (replace CODE with the actual code shown)
openclaw pairing approve telegram CODE
```

Once approved, you're paired. The bot will now respond to your messages.

### Step 7d: Find Your Telegram User ID (For Extra Security)

If you want to lock down the bot to ONLY your account (instead of the pairing flow), you need your numeric Telegram user ID:

1. Send a message to your bot
2. In the VM, run: `openclaw logs --follow`
3. Look for `from.id` in the log output — that's your numeric user ID

Then update the config to use an allowlist:

```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "YOUR_BOT_TOKEN",
      dmPolicy: "allowlist",
      allowFrom: [YOUR_NUMERIC_ID],
    },
  },
}
```

This means **only you** can talk to the bot. Period.

---

## 8. Configure Your AI Model

You have three options. Pick **one**.

### Option A: Claude (Direct from Anthropic)

This connects directly to Anthropic's API. Best if you only want Claude models.

**Via the onboarding wizard:**

```bash
openclaw onboard --anthropic-api-key "sk-ant-YOUR_KEY_HERE"
```

**Or by editing the config file manually:**

```bash
nano ~/.openclaw/openclaw.json
```

Add/update:

```json5
{
  env: {
    ANTHROPIC_API_KEY: "sk-ant-YOUR_KEY_HERE",
  },
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-sonnet-4-5",
      },
    },
  },
}
```

**Available Claude models (from most to least capable/expensive):**

| Model | Best For |
|---|---|
| `anthropic/claude-opus-4-6` | Hardest tasks, most capable, most expensive |
| `anthropic/claude-sonnet-4-6` | Great balance of capability and cost |
| `anthropic/claude-sonnet-4-5` | Good default for daily use |
| `anthropic/claude-haiku-4-5` | Fast and cheap for simple tasks |

### Option B: Gemini (Direct from Google)

This connects directly to Google's Gemini API. Gemini offers a generous free tier, making it a great option if you want capable models without paying.

**Via the onboarding wizard:**

```bash
openclaw onboard --auth-choice gemini-api-key
```

**Or by editing the config file manually:**

```bash
nano ~/.openclaw/openclaw.json
```

Add/update:

```json5
{
  env: {
    GEMINI_API_KEY: "YOUR_GEMINI_API_KEY_HERE",
  },
  agents: {
    defaults: {
      model: {
        primary: "google/gemini-2.5-flash",
      },
    },
  },
}
```

**Available Gemini models:**

| Model | Best For |
|---|---|
| `google/gemini-2.5-pro` | Most capable, complex reasoning |
| `google/gemini-2.5-flash` | Good balance of speed and capability (recommended) |
| `google/gemini-2.0-flash` | Fast and cheap for simple tasks |

> **Why Gemini?** Google's free tier gives you significantly more requests than OpenRouter's free models, with better tool-calling support. If you want a reliable free option, this is the best choice.

### Option C: OpenRouter (Access to 300+ Models)

OpenRouter acts as a "middleman" that gives you access to Claude, GPT, Gemini, Llama, and hundreds of other models through one API key.

**Via the onboarding wizard:**

```bash
openclaw onboard --auth-choice apiKey --token-provider openrouter --token "sk-or-YOUR_KEY_HERE"
```

**Or by editing the config file manually:**

```json5
{
  env: {
    OPENROUTER_API_KEY: "sk-or-YOUR_KEY_HERE",
  },
  agents: {
    defaults: {
      model: {
        primary: "openrouter/anthropic/claude-sonnet-4-5",
      },
    },
  },
}
```

**Note the model format difference:** With OpenRouter, you prefix models with `openrouter/`, so Claude becomes `openrouter/anthropic/claude-sonnet-4-5`.

**You can also set up fallback models** (if the primary is down, try the next one):

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "openrouter/anthropic/claude-sonnet-4-5",
        fallback: [
          "openrouter/google/gemini-2.5-pro",
          "openrouter/openai/gpt-4o",
        ],
      },
    },
  },
}
```

#### Using Free Models Only (Zero Cost)

OpenRouter offers dozens of models that are completely free to use — no credit card required. If you want to try OpenClaw without spending anything, this is the way.

**The easiest way** — use `openrouter/free`, a special "meta-model" that automatically picks the best free model for each request:

```json5
{
  env: {
    OPENROUTER_API_KEY: "sk-or-YOUR_KEY_HERE",
  },
  agents: {
    defaults: {
      model: {
        primary: "openrouter/free",
      },
    },
  },
}
```

**Or pick specific free models yourself** by appending `:free` to the model slug:

```json5
{
  env: {
    OPENROUTER_API_KEY: "sk-or-YOUR_KEY_HERE",
  },
  agents: {
    defaults: {
      model: {
        primary: "openrouter/google/gemini-2.5-flash:free",
        fallback: [
          "openrouter/deepseek/deepseek-v3.2-20251201:free",
          "openrouter/free",
        ],
      },
    },
  },
}
```

**Notable free models available right now:**

| Model | Good For |
|---|---|
| `google/gemini-2.5-flash` | General purpose, fast |
| `google/gemini-3-flash-preview` | Latest Google model |
| `deepseek/deepseek-v3.2` | Roleplay, creative |
| `minimax/minimax-m2.1` | Coding |
| `arcee-ai/trinity-large-preview` | Creative writing, agents (400B parameters) |

Browse the full list at [openrouter.ai/collections/free-models](https://openrouter.ai/collections/free-models).

**Important things to know about free models:**

| Caveat | Details |
|---|---|
| **Rate limits** | 20 requests/min, 200 requests/day. Bumped to 1,000/day if you buy at least 10 credits ($10) on OpenRouter |
| **No Claude** | Claude models are NOT available for free. If you want Claude, you need to pay (direct or via OpenRouter) |
| **Less reliable** | Free models are more likely to hallucinate, skip steps, or claim they did something they didn't |
| **Extra guardrails recommended** | Consider adding a two-phase "propose then execute" workflow so the AI shows you its plan before doing anything |

> **Bottom line:** Free models are great for trying out OpenClaw and casual use. For serious/reliable daily use, a paid model (even a cheap one like Claude Haiku) will give you much better results.

### Which Should You Pick?

| | Claude Direct | Gemini Direct | OpenRouter (Paid) | OpenRouter (Free) |
|---|---|---|---|---|
| **Cost** | Pay per use | Free tier + pay per use | Pay per use (tiny markup) | Free |
| **Simplicity** | Simpler — one provider, one key | One provider, one key | One key for everything | One key, no billing |
| **Model variety** | Claude models only | Gemini models only | 300+ models from all providers | ~25-30 free models (no Claude) |
| **Tool calling** | Excellent | Good | Varies by model | Poor — most free models can't use tools |
| **Reliability** | Direct connection | Direct connection | Extra failover across providers | Lower — weaker models, rate limits |
| **Best for** | Serious use, best tool support | Free/cheap with good capability | Flexibility and experimentation | Trying out OpenClaw only |

**My recommendation:** If you want the most reliable experience, go Claude direct. If you want a capable free option that actually supports tool calling (like editing files), go Gemini direct. If you want to experiment with different models, go OpenRouter paid. Avoid OpenRouter free for anything beyond basic chat — the models can't reliably use tools.

---

## 9. Lock Down Security

Now that OpenClaw is installed, let's harden it. Edit the config:

```bash
nano ~/.openclaw/openclaw.json
```

Add or merge these security settings:

```json5
{
  gateway: {
    mode: "local",
    bind: "loopback",
    auth: {
      mode: "token",
      token: "REPLACE-WITH-A-LONG-RANDOM-STRING",
    },
  },
  session: {
    dmScope: "per-channel-peer",
  },
  tools: {
    profile: "messaging",
    allow: ["group:fs"],
    deny: ["group:automation", "group:runtime"],
    exec: {
      security: "deny",
      ask: "always",
    },
    elevated: {
      enabled: false,
    },
  },
}
```

> **Note:** `group:fs` is explicitly allowed so the bot can read and update its own files (e.g., SOUL.md). This is independent of `exec.security`, which only controls shell command execution. Since OpenClaw is already running inside an isolated VM, filesystem access is scoped to the VM only — it cannot touch your host Mac's files.

**What each setting does:**

| Setting | What It Does |
|---|---|
| `mode: "local"` | Gateway only accessible locally (not from the internet) |
| `bind: "loopback"` | Only listens on 127.0.0.1 — no external access |
| `auth.mode: "token"` | Requires a token to access the gateway API |
| `dmScope: "per-channel-peer"` | Each person gets their own isolated conversation context |
| `tools.profile: "messaging"` | Only enables messaging-related tools |
| `allow: ["group:fs"]` | Adds filesystem tools (read, write, edit) on top of the messaging profile |
| `deny: [...]` | Blocks dangerous tool groups (automation, runtime) |
| `exec.security: "deny"` | Blocks shell command execution (does not affect filesystem tools) |
| `elevated.enabled: false` | No elevated/admin tool access |

Generate a random token for the auth setting:

```bash
openssl rand -hex 32
```

Copy the output and paste it as the `token` value.

### Run the Security Audit

OpenClaw has a built-in security checker:

```bash
openclaw security audit
```

This will flag any issues with your configuration. Fix anything it complains about.

---

## 10. Run OpenClaw Headlessly (Background Mode)

Once everything is configured and working, you don't need the VNC window anymore. Stop the VM and restart it without a display:

```bash
# From your HOST Mac (not the VM)
lume stop openclaw
lume run openclaw --no-display
```

Your OpenClaw instance is now running in the background. You interact with it entirely through Telegram.

To check if it's running:

```bash
ssh youruser@192.168.64.X "openclaw status"
```

### Keep It Running 24/7

On your **host Mac**:

- **System Settings → Energy Saver** — set "Prevent automatic sleeping" and keep it plugged in
- The VM will survive sleep/wake cycles, but for true 24/7 operation, prevent sleep entirely

---

## 11. Backups & Snapshots

Save a "golden" snapshot of your fully configured VM so you can reset to a known-good state:

```bash
# From your HOST Mac
lume stop openclaw
lume clone openclaw openclaw-golden
```

If things go wrong, reset to the snapshot:

```bash
lume stop openclaw && lume delete openclaw
lume clone openclaw-golden openclaw
lume run openclaw --no-display
```

---

## 12. Troubleshooting

| Problem | Fix |
|---|---|
| `lume: command not found` | Run: `echo 'export PATH="$PATH:$HOME/.local/bin"' >> ~/.zshrc && source ~/.zshrc` |
| SSH connection refused | Make sure "Remote Login" is enabled in VM's System Settings → General → Sharing |
| Can't find VM IP | Wait for full boot, then run `lume get openclaw` again |
| Telegram bot not responding | Check the bot token is correct. Run `openclaw logs --follow` to see errors |
| Bot ignores group messages | Disable Privacy Mode: message @BotFather → `/setprivacy` → Disable. Then remove and re-add the bot to the group |
| `openclaw: command not found` (in VM) | Run: `npm install -g openclaw@latest` again, or check that Node is installed with `node --version` |
| High API costs | Switch to a cheaper model like `claude-haiku-4-5` or use `openrouter/openrouter/auto` for automatic cost optimization |

---

## Quick Reference: Your Complete Config File

Here's what a fully configured `~/.openclaw/openclaw.json` looks like with all the pieces together:

```json5
{
  // AI Model (pick ONE of these env blocks)
  env: {
    // For Claude direct:
    ANTHROPIC_API_KEY: "sk-ant-YOUR_KEY",
    // OR for Gemini direct:
    // GEMINI_API_KEY: "YOUR_GEMINI_KEY",
    // OR for OpenRouter:
    // OPENROUTER_API_KEY: "sk-or-YOUR_KEY",
  },

  // Model selection
  agents: {
    defaults: {
      model: {
        // For Claude direct:
        primary: "anthropic/claude-sonnet-4-5",
        // OR for Gemini direct:
        // primary: "google/gemini-2.5-flash",
        // OR for OpenRouter:
        // primary: "openrouter/anthropic/claude-sonnet-4-5",
      },
    },
  },

  // Telegram
  channels: {
    telegram: {
      enabled: true,
      botToken: "123456789:ABCdef-your-token",
      dmPolicy: "pairing",
      groups: {
        "*": { requireMention: true },
      },
    },
  },

  // Security
  gateway: {
    mode: "local",
    bind: "loopback",
    auth: {
      mode: "token",
      token: "your-long-random-token-here",
    },
  },
  session: {
    dmScope: "per-channel-peer",
  },
  tools: {
    profile: "messaging",
    allow: ["group:fs"],
    deny: ["group:automation", "group:runtime"],
    exec: { security: "deny", ask: "always" },
    elevated: { enabled: false },
  },
}
```

---

## 13. Tearing It All Down (Safe & Secure Deletion)

When you're done with OpenClaw and want to completely remove it — VM, credentials, firewall rules, and all — follow these steps in order. This ensures nothing is left behind on your host Mac.

### Step 13a: Revoke Your API Keys

Before deleting anything, revoke the API keys so they can't be used even if someone somehow recovered the VM data later.

- **Anthropic:** Go to [console.anthropic.com](https://console.anthropic.com) → API Keys → delete the key you used
- **OpenRouter:** Go to [openrouter.ai/settings/keys](https://openrouter.ai/settings/keys) → delete the key you used

### Step 13b: Delete Your Telegram Bot

Open Telegram, go to **@BotFather**, and send:

```
/deletebot
```

Select your bot from the list. This permanently destroys the bot and invalidates the token. Nobody can message it or use the token after this.

### Step 13c: Stop and Delete the VM

Run these commands on your **host Mac**:

```bash
# Stop the VM if it's running
lume stop openclaw

# Delete the VM and all its data
lume delete openclaw
```

If you also created a golden snapshot in Step 11, delete that too:

```bash
lume delete openclaw-golden
```

**What this does:** Lume removes the entire virtual disk image (~60 GB). Everything inside the VM — OpenClaw, its config, session transcripts, credentials, chat history — is gone.

### Step 13d: Verify the VM Files Are Gone

Lume stores VMs in `~/.lume/` by default. Double-check nothing is left:

```bash
ls ~/.lume/vms/
```

If you see an `openclaw` or `openclaw-golden` folder still there, remove it manually:

```bash
rm -rf ~/.lume/vms/openclaw
rm -rf ~/.lume/vms/openclaw-golden
```

### Step 13e: Remove the Firewall Rules

In Step 5, you added `pf` firewall rules on your host Mac to isolate the VM. Now that the VM is gone, clean those up.

Remove the anchor file:

```bash
sudo rm /etc/pf.anchors/openclaw-isolate
```

Edit the main pf config to remove the lines you added:

```bash
sudo nano /etc/pf.conf
```

Delete these two lines (they were added in Step 5):

```
anchor "openclaw-isolate"
load anchor "openclaw-isolate" from "/etc/pf.anchors/openclaw-isolate"
```

Save the file, then reload the firewall:

```bash
sudo pfctl -f /etc/pf.conf
```

### Step 13f: Uninstall Lume (Optional)

If you no longer need Lume (the VM manager), remove it:

```bash
rm -rf ~/.local/bin/lume
rm -rf ~/.lume
```

Then remove the PATH line from your shell config:

```bash
nano ~/.zshrc
```

Find and delete this line:

```
export PATH="$PATH:$HOME/.local/bin"
```

Save, then reload:

```bash
source ~/.zshrc
```

### Step 13g: Clear Your SSH Known Hosts Entry

When you first SSH'd into the VM, your Mac saved its fingerprint. Clean that up so it doesn't cause conflicts if you ever reuse that IP:

```bash
ssh-keygen -R 192.168.64.X
```

Replace `192.168.64.X` with the VM IP you used.

### Deletion Checklist

Run through this to make sure you got everything:

| Item | How to Verify |
|---|---|
| API keys revoked | Check Anthropic Console / OpenRouter dashboard — key should be gone |
| Telegram bot deleted | Search for your bot in Telegram — it should no longer exist |
| VM deleted | `lume list` shows no `openclaw` entries |
| VM files gone | `ls ~/.lume/vms/` shows no `openclaw` folders |
| Firewall rules removed | `sudo pfctl -sr` output should not mention `openclaw-isolate` |
| Lume uninstalled (optional) | `which lume` returns nothing |
| SSH known hosts cleaned | `grep 192.168.64.X ~/.ssh/known_hosts` returns nothing |

Once all boxes are checked, your Mac is completely clean — as if OpenClaw was never there.

---

## Security Summary: What's Protecting You

Your setup has **four layers** of isolation:

1. **VM Isolation** — OpenClaw runs in a separate macOS virtual machine. It has its own filesystem, processes, and memory. It literally cannot see your host Mac's files.

2. **Network Firewall (pf rules on host)** — The VM can reach the internet (for Telegram + AI APIs) but is **blocked from accessing your local network** (no access to your NAS, printers, other computers, etc.).

3. **VM Firewall** — The macOS firewall inside the VM blocks all incoming connections.

4. **OpenClaw Security Config** — The gateway is bound to loopback only, dangerous tools are disabled, and Telegram access is restricted to your account via pairing or allowlist.

---

## Sources

- [OpenClaw macOS VM Install Guide](https://docs.openclaw.ai/install/macos-vm)
- [Claw School Guide (Every.to)](https://every.to/guides/claw-school)
- [OpenClaw Telegram Channel Docs](https://docs.openclaw.ai/channels/telegram)
- [OpenClaw Anthropic Provider Docs](https://docs.openclaw.ai/providers/anthropic)
- [OpenClaw OpenRouter Provider Docs](https://docs.openclaw.ai/providers/openrouter)
- [OpenClaw Security Docs](https://docs.openclaw.ai/gateway/security)
- [OpenClaw GitHub](https://github.com/openclaw/openclaw)
- [Lume VM Manager](https://github.com/SpaceBlocks/lume-mac)
- [macOS pf Firewall Guide](https://blog.neilsabol.site/post/quickly-easily-adding-pf-packet-filter-firewall-rules-macos-osx/)
