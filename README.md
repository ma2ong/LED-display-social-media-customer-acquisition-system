# LED Display Social Media Customer Acquisition System

Automated B2B outreach system for Maicai Visual (LED display manufacturer, Shenzhen).
Targets overseas LED integrators, rental companies, installers via Instagram & Facebook.

## Quick Start

```bash
# Day 1 — warm up accounts
python daily_runner.py plan     # preview today's queue
python daily_runner.py warmup   # auto like + comment + follow via opencli

# Day 2 — send DMs
python daily_runner.py send     # generate message → open profile → paste & confirm
```

## Config

Edit `TARGET_COUNTRIES` in `daily_runner.py` to change target market (default: `["USA"]`).

## Requirements

- `npm install -g @jackwener/opencli` (v1.7.14+)
- Chrome logged into Instagram & Facebook
- Claude Code (uses `claude -p` for message generation — no API key needed)

## Daily Limits

| Platform | DMs/day | Min interval |
|----------|---------|-------------|
| Instagram | 8 | 20 min |
| Facebook | 5 | 30 min |
| Combined | 15 | — |

Sends only Tue–Thu, 10:00–16:00 target local time.

## Files

| File | Purpose |
|------|---------|
| `daily_runner.py` | Main entry: plan / warmup / send / status |
| `message_crafter.py` | Personalized message generation via Claude |
| `qa_checker.py` | Pre-send compliance checks (9 rules) |
| `pipeline_init.py` | One-time init from leads CSV/scripts |
| `SKILL.md` | Full skill documentation for Claude Code |
