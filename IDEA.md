# ai-sudo (aisudo)

**Date:** 2026-01-10
**Status:** Idea
**Tags:** #security #macos #ai-tooling

## Problem

When AI assistants (like Clawdbot) need to run commands requiring sudo, there's no good way to approve them remotely. Options are:
- Give AI full sudo access (dangerous)
- Be physically present to type password (defeats remote use)
- Pre-approve specific commands (inflexible)

## Solution

A self-hosted sudo approval system with mobile notifications:

1. **PAM module** intercepts sudo requests
2. **Local service** receives the request, sends push notification
3. **Phone app** shows: command, user, context → Approve/Deny
4. **PAM module** receives response, allows or blocks

## Architecture

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│ sudo cmd    │────▶│ PAM Module   │────▶│ aisudo      │
│ (terminal)  │     │ (pam_aisudo) │     │ daemon      │
└─────────────┘     └──────────────┘     └──────┬──────┘
                                                │
                           Push notification    │
                                                ▼
                                         ┌─────────────┐
                                         │ Phone App   │
                                         │ (or Clawd   │
                                         │  iOS node)  │
                                         └─────────────┘
```

## Features

- **Timeout** - Auto-deny after X seconds
- **Context** - Show which AI/process requested it
- **Logging** - Full audit trail
- **Allowlist** - Pre-approve safe commands (optional)
- **Integration** - Could use Clawdbot iOS/Android node for notifications

## Security Considerations

- Notification channel must be authenticated
- Replay attack prevention (nonces)
- Rate limiting on requests
- Fallback to local password if service down

## Existing Tools (none fit)

- **Duo Security** - Enterprise, not self-hosted
- **Teleport** - Overkill for personal use
- **Custom PAM + webhook** - DIY, this is basically that but polished

## Implementation Notes

- macOS uses OpenPAM (BSD-style)
- PAM config: `/etc/pam.d/sudo`
- Could start as CLI tool, add GUI later
- Rust or Go for the daemon (security-critical)

## MVP

1. PAM module in C (required for PAM)
2. Simple daemon with HTTP endpoint
3. Telegram/Signal notification (before dedicated app)
4. CLI to approve from phone via bot command
