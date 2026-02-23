---
name: add-apilens
description: >-
  Trigger when the user wants the agent to discover and use npm library
  APIs, or execute TypeScript code in a sandboxed environment. This skill
  installs apilens and user-chosen npm libraries inside the container so
  the agent can read type definitions and run code against real library
  APIs.
---

# Add apilens – npm library discovery & sandboxed execution

apilens teaches AI agents to use npm libraries by generating reference
files from package type definitions and providing a sandboxed TypeScript
execution environment.

Everything runs inside the container. No host-side changes are needed.

## Phase 1 · Pre-flight

1. Check if apilens is already set up:
   ```bash
   test -f /workspace/group/.apilens.yaml && echo "EXISTS" || echo "MISSING"
   ```
   If it exists, inform the user that apilens is already configured and
   stop.

2. Verify npm is available:
   ```bash
   npm --version
   ```

## Phase 2 · Configure libraries

First, check if the project already ships an `.apilens.yaml`:

```bash
test -f /workspace/project/.apilens.yaml && echo "FOUND" || echo "NOT FOUND"
```

### If the config exists in the project

Copy it into the writable workspace and skip the interactive steps:

```bash
cp /workspace/project/.apilens.yaml /workspace/group/.apilens.yaml
```

### If no config exists — interactive setup

Use `AskUserQuestion` to build the library list. Collect entries and
loop until the user is done.

For each library, ask:

1. **npm package name** – the exact name from the npm registry
   (e.g. `@kubernetes/client-node`, `pg`, `axios`).
2. **Title** – a short human-readable label
   (e.g. "Kubernetes API client").
3. **Description** _(optional)_ – a longer explanation of what the
   library is used for.

After each library, ask:

> "Add another library?"
>   - Yes
>   - No, done adding libraries

Repeat until the user selects **No**.

At least one library is required. If the user tries to proceed with
zero libraries, ask again.

Write `/workspace/group/.apilens.yaml`:

```yaml
libraries:
  - name: "<package-name>"
    title: "<title>"
    description: "<description>"
  # one entry per library
```

Omit the `description` key for any library where the user left it blank.

## Phase 3 · Install and configure

All commands run from `/workspace/group`.

### 3b  Run apilens setup

```bash
cd /workspace/group
npx apilens setup --config .apilens.yaml
```

This single command handles everything:
- Initialises `package.json` if missing
- Installs all configured libraries via npm
- Generates skill and reference files under
  `/workspace/group/.claude/skills/apilens/`

## Phase 4 · Verify

1. Confirm the skill file was generated:
   ```bash
   cat /workspace/group/.claude/skills/apilens/SKILL.md
   ```
   It should list every configured library.

2. Smoke-test sandboxed execution with the first library:
   ```bash
   npx apilens exec - <<'SCRIPT'
   const lib = require("<first-library-name>");
   console.log("OK:", typeof lib);
   SCRIPT
   ```

3. If the test prints `OK: object` (or `OK: function`), report success
   to the user. If it fails, read the error, fix it, and re-run — do
   not tell the user to fix it themselves.

## Usage after setup

Once installed, the agent can:

- **Read API types**: Follow the reference files in
  `/workspace/group/.claude/skills/apilens/references/` to discover
  library APIs by reading their `.d.ts` type definitions.

- **Execute code**: Run TypeScript against installed libraries:
  ```bash
  npx apilens exec - <<'SCRIPT'
  const k8s = require("@kubernetes/client-node");
  const kc = new k8s.KubeConfig();
  kc.loadFromDefault();
  console.log(kc.contexts);
  SCRIPT
  ```

- **All state lives in `/workspace/group/`** — `node_modules`,
  `.apilens.yaml`, and generated skill files persist across container
  restarts since this directory is bind-mounted from the host.
