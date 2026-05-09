# Warp OSS: AI with Bring-Your-Own Keys (No Login Required)

Warp OSS supports AI features — Agent Mode, inline AI, command suggestions — without
any Warp.dev account. You supply your own API keys for the LLM provider of your choice
and route requests through a local proxy.

## Overview

```
WarpOss.app
  └─ AI request → WARP_OSS_LLM_PROXY_URL (http://127.0.0.1:4000)
                     └─ liter-llm proxy
                          ├─ reads API keys from liter-llm-proxy.toml
                          └─ routes to: Anthropic / OpenAI / Google / OpenRouter
```

`liter-llm` is a lightweight Rust-native proxy (~35 MB) that speaks the OpenAI wire
format and forwards requests to whichever provider you configure. Warp OSS supports
this path via the `WARP_OSS_LLM_PROXY_URL` environment variable.

## Step 1 — Install liter-llm

```bash
brew install kreuzberg-dev/tap/liter-llm
```

Verify:

```bash
liter-llm --version
```

## Step 2 — Create a config file

Create `~/.config/warp-oss/liter-llm-proxy.toml`. API keys are read from environment
variables at startup — never stored in the config file.

### Anthropic only

```toml
[[models]]
name = "default"
provider_model = "anthropic/claude-opus-4-7"
api_key = "${ANTHROPIC_API_KEY}"
```

### OpenAI only

```toml
[[models]]
name = "default"
provider_model = "openai/gpt-4o"
api_key = "${OPENAI_API_KEY}"
```

### Multiple providers (Anthropic + OpenAI + Google + OpenRouter)

```toml
[[models]]
name = "claude"
provider_model = "anthropic/claude-opus-4-7"
api_key = "${ANTHROPIC_API_KEY}"

[[models]]
name = "gpt-4o"
provider_model = "openai/gpt-4o"
api_key = "${OPENAI_API_KEY}"

[[models]]
name = "gemini"
provider_model = "google_ai/gemini-2.0-flash"
api_key = "${GOOGLE_API_KEY}"

[[models]]
name = "openrouter"
provider_model = "openrouter/openai/gpt-4o"
api_key = "${OPENROUTER_API_KEY}"
```

OpenRouter is useful if you want access to many models through a single key. See
[openrouter.ai/models](https://openrouter.ai/models) for available model IDs.

### Fallback chain (recommended for reliability)

```toml
[[models]]
name = "default"
provider_model = "anthropic/claude-opus-4-7"
api_key = "${ANTHROPIC_API_KEY}"
fallbacks = ["gpt4o-fallback"]

[[models]]
name = "gpt4o-fallback"
provider_model = "openai/gpt-4o"
api_key = "${OPENAI_API_KEY}"
```

## Step 3 — Start the proxy

```bash
mkdir -p ~/.config/warp-oss

ANTHROPIC_API_KEY=sk-ant-... \
liter-llm api --config ~/.config/warp-oss/liter-llm-proxy.toml
```

The proxy listens on `http://127.0.0.1:4000` by default. You can change the port:

```toml
[server]
port = 9000
```

## Step 4 — Launch Warp OSS

Point Warp at the local proxy:

```bash
WARP_OSS_LLM_PROXY_URL=http://127.0.0.1:4000 open /Applications/WarpOss.app
```

Or set it in your shell profile so it applies to every launch:

```bash
# ~/.zshrc or ~/.bashrc
export WARP_OSS_LLM_PROXY_URL=http://127.0.0.1:4000
```

## Step 5 — Enter your API key in Warp settings

1. Open **Settings → AI**
2. Enter your API key in the appropriate provider field
3. The key is stored in the macOS keychain — you only need to enter it once

> **Note:** The key in Warp Settings and the key passed to liter-llm must match.
> Warp uses the Settings key for its internal BYOK feature gate; liter-llm uses the
> env-var key for the actual API call. Keep them in sync.

## Verifying it works

Send an AI prompt in a terminal block (Ctrl+I to open Agent Mode). While it runs,
check the liter-llm output in the terminal where you started it — you should see
a `/v1/chat/completions` request logged.

To confirm no requests go to `oz.warp.dev`:

```bash
# In a separate terminal while using Warp AI:
lsof -i -n | grep warp-oss | grep ESTABLISHED
```

All outbound connections should be to `127.0.0.1:4000`, not to `oz.warp.dev`.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| "AI unavailable" in Warp | `WARP_OSS_LLM_PROXY_URL` not set | Launch with the env var or add to shell profile |
| Login modal appears | API key not entered in Settings → AI | Enter key; BYOK gate requires at least one key |
| Proxy returns 401 | Wrong API key in env var | Check `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` value |
| Proxy not reachable | Proxy not running | Run `liter-llm api --config ...` first |
| Unknown model error | Provider model name wrong | Check provider's model list |

## Running the proxy as a background service (macOS)

Create `~/Library/LaunchAgents/dev.warp.liter-llm.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple Computer//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>dev.warp.liter-llm</string>
  <key>ProgramArguments</key>
  <array>
    <string>/opt/homebrew/bin/liter-llm</string>
    <string>api</string>
    <string>--config</string>
    <string>/Users/YOUR_USERNAME/.config/warp-oss/liter-llm-proxy.toml</string>
  </array>
  <key>EnvironmentVariables</key>
  <dict>
    <key>ANTHROPIC_API_KEY</key>
    <string>sk-ant-REPLACE_ME</string>
  </dict>
  <key>RunAtLoad</key>
  <true/>
  <key>KeepAlive</key>
  <true/>
  <key>StandardOutPath</key>
  <string>/tmp/liter-llm.log</string>
  <key>StandardErrorPath</key>
  <string>/tmp/liter-llm.err</string>
</dict>
</plist>
```

Load it:

```bash
launchctl load ~/Library/LaunchAgents/dev.warp.liter-llm.plist
```

Then add `WARP_OSS_LLM_PROXY_URL=http://127.0.0.1:4000` to your shell profile and
restart your terminal. The proxy will start automatically at login.
