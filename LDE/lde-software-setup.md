1. Idempotency (The Golden Rule)
While not a step itself, idempotency is the core principle governing all setup primitives. An idempotent operation yields the exact same result whether you run it once or a thousand times.

2. Detection (State Check)
Before changing anything, the system must assess the current state. This involves querying the environment to see if the tool or its dependencies exist.

3. Resolution (Version Matching)
Simply having the software isn't enough; you need the correct version. you may have to ask user which version should be installed. have save user selection to ~/.config/LDE/user.config.json
- Version parsing: Running node --version and parsing the output.
- Constraint checking: Comparing the installed version against the required version (e.g., ensuring Node is >= 18.0.0). If the version matches, the setup skips to verification. If it's outdated, it proceeds to provisioning.

4. Provisioning (Fetching and Installing)
If the tool is missing or outdated, the system must acquire and install it. Prompt user if software is outdated and what version is available. Ensure to check ~/.config/LDE/{software name}/setup.config.json for setup rules, restriction, forced or fixed version.
> Examples:
- Package Managers: Handing off the request to a system-level tool (e.g., brew install node, apt-get install npm).
- Direct Fetch: Downloading a binary or installation script directly from a remote source (e.g., curl -o- [https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh](https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh) | bash).
- Unpacking/Linking: Extracting tarballs and symlinking binaries to a location in the user's $PATH (like /usr/local/bin).

5. Configuration (Environment Hooking)
Software is rarely useful immediately after the binary hits the disk. It needs to be hooked into the user's environment.

> Examples:
- Path injection: Modifying .bashrc, .zshrc, or .profile so the terminal knows where to find the tool (e.g., export NVM_DIR="$HOME/.nvm").
- Configuration files: Generating or modifying default settings files (like a .npmrc file for registry settings).

6. Verification (Health Check)
The final primitive ensures the installation was actually successful and the tool is operable in the current context.

Execution test: Running the tool with a harmless flag to ensure it doesn't instantly crash (e.g., npm whoami or node -e "console.log('working')").

Exit code validation: Ensuring the command returns a 0 (success) rather than throwing an error.
