# ai-sudo

**Secure remote sudo approval for AI assistants**

A self-hosted sudo approval system that sends push notifications to your phone when an AI assistant (like Clawdbot) needs elevated privileges. Approve or deny requests remotely without giving blanket sudo access.

## What It Is

ai-sudo is a PAM (Pluggable Authentication Module) based system that intercepts sudo requests and routes them through a mobile approval workflow. When an AI needs to run a privileged command, you get a notification on your phone showing:

- The command being requested
- Which AI/process is requesting it
- The context/cwd of the request

You can then tap **Approve** or **Deny** from anywhere. The decision is communicated back to your machine, which either allows or blocks the command.

## Why It Should Be Built

AI assistants are becoming integral parts of our workflows, but they face a fundamental security constraint: they often need to run administrative commands but can't type passwords. This creates a dangerous tradeoff:

| Current Options | Problem |
|-----------------|---------|
| Grant full sudo | Complete security failure - AI can do anything |
| Physical presence | Defeats the purpose of remote AI assistance |
| Pre-approved commands | Inflexible - breaks when commands vary |

ai-sudo bridges this gap by adding a **human-in-the-loop** approval step. The AI can request privileges, but a human must explicitly approve each sensitive operation. This transforms AI + sudo from a security liability into a secure, auditable workflow.

## Problem Statement

**The Core Problem:** AI assistants cannot handle sudo password prompts, creating a security bottleneck in AI-automated workflows.

When AI tooling needs elevated access, we currently have only bad options:
1. **Unconditional trust** - Give AI full sudo (catastrophic if compromised)
2. **Manual intervention** - Require human at the keyboard (breaks automation)
3. **Static allowlists** - Only work for predictable, repetitive commands

This affects not just AI assistants but any headless system where privileged commands need occasional approval. The existing enterprise solutions (Duo, Teleport) are:

- Over-engineered for personal use
- Not self-hosted friendly
- Expensive and complex

## Solution Overview

ai-sudo implements a **human-gated sudo** pattern:

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
                                         │ or Clawdbot │
                                         │ iOS node    │
                                         └─────────────┘
```

### How It Works

1. **Intercept** - A user or AI runs `sudo <command>`
2. **Pause** - The PAM module intercepts the request and notifies the daemon
3. **Notify** - The daemon sends a push notification with command details
4. **Approve/Deny** - Human reviews and responds via phone app or messaging bot
5. **Execute** - PAM receives the decision and allows/blocks accordingly

### Key Features

- **Timeout support** - Auto-deny after configurable seconds (prevents hanging)
- **Rich context** - Shows command, user, process, working directory
- **Audit logging** - Complete trail of all requests and decisions
- **Optional allowlist** - Auto-approve known-safe commands (e.g., `brew upgrade`)
- **Fallback mode** - If the service is down, fall back to local password
- **Integration ready** - Works with Clawdbot iOS node or Telegram/Signal bots

## Getting Started

See [agents/ARCHITECTURE.md](agents/ARCHITECTURE.md) for implementation details.

## Security Considerations

- Notification channel must be authenticated (prevent spoofing)
- Nonce-based responses prevent replay attacks
- Rate limiting prevents denial-of-service
- All decisions are logged for audit

## License

MIT
