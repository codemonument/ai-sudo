# ADR-001: Rust PAM Module - nonstick Chosen

**Date:** 2026-01-12
**Status:** DECIDED
**Decision:** Use nonstick crate for PAM module

## Context

We need to implement the PAM (Pluggable Authentication Module) component for ai-sudo. The module must:
- Intercept sudo authentication requests
- Communicate with the aisudo daemon
- Support both Linux-PAM and OpenPAM/macOS
- Be written in Rust (not C)

## Options Evaluated

| Crate | Approach | macOS/OpenPAM | Maintenance | Effort | Risk |
|-------|----------|---------------|-------------|--------|------|
| **nonstick** | Trait-based wrapper | ✅ Explicit support | Active (2024) | Low | Low |
| **pamsm** | Service Module focus | ✅ Should work | Stable (older) | Low | Low |
| **pam-sys** | Raw FFI bindings | ⚠️ Linux-focused | Active | High | Medium |

### Option 1: nonstick

**Approach:** Modern, trait-based Rust wrapper around PAM

**Pros:**
- Modern Rust patterns (traits, macros)
- Handles extern "C" boilerplate with `pam_export!` macro
- Explicit cross-platform support: Linux-PAM + OpenPAM/macOS
- Type-safe wrapper reduces memory safety issues
- Well-documented API
- ~1-2 weeks for MVP module

**Cons:**
- Requires understanding of trait-based design
- Newer crate (less battle-tested than C alternatives)

**Repository:** https://github.com/nicopop91/nonstick

### Option 2: pamsm

**Approach:** Lightweight wrapper specifically for Service Modules

**Pros:**
- Mature, stable codebase
- Macro for module entry points
- Minimal overhead
- ~1-2 weeks for MVP module

**Cons:**
- Older API design (pre-traits era)
- Less explicit macOS/OpenPAM documentation
- Less active maintenance

**Repository:** https://github.com/wfraser/pamsm

### Option 3: pam-sys / libpam-sys

**Approach:** Raw FFI bindings for maximum control

**Pros:**
- Maximum flexibility and control
- Used in production by Tailscale (pam_tailscale)
- No abstraction overhead

**Cons:**
- Significant boilerplate required
- Manual memory management required
- Linux-focused (macOS support unclear)
- ~3-4 weeks for MVP module
- Higher security risk (manual memory management)

**Repository:** https://github.com/tailscale/pam_tailscale (example usage)

## Decision

**Choose: nonstick**

### Rationale

1. **Type Safety:** nonstick's trait-based approach provides better type safety than raw FFI bindings
2. **Cross-Platform:** Explicit OpenPAM/macOS support matches our target platform
3. **Development Speed:** 1-2 weeks vs 3-4 weeks for pam-sys
4. **Modern Rust:** Aligns with our daemon's Rust implementation
5. **Risk Mitigation:** Low risk despite being a newer crate; well-documented API

## Implementation Plan

1. Add `nonstick` crate to `Cargo.toml`
2. Implement `PamModule` trait with authentication logic
3. Use `pam_export!` macro for C entry points
4. Test on both Linux (PAM) and macOS (OpenPAM)

## References

- nonstick documentation: https://docs.rs/nonstick/latest/nonstick/
- nonstick repository: https://github.com/nicopop91/nonstick
- OpenPAM (macOS): https://www.freebsd.org/cgi/man.cgi?query=openpam
- Linux PAM: https://github.com/linux-pam/linux-pam
