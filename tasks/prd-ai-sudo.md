# PRD: ai-sudo - Secure Remote Sudo Approval

## Introduction
ai-sudo is a self-hosted PAM-based sudo approval system that enables remote human-in-the-loop authorization for AI assistants. When an AI needs elevated privileges, users receive push notifications on their phone to approve or deny requests.

## Goals
- Create a secure, auditable workflow for AI assistants to request sudo access
- Enable remote approval without physical presence or blanket sudo access
- Provide rich context (command, user, PID, cwd) for informed decisions
- Support multiple notification backends (Clawdbot iOS node, Telegram, Signal)
- Implement security best practices (nonces, rate limiting, replay prevention)

## User Stories

### US-001: PAM Module Interception
**Description:** As the system, I need to intercept sudo requests and communicate with the aisudo daemon so that privileged commands can be gated behind human approval.

**Acceptance Criteria:**
- PAM module written in C using OpenPAM API
- Extract command, user, PID, and working directory from PAM handle
- Communicate with daemon via Unix socket
- Allow or block based on daemon response
- Typecheck passes

### US-002: Daemon Setup and Socket Communication
**Description:** As the daemon, I need to listen for PAM requests via Unix socket and manage the approval state so that I can orchestrate the approval workflow.

**Acceptance Criteria:**
- Rust daemon with Tokio async runtime
- Listen on Unix socket for PAM module connections
- Deserialize request JSON from socket
- Return approval/deny response
- Typecheck passes

### US-003: Request State Persistence
**Description:** As the system, I need to persist pending requests in SQLite so that approval state survives daemon restarts and timeouts can be enforced.

**Acceptance Criteria:**
- Create SQLite database with requests and audit_log tables
- Store request_id, user, command, cwd, pid, timestamp, status, timeout_seconds
- Mark requests as pending/approved/denied/timeout
- Typecheck passes

### US-004: Notification Backend Interface
**Description:** As the system, I need a pluggable notification backend interface so that different notification channels can be supported.

**Acceptance Criteria:**
- Define NotificationBackend trait in Rust
- Interface includes send_request() and send_response() methods
- Concrete implementations for Clawdbot iOS node, Telegram, Signal
- Typecheck passes

### US-005: Clawdbot iOS Node Notification
**Description:** As a user, I want to receive sudo request notifications via the Clawdbot iOS node so that I can approve requests from my phone.

**Acceptance Criteria:**
- Implement NotificationBackend for Clawdbot nodes API
- Send push notification with request_id, command, user, timeout
- Include rich context (command, user, PID, cwd)
- Typecheck passes

### US-006: Telegram Bot Backend
**Description:** As a user, I want to receive sudo request notifications via Telegram so that I can approve requests using inline keyboards.

**Acceptance Criteria:**
- Implement NotificationBackend for Telegram Bot API
- Format notification with Markdown escape for command
- Include Approve/Deny inline keyboard URLs
- Typecheck passes

### US-007: Signal Bot Backend
**Description:** As a privacy-conscious user, I want to receive sudo request notifications via Signal so that my approval workflow is end-to-end encrypted.

**Acceptance Criteria:**
- Implement NotificationBackend for Signal Bot API
- Send notification with request details
- Provide approval/deny URL endpoints
- Typecheck passes

### US-008: HTTP API for Approval
**Description:** As the notification backend, I need HTTP endpoints to receive approval/deny responses so that users can interact with requests from their phone.

**Acceptance Criteria:**
- Implement /approve/<request_id> endpoint
- Implement /deny/<request_id> endpoint
- Update request status in SQLite
- Return confirmation response
- Typecheck passes

### US-009: Timeout Handling
**Description:** As the system, I need to auto-deny requests after a configurable timeout so that commands don't hang indefinitely waiting for approval.

**Acceptance Criteria:**
- Track timeout per request (default 30 seconds)
- Spawn async timeout task for each pending request
- Mark request as timeout status on expiry
- Send timeout notification to user
- Typecheck passes

### US-010: Audit Logging
**Description:** As a security-conscious user, I want a complete audit trail of all requests and decisions so that I can review security events.

**Acceptance Criteria:**
- Log all requests to audit_log table
- Log approval/deny/timeout decisions with timestamp and decision source
- Provide CLI command to view audit log
- Typecheck passes

### US-011: Rate Limiting
**Description:** As a security-conscious user, I want to prevent denial-of-service attacks through rapid sudo requests so that the system remains responsive.

**Acceptance Criteria:**
- Track requests per user with time windows
- Reject requests exceeding limit (e.g., 10/minute)
- Return error to PAM module for fallback to password
- Typecheck passes

### US-012: Nonce-Based Response Validation
**Description:** As a security-conscious user, I want to prevent replay attacks on approval responses so that old or intercepted responses can't be reused.

**Acceptance Criteria:**
- Generate cryptographic nonce for each request
- Include nonce in notification and require in response
- Validate nonce before processing approval/deny
- Typecheck passes

### US-013: PAM Configuration
**Description:** As a user, I want to configure PAM to use ai-sudo so that the system can intercept sudo requests.

**Acceptance Criteria:**
- Document PAM configuration for /etc/pam.d/sudo
- Support configurable timeout option
- Support optional allowed_users allowlist
- Typecheck passes

### US-014: Daemon Installation and Service
**Description:** As a user, I want to install ai-sudo as a system service so that the daemon runs automatically on boot.

**Acceptance Criteria:**
- Create systemd service file for Linux
- Create launchd plist for macOS
- Support enable/disable on boot
- Typecheck passes

### US-015: macOS OpenPAM Compatibility
**Description:** As a macOS user, I want ai-sudo to work with OpenPAM so that the system functions correctly on my platform.

**Acceptance Criteria:**
- Use OpenPAM headers and API (not Linux PAM)
- Compile successfully on macOS with clang
- Test PAM module integration on macOS
- Typecheck passes

### US-016: CLI Tool for Local Testing
**Description:** As a developer, I want a CLI tool to test the approval workflow locally so that I can debug without triggering actual sudo.

**Acceptance Criteria:**
- Create aisudo CLI that simulates PAM module
- Send test request to daemon
- Print notification status
- Typecheck passes

## Non-Goals
- Native mobile apps (iOS/Android) - future enhancement
- Biometric authentication - future enhancement
- YubiKey/FIDO2 integration - future enhancement
- Multi-user approval workflows - future enhancement

## Technical Considerations
- PAM module must be written in C due to OpenPAM API requirements
- Daemon written in Rust for memory safety and async performance
- Use sqlx or rusqlite for SQLite database
- Notification backends are pluggable via trait
- Unix socket for fast local IPC with PAM module
- HTTP API for receiving responses from notification backends

## Out of Scope
- Command preview/dry-run feature
- Batch approval for multiple commands
- Integration with enterprise SSO providers
