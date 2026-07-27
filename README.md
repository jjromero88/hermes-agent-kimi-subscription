# Hermes Agent + Kimi Code via Subscription

*Leer esto en [español](README.es.md).*

> A guide to configuring **Hermes Agent** to use your **Kimi subscription** (Kimi Code), drawing from your membership quota instead of paying per token.

Hermes Agent natively includes the `kimi-coding` provider. This means that if you have a Kimi membership (for example, the $19/month plan), you can generate a **Kimi Code** API key and use it directly in Hermes: usage is deducted from your subscription's weekly quota, with no per-token billing.

---

## Table of contents

- [How it works](#how-it-works)
- [Requirements](#requirements)
- [Step 1 — Update Hermes Agent](#step-1--update-hermes-agent)
- [Step 2 — Generate the Kimi Code API key](#step-2--generate-the-kimi-code-api-key)
- [Step 3 — Save the key in Hermes' environment](#step-3--save-the-key-in-hermes-environment)
- [Step 4 — Configure the provider in config.yaml](#step-4--configure-the-provider-in-configyaml)
- [Step 5 — Test the configuration](#step-5--test-the-configuration)
- [Step 6 — Restart the service (24/7 installs)](#step-6--restart-the-service-247-installs)
- [Verifying subscription usage](#verifying-subscription-usage)
- [Subscription limits](#subscription-limits)
- [Optional: fallback model](#optional-fallback-model)
- [Troubleshooting](#troubleshooting)
- [References](#references)

---

## How it works

Your Kimi membership includes **Kimi Code**, an AI coding assistant service that lets you generate up to **5 API keys** from its web console. These keys:

- Start with the `sk-kimi-` prefix.
- **Don't charge per token**: they draw from your subscription's weekly quota (automatically renewed every 7 days).
- Work with third-party agents through the `https://api.kimi.com/coding/v1` endpoint, using the `kimi-for-coding` model.

Hermes Agent ships with this endpoint pre-configured under the `kimi-coding` provider, so setup comes down to two files: `~/.hermes/.env` (the key) and `~/.hermes/config.yaml` (the provider).

> ⚠️ **Don't confuse this** with API keys from `platform.moonshot.ai`: those belong to Moonshot's pay-per-token platform and **do not** draw from your subscription. The only valid console for this method is `kimi.com/code/console`.

## Requirements

- Hermes Agent installed, on a reasonably recent version (see Step 1).
- An active Kimi membership that includes Kimi Code.
- Linux/macOS with terminal access (examples use Ubuntu).

## Step 1 — Update Hermes Agent

Before anything else, update Hermes to the latest version — very old installs only knew about the classic Moonshot endpoint (pay-per-token) and didn't ship the `kimi-coding` provider:

```bash
hermes update
```

Check the installed version:

```bash
hermes version
```

There's no strict, publicly documented minimum version — just make sure you're on a recently updated install. If the `kimi-coding` provider isn't recognized after Step 4, an outdated install is almost always the cause (see [Troubleshooting](#troubleshooting)).

## Step 2 — Generate the Kimi Code API key

1. Open the Kimi Code console: **https://www.kimi.com/code/console**
2. Sign in with the account that has the active membership.
3. In the **API Keys** section, click **"+ Create API Key"**.
4. Give it a descriptive name (e.g. `hermes-agent`) and create it.
5. **Copy the key immediately** — it's only shown in full once. It should start with `sk-kimi-`.

## Step 3 — Save the key in Hermes' environment

Add the key to Hermes' environment file (the command creates the file if it doesn't exist):

```bash
echo 'KIMI_API_KEY=sk-kimi-YOUR-KEY-HERE' >> ~/.hermes/.env
chmod 600 ~/.hermes/.env
```

Replace `sk-kimi-YOUR-KEY-HERE` with your actual key, with no spaces around the `=`, keeping the single quotes as shown (they only matter for shell quoting — they won't end up in the file itself).

Verify it was saved correctly (a single `KIMI_API_KEY=` line, no duplicates):

```bash
grep KIMI_API_KEY ~/.hermes/.env
```

## Step 4 — Configure the provider in config.yaml

Back up your current configuration:

```bash
cp ~/.hermes/config.yaml ~/.hermes/config.yaml.bak
```

Edit the file:

```bash
nano ~/.hermes/config.yaml
```

Find the `model:` block near the top of the file. This block exists no matter which provider you currently have configured — it could be Nous, OpenRouter, Anthropic directly, OpenAI, a local model, etc. **Replace it entirely**, regardless of its current contents, with:

```yaml
model:
  default: kimi-for-coding
  provider: kimi-coding
```

Some examples of what the block might look like *before* replacing it, depending on your previous provider (these are just illustrative — yours may look different):

```yaml
# Example coming from Nous
model:
  default: deepseek/deepseek-v4-flash
  provider: nous
  base_url: https://inference-api.nousresearch.com/v1
```

```yaml
# Example coming from OpenRouter
model:
  default: anthropic/claude-opus-4.7
  provider: openrouter
```

Important points:

- If your previous block had a `base_url` line, **remove it entirely** (don't leave it empty): the `kimi-coding` provider already knows its endpoint, and a leftover URL from another provider will break the connection. This rule applies no matter which provider you're coming from.
- Don't modify any other section of the file (`agent`, `terminal`, `memory`, etc.).
- A more guided alternative: instead of hand-editing the YAML, you can run `hermes setup` or `hermes model` and pick "Kimi / Moonshot" from the interactive wizard — it makes the same change without touching the file directly.

Save with `Ctrl+O`, `Enter`, and exit with `Ctrl+X`.

## Step 5 — Test the configuration

```bash
hermes chat -q "Reply with just: PING" --max-turns 1
```

If it responds without authentication errors (`401`/`403`) or "unknown model" errors, Hermes is now drawing from your subscription. ✅

## Step 6 — Restart the service (24/7 installs)

If Hermes runs as a persistent service, restart it so it picks up the new configuration. With systemd (user mode):

```bash
systemctl --user list-units | grep -i hermes   # identify the service
systemctl --user restart <service-name>        # e.g. hermes-gateway.service
```

If it runs in `tmux`/`screen` or an interactive session, close and relaunch the process.

## Verifying subscription usage

Go back into **https://www.kimi.com/code/console** and check the dashboard:

- **Rate limit details**: moves almost immediately with the first requests (this is the 5-hour window).
- **Weekly usage**: reflects cumulative usage for the week; with small tests it may still show 0% due to rounding.
- **API Keys**: your key should show as `Enabled`.

If those indicators move when using Hermes, the setup is correct: you're consuming subscription quota, not tokens.

## Subscription limits

Kimi Code's quota works with two gauges:

| Gauge | Behavior |
|---|---|
| Weekly quota | Automatically renews every 7 days |
| Rate limit | Rolling ~5-hour window (roughly 300–1,200 requests depending on plan, up to 30 concurrent) |

> The exact request numbers above come from community reporting, not an official published table — treat them as approximate. The underlying mechanism (weekly quota + rolling 5-hour rate-limit window) is confirmed in Kimi's own docs.

In 24/7 operation you may hit the 5-hour window during usage spikes: you'll see temporary `429` errors that **resolve on their own** once the window renews. This isn't a configuration error.

## Optional: fallback model

Hermes supports automatic provider failover on `429`/`503`/`529` errors. If you have credentials for another provider (OpenRouter, Nous, etc.), you can uncomment and adapt the `fallback_model` section at the end of `config.yaml`:

```yaml
fallback_model:
  provider: openrouter
  model: anthropic/claude-sonnet-4
```

This way, if your Kimi quota runs out temporarily, Hermes keeps working with the fallback and switches back to Kimi once the window frees up.

> `fallback_model` (singular) is the classic form and still works, but Hermes also supports `fallback_models` (plural, as a list) if you want to chain more than one fallback. Use whichever fits how many backup providers you have.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `401` / `403` on the test | Key copied incorrectly, has spaces, or is duplicated in `.env` | Check `~/.hermes/.env`: a single, clean `KIMI_API_KEY=sk-kimi-...` line |
| `kimi-coding` provider not recognized | Hermes is outdated | `hermes update` and try again |
| Connection/endpoint error | A `base_url` line from another provider was left in the `model:` block | Remove it and restart |
| The key isn't drawing from the subscription | Key generated on `platform.moonshot.ai` (pay-per-token) | Generate the key at `kimi.com/code/console` instead |
| Intermittent `429`s under heavy use | Rate limit on the 5-hour window | Wait for renewal, or configure `fallback_model` |
| The 24/7 service is still using the old provider | The process wasn't restarted | `systemctl --user restart <service>` |

If something goes wrong and you need to roll back:

```bash
cp ~/.hermes/config.yaml.bak ~/.hermes/config.yaml
systemctl --user restart <service-name>
```

## References

- [Kimi Code — Official docs](https://www.kimi.com/code/docs/en/)
- [Kimi Code — Using it in third-party agents](https://www.kimi.com/code/docs/en/third-party-tools/other-coding-agents)
- [Hermes Agent — Supported providers](https://hermes-agent.nousresearch.com/docs/integrations/providers)
- [Hermes Agent — GitHub repository](https://github.com/NousResearch/hermes-agent)

---

*This guide is a community contribution and isn't affiliated with Moonshot AI or Nous Research. Prices, limits, and versions mentioned reflect July 2026 and are subject to change.*
