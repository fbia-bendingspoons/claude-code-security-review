# Security Analysis: Root Privilege Check Bypass Vectors

## Context

The Claude Code CLI includes a security check that prevents using `--dangerously-skip-permissions` when running as root/sudo:

```
--dangerously-skip-permissions cannot be used with root/sudo privileges for security reasons
```

This document analyzes potential bypass vectors for such root privilege detection mechanisms.

**Note:** This repository (`claude-code-security-review`) is a GitHub Action that uses the Claude CLI as a dependency. The actual root check implementation is in the Claude Code CLI npm package (`@anthropic-ai/claude-code`), not in this repository.

## Potential Bypass Vectors

### 1. UID vs EUID Check Inconsistency

**Risk Level:** HIGH

Most Node.js/JavaScript implementations use `process.getuid()` to check for root privileges. However, there are multiple user ID types:

- **Real UID (RUID)**: The actual user who started the process
- **Effective UID (EUID)**: The user ID used for permission checks
- **Saved UID (SUID)**: Allows temporary privilege dropping

**Potential Bypass:**
```bash
# If only EUID is checked, setuid binaries could bypass
# If only RUID is checked, sudo could be bypassed in certain configurations
```

**Node.js APIs:**
- `process.getuid()` - Returns effective UID
- `process.geteuid()` - Returns effective UID (alias in Node)

**Recommendation:** Check both `process.getuid() === 0` AND ensure no elevated capabilities are present.

### 2. Environment Variable Spoofing

**Risk Level:** MEDIUM

Some root checks rely on environment variables like `SUDO_USER`, `SUDO_UID`, or `USER`.

**Potential Bypass:**
```bash
# Unset sudo-related environment variables before running
unset SUDO_USER SUDO_UID SUDO_GID SUDO_COMMAND
claude --dangerously-skip-permissions -p "command"
```

**Recommendation:** Never rely solely on environment variables for security decisions. Always use system calls (`getuid()`/`geteuid()`).

### 3. User Namespace / Container Scenarios

**Risk Level:** HIGH

In containerized environments or user namespaces, a user can appear as root (UID 0) inside the namespace while being unprivileged on the host.

**Potential Bypass:**
```bash
# Create a user namespace where current user maps to root
unshare --user --map-root-user claude --dangerously-skip-permissions -p "command"
```

**Why this matters:** The process sees `getuid() === 0`, but it's actually running with the original user's privileges on the host.

**Recommendation:**
- Check `/proc/self/uid_map` to detect user namespace remapping
- Consider whether namespace-root should be treated differently

### 4. Capabilities-Based Bypass

**Risk Level:** MEDIUM

Linux capabilities allow fine-grained privilege management. A non-root user might have dangerous capabilities without UID 0.

**Potential Bypass:**
```bash
# A binary with CAP_SYS_ADMIN but running as non-root user
# would pass UID check but have root-like powers
sudo setcap cap_sys_admin+ep /path/to/node
```

**Recommendation:** Consider checking for dangerous capabilities:
```javascript
const { execSync } = require('child_process');
const caps = execSync('getpcaps $$').toString();
// Check for CAP_SYS_ADMIN, CAP_DAC_OVERRIDE, etc.
```

### 5. setuid/setgid Binary Wrapper

**Risk Level:** MEDIUM

A setuid wrapper could drop privileges after starting but before the check.

**Potential Bypass:**
```c
// Wrapper that drops root before executing claude
int main() {
    setuid(getuid());  // Drop effective UID to real UID
    seteuid(getuid());
    execvp("claude", args);
}
```

**Recommendation:** Check saved UID as well, though this is harder in pure JavaScript.

### 6. Race Condition in Check

**Risk Level:** LOW

If the privilege check and the dangerous operation are not atomic, a TOCTOU (Time-of-check-time-of-use) attack might be possible.

**Scenario:**
1. Claude checks UID (non-root)
2. Attacker exploits vulnerability to gain root
3. Claude executes with root privileges

**Recommendation:** Perform privilege checks immediately before each dangerous operation, not just at startup.

### 7. Argument Injection via Environment

**Risk Level:** LOW-MEDIUM

If the CLI reads additional arguments from environment variables, an attacker might bypass the flag by using env vars.

**Potential Bypass:**
```bash
# If NODE_OPTIONS or similar is used to inject arguments
NODE_OPTIONS="--experimental-loader=./exploit.mjs" sudo claude -p "command"
```

**Recommendation:** Ensure the root check cannot be circumvented via environment-injected arguments.

### 8. Symbolic Link / Path Confusion

**Risk Level:** LOW

If the CLI checks its own path to determine execution context, symlinks could confuse it.

**Recommendation:** Use `realpath()` and canonical paths for any path-based security decisions.

## Recommendations for Claude Code CLI

1. **Multi-layered UID check:**
```javascript
function isRunningAsRoot() {
    // Check effective UID
    if (process.getuid && process.getuid() === 0) return true;

    // Check for user namespace with root mapping
    try {
        const uidMap = fs.readFileSync('/proc/self/uid_map', 'utf8');
        // If mapped to 0, we're in a user namespace
        if (uidMap.includes('0')) {
            // Decide policy: block or allow namespace-root
        }
    } catch {}

    return false;
}
```

2. **Capability awareness:**
```javascript
function hasDangerousCapabilities() {
    try {
        const caps = execSync('getpcaps $$', { encoding: 'utf8' });
        const dangerous = ['cap_sys_admin', 'cap_dac_override', 'cap_setuid'];
        return dangerous.some(cap => caps.toLowerCase().includes(cap));
    } catch {
        return false; // Conservative: assume safe if we can't check
    }
}
```

3. **Environment sanitization:**
- Don't rely on `SUDO_*` environment variables
- Document that `NODE_OPTIONS` and similar should be sanitized

4. **Defense in depth:**
- The root check is one layer; consider sandboxing (seccomp, namespaces) for additional protection
- Consider dropping capabilities even when running as non-root

## Testing Recommendations

To verify the robustness of the root check:

```bash
# Test 1: Direct root
sudo claude --dangerously-skip-permissions -p "test"
# Expected: BLOCKED

# Test 2: User namespace root
unshare --user --map-root-user claude --dangerously-skip-permissions -p "test"
# Expected: Depends on policy decision

# Test 3: Environment variable manipulation
sudo -E env SUDO_USER="" claude --dangerously-skip-permissions -p "test"
# Expected: BLOCKED

# Test 4: Capability-only root
sudo setcap cap_sys_admin+ep $(which node)
claude --dangerously-skip-permissions -p "test"
# Expected: Consider blocking
```

## Conclusion

Without access to the actual Claude Code CLI source code, this analysis covers common bypass vectors for root privilege checks. The implementation should be audited against each of these scenarios to ensure comprehensive protection.

The security goal of preventing `--dangerously-skip-permissions` from running as root is sound, as it prevents privilege escalation attacks where an attacker with root access could use Claude to execute arbitrary commands without any permission restrictions.
