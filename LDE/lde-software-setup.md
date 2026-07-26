# LDE Software Setup — Design Primitives

A tool-agnostic pattern for provisioning any piece of software onto a developer machine. Each
primitive below is a stage in a pipeline; a concrete setup routine implements the stages for one
specific tool, but the contract of each stage never changes.

Throughout, `<tool>` stands for the software being installed and `<version>` for the version
constraint it must satisfy.

---

# 1. Idempotency (The Golden Rule)

Not a step itself, but the principle governing every other primitive. An idempotent operation
yields the same end state whether it runs once or a thousand times.

- Every stage must be safe to re-enter after a partial or failed run.
- Prefer converging on a desired state over blindly applying an action.
- Never append to a file without first checking whether the entry is already present.
- A re-run on an already-correct machine should perform no mutations and exit successfully.

# 2. Detection (State Check)

Before changing anything, assess the current state by querying the environment.

- Is `<tool>` resolvable on the current `$PATH`?
- Is it installed but shadowed, or present only inside a shell-specific initialization?
- Are its prerequisites (runtimes, system libraries, package managers) present?
- Record the answer as an explicit state — `absent`, `present`, or `present-but-unusable` — rather
  than a boolean, so later stages can branch precisely.

# 3. Resolution (Version Matching)

Presence is not sufficient; the version must satisfy the required constraint.

- **Version parsing** — invoke the tool's version command and parse the output into a comparable
  form. Assume the output format is tool-specific and may include noise.
- **Constraint checking** — compare the installed version against the requirement (exact pin,
  minimum, or range). If it satisfies the constraint, skip ahead to verification; otherwise proceed
  to provisioning.
- **User selection** — when the target version is not fixed by policy, ask the user which version to
  install and persist the answer to `~/.config/LDE/user.config.json` so subsequent runs are
  non-interactive.

# 4. Provisioning (Fetching and Installing)

If the tool is missing or fails the constraint, acquire and install it.

- Consult `~/.config/LDE/<tool>/setup.config.json` first for setup rules, restrictions, and forced or
  pinned versions. Policy in that file overrides both defaults and user preference.
- If the tool is present but outdated, prompt the user with the installed version and the available
  upgrade before mutating anything.

> Acquisition strategies, in rough order of preference:
- **System package manager** — delegate to the platform's own installer so upgrades and removal stay
  under one owner.
- **Version manager** — install through a per-user tool that supports multiple concurrent versions,
  when the ecosystem provides one and side-by-side versions are needed.
- **Direct fetch** — download a binary, archive, or installer script from an official source. Pin the
  source to a specific release and verify integrity (checksum or signature) before executing.
- **Unpack and link** — extract the artifact into a stable install root and symlink its executables
  into a directory already on the user's `$PATH`.

# 5. Configuration (Environment Hooking)

Software is rarely usable the moment the binary lands on disk; it must be hooked into the user's
environment.

- **Path and shell injection** — add the tool's directories and any required environment variables to
  the user's shell initialization files. Write the block once, guarded by a marker or a
  presence check, so repeat runs do not duplicate it.
- **Shell coverage** — apply the change to the shells the user actually runs, not just the one
  currently active.
- **Configuration files** — generate or amend the tool's own settings files (registries, proxies,
  credentials, defaults). Merge into existing files rather than overwriting them, and back up
  anything replaced.
- **Current session** — either apply the configuration to the running shell or tell the user
  explicitly that a new shell is required.

# 6. Verification (Health Check)

The final primitive confirms the install actually worked and is operable in the current context.

- **Execution test** — run the tool with a harmless, side-effect-free command to prove it starts and
  loads its dependencies.
- **Version assertion** — re-read the reported version and confirm it satisfies the constraint from
  stage 3. This closes the loop and catches installs that landed but were shadowed by an older copy.
- **Exit code validation** — require a `0` exit status; treat any non-zero result as a failed setup,
  with the captured output surfaced to the user.
- **Fail loudly** — a setup that cannot verify itself must report failure rather than assume success.
