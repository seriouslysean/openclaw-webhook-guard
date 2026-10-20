# Changelog

## Unreleased

First version, built and tested against OpenClaw 2026.10.4.

- Blocks scheduling calls (`cron`, `automations`) that create or change a job with a webhook route.
- Blocks scheduling calls it can't read and actions it doesn't know.
- Checks every agent; `allowAgents` exempts listed ones.
