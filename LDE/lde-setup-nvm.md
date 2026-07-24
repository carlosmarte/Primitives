# NVM Setup Skill: Automated Provisioning & Configuration

This document defines the automated setup skill for **NVM (Node Version Manager)**, adhering strictly to the requested primitives: Idempotency, Detection, Resolution, Provisioning, Configuration, and Verification.

## 1. Idempotency (The Golden Rule)
The provided bash script is designed to safely execute multiple times. It checks for existing configurations in `~/.bashrc` and `~/.zshrc` before appending initialization strings, and it validates the existing `$NVM_DIR` before attempting to re-download the installation script.

## 2. Detection (State Check)
The system checks if NVM is already installed by looking for the `nvm.sh` script in the expected directory (`$HOME/.nvm`). If present, it sources it and queries the current version.

## 3. Resolution (Version Matching)
The resolution phase leverages the local configuration directory (`~/.config/LDE/`).
- It checks `~/.config/LDE/nvm/setup.config.json` for any fixed or forced setup rules.
- If no fixed version is specified, it prompts the user to select their desired Node version.
- The user's selection is saved to `~/.config/LDE/user.config.json` using `jq` for future idempotency.

## 4. Provisioning (Fetching and Installing)
If NVM is missing, the script fetches the installation payload directly from the remote source:
`curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/master/install.sh | bash`.
It then proceeds to install the user's targeted Node.js version using `nvm install`.

## 5. Configuration (Environment Hooking)
Software must be hooked into the user's terminal environment. This skill automatically injects the `$NVM_DIR` and bash completion hooks into both `.bashrc` and `.zshrc`. It uses `grep` to ensure this only happens once.

## 6. Verification (Health Check)
After installation and configuration, the script sources the environment and runs an execution test on both `nvm --version` and `node --version`. It relies on proper exit code validation to confirm the system is healthy.

---

## Executable Bash Skill (`setup_nvm.sh`)

```bash
#!/usr/bin/env bash
set -e # Exit on error

# Configuration Paths
LDE_USER_CONFIG="$HOME/.config/LDE/user.config.json"
LDE_NVM_CONFIG="$HOME/.config/LDE/nvm/setup.config.json"
NVM_INSTALL_SCRIPT="[https://raw.githubusercontent.com/nvm-sh/nvm/master/install.sh](https://raw.githubusercontent.com/nvm-sh/nvm/master/install.sh)"
export NVM_DIR="$HOME/.nvm"

# Ensure config directories exist
mkdir -p "$(dirname "$LDE_USER_CONFIG")"
mkdir -p "$(dirname "$LDE_NVM_CONFIG")"

# ---------------------------------------------------------
# [2] Detection (State Check) & [1] Idempotency
# ---------------------------------------------------------
echo "==> [Detection] Checking current NVM state..."
NEEDS_INSTALL=true

if [ -s "$NVM_DIR/nvm.sh" ]; then
    echo "    NVM is already installed at $NVM_DIR."
    # Source NVM to check version
    \. "$NVM_DIR/nvm.sh"
    CURRENT_NVM_VERSION=$(nvm --version)
    echo "    Current NVM version: $CURRENT_NVM_VERSION"
    NEEDS_INSTALL=false
else
    echo "    NVM is not installed."
fi

# ---------------------------------------------------------
# [3] Resolution (Version Matching)
# ---------------------------------------------------------
echo "==> [Resolution] Resolving version constraints..."
TARGET_NODE_VERSION=""
FIXED_VERSION=""

# Check if jq is installed for JSON parsing
if command -v jq &> /dev/null && [ -f "$LDE_NVM_CONFIG" ]; then
    FIXED_VERSION=$(jq -r '.fixed_version // empty' "$LDE_NVM_CONFIG")
fi

if [ -n "$FIXED_VERSION" ]; then
    echo "    Found forced/fixed Node version in setup.config.json: $FIXED_VERSION"
    TARGET_NODE_VERSION="$FIXED_VERSION"
else
    # Prompt user if outdated or not fixed
    read -p "    Which version of Node.js should be installed? (e.g., '20', 'lts/*', 'node'): " TARGET_NODE_VERSION
    TARGET_NODE_VERSION=${TARGET_NODE_VERSION:-"lts/*"}
    
    # Save user selection
    if command -v jq &> /dev/null; then
        if [ ! -f "$LDE_USER_CONFIG" ]; then echo "{}" > "$LDE_USER_CONFIG"; fi
        TMP_CONFIG=$(mktemp)
        jq --arg nv "$TARGET_NODE_VERSION" '.nvm_node_version = $nv' "$LDE_USER_CONFIG" > "$TMP_CONFIG"
        mv "$TMP_CONFIG" "$LDE_USER_CONFIG"
        echo "    Saved user selection ($TARGET_NODE_VERSION) to$LDE_USER_CONFIG."
    else
        echo "    'jq' not installed. Skipping save to user.config.json."
    fi
fi

# ---------------------------------------------------------
# [4] Provisioning (Fetching and Installing)
# ---------------------------------------------------------
if [ "$NEEDS_INSTALL" = true ]; then
    echo "==> [Provisioning] Fetching and installing NVM core..."
    curl -o- "$NVM_INSTALL_SCRIPT" | bash
    
    # Load newly installed NVM into the current session
    export NVM_DIR="$HOME/.nvm"
    [ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
else
    echo "==> [Provisioning] Skipping NVM core installation (Idempotent)."
fi

echo "==> [Provisioning] Installing Node version: $TARGET_NODE_VERSION..."
nvm install "$TARGET_NODE_VERSION"
nvm alias default "$TARGET_NODE_VERSION"
nvm use default

# ---------------------------------------------------------
# [5] Configuration (Environment Hooking)
# ---------------------------------------------------------
echo "==> [Configuration] Hooking NVM into environment files..."

HOOK_BLOCK='
# NVM Configuration
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"  # This loads nvm
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"  # This loads nvm bash_completion'

for RC_FILE in "$HOME/.bashrc" "$HOME/.zshrc"; do
    if [ -f "$RC_FILE" ]; then
        # Check if the hook already exists to ensure Idempotency
        if ! grep -qc 'NVM_DIR="$HOME/.nvm"' "$RC_FILE"; then
            echo "$HOOK_BLOCK" >> "$RC_FILE"
            echo "    ✅ Injected NVM hooks into $RC_FILE"
        else
            echo "    ⚡ NVM hooks already exist in $RC_FILE (Skipping)."
        fi
    fi
done

# ---------------------------------------------------------
# [6] Verification (Health Check)
# ---------------------------------------------------------
echo "==> [Verification] Running health checks..."

if nvm --version > /dev/null 2>&1; then
    echo "    ✅ NVM is operable. Version: $(nvm --version)"
else
    echo "    ❌ NVM verification failed (exit code non-zero)."
    exit 1
fi

if node --version > /dev/null 2>&1; then
    echo "    ✅ Node is operable. Version: $(node --version)"
else
    echo "    ❌ Node verification failed (exit code non-zero)."
    exit 1
fi

echo "==> 🎉 Setup Complete!
```
