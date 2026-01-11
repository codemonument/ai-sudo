# ai-sudo Architecture & Implementation Guide

This document provides a comprehensive technical guide for implementing ai-sudo.

## Architecture Components

### 1. PAM Module (`pam_aisudo`)

The core interception layer that integrates with the system's authentication stack.

**Responsibilities:**
- Intercept sudo authentication requests
- Extract command, user, and context information
- Communicate with the local aisudo daemon
- Allow or block based on daemon response

**Technical Details:**
- Written in C (required for PAM modules)
- Uses OpenPAM API (macOS/BSD compatible)
- Pluggable into `/etc/pam.d/sudo`
- Thread-safe implementation

**Configuration:**
```c
// pam_aisudo.h
#define PAM_EXAMPLE_MODULE_NAME "pam_aisudo"

typedef struct {
    int debug;
    int timeout;
    char *socket_path;
    char *allowed_users;
} pam_aisudo_options_t;
```

### 2. AISudo Daemon (`aisudo-daemon`)

The central service that manages approval workflow.

**Responsibilities:**
- Listen for PAM module requests via Unix socket
- Send push notifications to mobile devices
- Manage approval state and timeouts
- Provide HTTP API for approval queries
- Log all events for audit

**Technical Details:**
- Written in Rust (memory safety, async performance)
- Uses Tokio async runtime
- Unix socket communication with PAM module
- SQLite for request state persistence

**Architecture:**
```
┌─────────────────────────────────────────────────────────┐
│                    aisudo-daemon                        │
├─────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │ Socket      │  │ State       │  │ HTTP API    │     │
│  │ Listener    │  │ Manager     │  │ Server      │     │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘     │
│         │                │                │             │
│  ┌──────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐     │
│  │ Request     │  │ SQLite      │  │ /approve    │     │
│  │ Queue       │  │ Database    │  │ /deny       │     │
│  └─────────────┘  └─────────────┘  │ /status     │     │
│                                    └─────────────┘     │
│  ┌──────────────────────────────────────────────┐      │
│  │ Notification Backend (Pluggable)             │      │
│  │ - Clawdbot iOS node                          │      │
│  │ - Telegram bot                               │      │
│  │ - Signal bot                                 │      │
│  └──────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────┘
```

### 3. Notification Backend

**Interface:**
```rust
trait NotificationBackend {
    async fn send_request(&self, request: PendingRequest) -> Result<(), Error>;
    async fn send_response(&self, response: ApprovalResponse) -> Result<(), Error>;
}
```

**Available Backends:**
1. **Clawdbot iOS Node** - Direct push via existing infrastructure
2. **Telegram Bot** - Rich formatting, inline keyboards
3. **Signal Bot** - E2E encrypted notifications

### 4. Mobile Client (Future)

Native iOS/Android app for streamlined approval experience.

## Technology Recommendations

### Language Selection

| Component | Language | Rationale |
|-----------|----------|-----------|
| PAM Module | C | Required by PAM API; OpenPAM on macOS |
| Daemon | Rust | Memory safety, async performance, good crypto libs |
| Mobile App | Swift (iOS) / Kotlin (Android) | Native platform support |

### Key Libraries

**Rust (Daemon):**
- `tokio` - Async runtime
- `sqlx` - SQLite with async
- `rusqlite` - SQLite bindings
- `tungstenite` - WebSocket support
- `uuid` - Request tracking
- `serde`/`serde_json` - Serialization

**C (PAM Module):**
- OpenPAM headers (`<security/pam_modules.h>`)
- Darwin/macOS syslog

### Database Schema

```sql
CREATE TABLE requests (
    id TEXT PRIMARY KEY,
    user TEXT NOT NULL,
    command TEXT NOT NULL,
    cwd TEXT,
    pid INTEGER,
    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
    status TEXT DEFAULT 'pending', -- pending, approved, denied, timeout
    timeout_seconds INTEGER DEFAULT 30,
    decided_at DATETIME,
    decided_by TEXT,
    audit_log TEXT
);

CREATE TABLE audit_log (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    request_id TEXT,
    event TEXT,
    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
    details TEXT
);
```

## Implementation Steps

### Phase 1: Core Infrastructure

#### Step 1.1: Set up Rust Project Structure
```bash
cargo new aisudo-daemon
cd aisudo-daemon
cargo add tokio rusqlite sqlx uuid serde serde_json
```

#### Step 1.2: Create PAM Module Skeleton
```c
// pam_aisudo.c
#include <security/pam_modules.h>
#include <security/pam_ext.h>
#include <stdio.h>
#include <stdlib.h>

PAM_EXTERN int pam_sm_authenticate(pam_handle_t *pamh, int flags, int argc, const char **argv) {
    // Extract command and user info
    // Send to daemon socket
    // Wait for response
    return PAM_SUCCESS; // or PAM_AUTH_ERR
}

PAM_EXTERN int pam_sm_setcred(pam_handle_t *pamh, int flags, int argc, const char **argv) {
    return PAM_SUCCESS;
}
```

