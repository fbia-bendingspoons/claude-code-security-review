# Troubleshooting

This document covers common issues users may encounter when setting up the Claude Code Security Review action.

## Claude Code Installation Issues

### macOS: "Trace/BPT trap: 5" Error

When installing Claude Code on macOS using the install script, you may encounter:

```
bash: line 142:  5675 Trace/BPT trap: 5          "$binary_path" install ${TARGET:+"$TARGET"}
```

**Cause:** macOS automatically quarantines files downloaded from the internet via the `com.apple.quarantine` extended attribute. Gatekeeper may block execution of binaries that aren't properly notarized.

**Solution:** Manually install Claude Code and remove the quarantine attribute:

```bash
# Set up install directory
INSTALL_DIR="${HOME}/.claude"
mkdir -p "$INSTALL_DIR"

# Detect architecture (arm64 for Apple Silicon, x64 for Intel)
ARCH=$(uname -m)
if [ "$ARCH" = "arm64" ]; then
    ARCH_PATH="arm64"
else
    ARCH_PATH="x64"
fi

# Download the binary
curl -fsSL "https://storage.googleapis.com/claude-code-dist-86c565f3-f756-42ad-8dfa-d59b1c096819/claude-code-releases/darwin/${ARCH_PATH}/latest/claude" -o "$INSTALL_DIR/claude"

# Remove macOS quarantine attribute
xattr -d com.apple.quarantine "$INSTALL_DIR/claude" 2>/dev/null || true

# Make executable and install
chmod +x "$INSTALL_DIR/claude"
"$INSTALL_DIR/claude" install
```

**Alternative:** If the above doesn't work, try right-clicking the binary in Finder and selecting "Open" to bypass Gatekeeper, then run the install command from terminal.

### Linux: Permission Denied

If you encounter permission errors on Linux:

```bash
chmod +x ~/.claude/claude
~/.claude/claude install
```

### Claude Code Not Found in PATH

After installation, if `claude` command is not found:

1. Ensure the install completed successfully
2. Restart your terminal or run `source ~/.bashrc` (or `source ~/.zshrc` for zsh)
3. Verify the binary exists: `ls -la ~/.claude/claude`
4. Manually add to PATH if needed: `export PATH="$HOME/.claude:$PATH"`

## GitHub Actions Issues

### API Key Not Working

If you see authentication errors:

1. Ensure your `CLAUDE_API_KEY` secret is set correctly in repository settings
2. Verify the API key has both Claude API and Claude Code access enabled
3. Check that the key hasn't expired or been revoked

### Timeout During Analysis

For large PRs, you may need to increase the timeout:

```yaml
- uses: anthropics/claude-code-security-review@main
  with:
    claude-api-key: ${{ secrets.CLAUDE_API_KEY }}
    claudecode-timeout: 30  # Increase from default 20 minutes
```

### Binary Files Causing Issues

The action gracefully handles binary files (images, compiled assets, etc.) in PRs. If you're seeing issues with binary file handling, ensure you're using the latest version of the action.
