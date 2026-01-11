# Copilot Coding Agent Onboarding

This repository provides a **composite GitHub Action that installs and configures Node.js and pnpm** on GitHub-hosted Ubuntu runners. It is an infrastructure utility for Node projects, not an application library.

---

## High‑Level Details

- **Type:** GitHub Action (composite)
- **Primary entry point:** `action.yml`
- **Languages:** YAML and Bash
- **Target runtime:** `ubuntu-latest` (Ubuntu 24.04 images)
- **What it does:** Downloads a specific Node.js tarball, extracts it into `/usr/local`, updates `PATH`, enables Corepack, and activates a specific pnpm version. Optionally installs project deps via `pnpm install` (consumer repo).
- **Configurable inputs:**
  - `node-version` (default: `24.9.0`)
  - `pnpm-version` (default: `10.18.2`)

**Repository footprint:** Small; one composite action plus several CI workflows (labeling, PR title lint, auto-approve, enqueue to Merge Queue).

---

## Build & Validation Instructions

> CI validation for this repo is workflow‑based. There is no compile step. Follow the sequences below to avoid bash failures and broken pipelines.

### Bootstrap (local reproduction on Ubuntu)
Use these steps to emulate the Action locally:
```bash
# 1) Choose versions (use valid released versions)
NODE_VERSION=24.9.0
PNPM_VERSION=10.18.2

# 2) Install Node exactly like the Action
curl -fsSL "https://nodejs.org/dist/v${NODE_VERSION}/node-v${NODE_VERSION}-linux-x64.tar.xz" -o "node-v${NODE_VERSION}-linux-x64.tar.xz"
sudo tar -xJf "node-v${NODE_VERSION}-linux-x64.tar.xz" -C /usr/local
sudo chown -R "$(whoami)" "/usr/local/node-v${NODE_VERSION}-linux-x64"
rm -f "node-v${NODE_VERSION}-linux-x64.tar.xz"
export PATH="/usr/local/node-v${NODE_VERSION}-linux-x64/bin:${PATH}"

# 3) Enable pnpm via Corepack
corepack enable
corepack prepare "pnpm@${PNPM_VERSION}" --activate

# 4) Optional: install deps in the consumer repository
test -f package.json && pnpm install || echo "No package.json; skipping pnpm install"
```

**Preconditions:** network access; `sudo` available; valid Node release for Linux x64.  
**Postconditions:** `node`, `npm`, `corepack`, and `pnpm` are on `PATH`; `pnpm install` succeeds when a `package.json` is present.

### What to always do
- **Always** set `node-version` and `pnpm-version` to real released versions.
- **Always** run `corepack enable` before preparing pnpm.
- **Always** ensure `PATH` includes `/usr/local/node-v<version>-linux-x64/bin` before running Node‑based commands.
- **If the repo has no `package.json`,** skip `pnpm install` (the Action runs it; consumers should ensure a Node project context).

### Common errors and mitigations
- **404 on Node tarball:** Version string invalid. Use a released `x.y.z`.  
- **Permission denied under `/usr/local`:** Use a runner with `sudo` (GitHub-hosted runners support this).  
- **`pnpm` not found:** Run `corepack enable` + `corepack prepare pnpm@<ver> --activate` again and re-check `PATH`.  
- **Install step fails:** Ensure the consumer repo has `package.json` and a working registry auth if needed.

### CI timing notes
- Node download/extract is typically <30s on GitHub runners. Network slowness can extend this; `curl -fsSL` will fail fast on HTTP errors.

---

## Project Layout & CI Gates

```
action.yml                   # Composite action: install Node, enable Corepack, activate pnpm, pnpm install
.github/workflows/
  build.yml                  # "Build" workflow (no-op pass) used as a required check
  auto-queue.yml             # Enqueue successful PRs into Merge Queue after Build
  auto-approve.yml           # Auto-approve for trusted authors/labels
  pr-labeler.yml             # Automatic labeling for core/auto-merge
  pull-request-lint.yml      # Conventional-commit PR title enforcement
.vscode/settings.json        # Editor/YAML defaults
LICENSE
README.md                    # Short description
```

### Checks run before merge (and how to replicate)
- **Build / build:** no-op success check. Run automatically on `pull_request` and via manual dispatch.
- **PR title lint:** requires one of the allowed types (`chore|ci|docs|feat|fix|major|perf|refactor|revert|style|test`).
- **Labeler & Auto-approve:** labels and approvals applied for specific authors or labels.
- **Auto-queue:** after **Build** completes successfully for a PR, enqueues that PR into Merge Queue.

**Replication:**
- Create a test PR; confirm **Build** passes.  
- Ensure branch protection lists **Build / build (pull_request)** as a required status check if you use Merge Queue.  
- The auto-queue workflow listens for the workflow named **"Build"**; keep that name stable or update the listener if you change it.

### Dependencies not obvious from the layout
- The action expects a **Node project** when running `pnpm install`. Consumer repositories without `package.json` will fail at that step.  
- Corepack is bundled with recent Node releases; enabling + preparing is required for deterministic pnpm versions.  
- The action assumes **Linux x64** runners.

---

## Usage (in a consumer workflow)

```yaml
- uses: p6m7g8-actions/node-setup@<ref>
  with:
    node-version: "24.9.0"
    pnpm-version: "10.18.2"
```

After this step, `node`, `npm`, and `pnpm` are available. Add `pnpm install`, `pnpm test`, or build steps as needed for the consumer repository.

---

## Agent Rules

- Treat this repo as a **Node toolchain bootstrap action**. Do not add application code here.
- Keep `action.yml` step order intact: *install Node → enable corepack → prepare pnpm → verify → optional install*.
- Maintain input names and defaults for backward compatibility.
- Keep the workflow named **"Build"** unless you update listeners and protections accordingly.
- Trust this document. Only search the repo if something here is incomplete or demonstrably wrong.