#### Step 1.3: Implement Unix Socket Communication

**Daemon side (Rust):**
```rust
use std::os::unix::net::UnixListener;

async fn handle_connection(stream: UnixStream) {
    let request: Request = deserialize_from_stream(stream).await?;
    let request_id = process_request(request).await;
    send_notification(request_id).await;
}
```

**PAM module side (C):**
```c
int send_to_daemon(const char *socket_path, const char *json_data, char *response, size_t response_size) {
    int sock = socket(AF_UNIX, SOCK_STREAM, 0);
    connect(sock, socket_path, sizeof(sockaddr_un));
    write(sock, json_data, strlen(json_data));
    read(sock, response, response_size);
    close(sock);
    return 0;
}
```

### Phase 2: Notification Integration

#### Step 2.1: Clawdbot iOS Node Integration

Use the existing `nodes` API to send notifications:

```rust
async fn send_push_via_nodes(request_id: &str, details: &RequestDetails) -> Result<()> {
    let payload = json!({
        "type": "sudo_request",
        "request_id": request_id,
        "command": details.command,
        "user": details.user,
        "timeout": details.timeout_seconds
    });
    
    nodes::notify("ios", &payload).await?;
    Ok(())
}
```

#### Step 2.2: Telegram Bot Backend

```rust
struct TelegramBackend {
    token: String,
    chat_id: String,
}

impl NotificationBackend for TelegramBackend {
    async fn send_request(&self, request: PendingRequest) -> Result<()> {
        let text = format!(
            "🔐 *Sudo Request*\n\n\
             *Command:* `{}`\n\
             *User:* {}\n\
             *PID:* {}\n\n\
             ⏱️ Auto-deny in {} seconds",
            request.command.escape_markdown(),
            request.user,
            request.pid,
            request.timeout_seconds
        );
        
        telegram_send_message(&self.token, &self.chat_id, &text).await?;
        Ok(())
    }
}
```

### Phase 3: Approval Workflow

#### Step 3.1: HTTP API Endpoints

```rust
#[get("/approve/<request_id>")]
async fn approve(request_id: PathBuf) -> impl Responder {
    state.mark_approved(&request_id, "telegram").await;
    HttpResponse::Ok().body("✅ Approved")
}

#[get("/deny/<request_id>")]
async fn deny(request_id: PathBuf) -> impl Responder {
    state.mark_denied(&request_id, "telegram").await;
    HttpResponse::Ok().body("🚫 Denied")
}
```

#### Step 3.2: Timeout Handling

```rust
async fn handle_timeout(request_id: &str) {
    let mut state = state.lock().await;
    if state.get_status(request_id) == Status::Pending {
        state.mark_timeout(request_id).await;
        send_notification_timeout(request_id).await;
    }
}
```

### Phase 4: Security Hardening

#### Step 4.1: Nonce-Based Response Validation

```rust
struct ApprovalRequest {
    request_id: String,
    nonce: String,  // Cryptographic random
    timestamp: u64,
}

struct ApprovalResponse {
    request_id: String,
    nonce: String,
    decision: Decision,  // approve/deny
    signature: Vec<u8>,  // HMAC or Ed25519
}
```

#### Step 4.2: Rate Limiting

```rust
struct RateLimiter {
    requests_per_minute: usize,
    // Use redis or in-memory with atomic counters
}

impl RateLimiter {
    async fn check(&self, user: &str) -> Result<(), Error> {
        let count = self.get_count(user).await;
        if count >= self.requests_per_minute {
            return Err(Error::RateLimited);
        }
        self.increment(user).await;
        Ok(())
    }
}
```

### Phase 5: Testing

#### Unit Tests
```bash
# Test Rust daemon
cargo test

# Test C module
gcc -Wall -Wextra -o pam_aisudo_test pam_aisudo.c -lcunit
```

#### Integration Tests
```bash
# Test full flow with mock notification
./scripts/integration_test.sh --mock-notifications

# Test timeout behavior
./scripts/integration_test.sh --test-timeout=5s
```

## Deployment

### Installation
```bash
# Install PAM module
sudo cp pam_aisudo.so /usr/lib/pam/
sudo chmod 755 /usr/lib/pam/pam_aisudo.so

# Configure PAM
echo "auth sufficient pam_aisudo.so timeout=30" | sudo tee /etc/pam.d/sudo

# Install daemon
cargo install --path aisudo-daemon
sudo cp aisudo-daemon.service /etc/systemd/system/
sudo systemctl enable aisudo-daemon
sudo systemctl start aisudo-daemon
```

### macOS Notes
- Uses OpenPAM (BSD-style PAM)
- PAM config location: `/etc/pam.d/sudo`
- Daemon should run as launchd agent
- Code signing required for PAM modules

## Future Enhancements

- [ ] Native iOS/Android apps
- [ ] Biometric authentication on mobile
- [ ] Command preview (dry run before execution)
- [ ] Batch approval for multiple similar commands
- [ ] Integration with YubiKey/fIDO2
- [ ] Multi-user approval workflows
