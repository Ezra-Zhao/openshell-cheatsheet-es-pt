# 🛡️ OpenShell — one-page cheatsheet (English)

> Commands verified against the [official docs (v0.1.3)](https://docs.nvidia.com/openshell/latest/index.html) and the [NVIDIA/OpenShell](https://github.com/NVIDIA/OpenShell) README on 2026-10-12.
> What it is: a policy-driven sandbox runtime for AI agents. You declare in a policy what each agent may touch — files, network, credentials — and OpenShell enforces it at the kernel level.

---

## 0. Install and first sandbox

Requirements: Linux, macOS on Apple Silicon, or Windows with WSL 2 (experimental), plus Docker, Podman, or host virtualization.

```shell
curl -LsSf https://raw.githubusercontent.com/NVIDIA/OpenShell/main/install.sh | sh
openshell sandbox create --name demo
```

The installer sets up the CLI and a local gateway. The default sandbox image is a minimal Ubuntu with no agent installed.

---

## 1. Core commands

**Sandbox lifecycle**

```shell
openshell sandbox create --name my-sandbox --policy ./policy.yaml  # create with a policy
openshell sandbox list                                             # list sandboxes
openshell sandbox get my-sandbox                                   # inspect one
openshell sandbox connect my-sandbox                               # open a session
openshell sandbox exec -n my-sandbox -- ls -la /workspace          # run a command inside
openshell sandbox stop my-sandbox / start my-sandbox               # stop / start
openshell sandbox delete my-sandbox                                # delete
openshell logs my-sandbox                                          # view logs
```

**Policies**

```shell
openshell policy get my-sandbox --base        # the policy you own (base)
openshell policy get my-sandbox --full        # base + provider rules (effective)
openshell policy list my-sandbox              # policy revisions
openshell policy set my-sandbox --policy policy.yaml --wait   # replace the whole policy
```

Add or remove network access on a running sandbox — no full policy file needed:

```shell
# Let curl in the sandbox make read-only requests to the GitHub API
openshell policy update my-sandbox \
  --rule-name github_readonly \
  --binary /usr/bin/curl \
  --add-endpoint api.github.com:443:read-only:rest:enforce \
  --wait

openshell policy update my-sandbox --remove-rule github_readonly --wait
```

Endpoint format: `host:port:preset:protocol:enforcement`. `read-only` allows GET, HEAD and OPTIONS; `rest` turns on HTTP request inspection; `enforce` blocks everything else. Use `--dry-run` instead of `--wait` to preview a change.

**Check a policy before applying it (standalone prover)**

```shell
openshell-prover check candidate.yaml --boundary boundary.yaml
```

The prover uses formal verification: it checks that a candidate policy grants no more access than the boundary you define.

---

## 2. The policy file

YAML, starts with `version: 1`. Anything the policy does not allow is denied.

| Section | Controls | Enforced by | Takes effect |
|---|---|---|---|
| `filesystem_policy` | Paths processes can read, or read and write | Landlock LSM in the kernel | At sandbox startup |
| `landlock` | Whether the sandbox still starts if filesystem rules cannot be applied | Sandbox runtime | At sandbox startup |
| `process` | User and group the processes run as | Docker / Podman at creation | At sandbox creation |
| `network_policies` | Which destinations each binary can reach, and which requests it can send | Sandbox network proxy | While the sandbox runs |
| `network_middlewares` | Extra inspection or blocking of allowed traffic | Sandbox network proxy | While the sandbox runs |

- Every sandbox runs under a policy, even if you supply none: OpenShell falls back to a restrictive default policy.
- Where the policy comes from, in order: a global policy from the gateway admin → the sandbox's saved policy (`--policy` at creation, or the `OPENSHELL_SANDBOX_POLICY` env var) → a policy inside the image at `/etc/openshell/policy.yaml` → the default policy.
- Network rules can change while the sandbox runs. Filesystem and process rules are fixed once it starts: changing them means recreating the sandbox.

---

## 3. Architecture in three pieces

- **Sandbox** — lives inside the boundary, next to the agent. It owns the agent's processes, knows which program made each request, and forwards TCP and DNS traffic to the supervisor. Its only allowed network egress is the protected channel to the supervisor.
- **Supervisor** — lives on the trusted side of the boundary. It checks every request against policy, supplies credentials, resolves DNS, and opens the approved connections.
- **Gateway** — the control plane. It manages the sandbox lifecycle, delivers policy and settings, attaches providers, checks who you are, and runs the policy prover.
- **Providers** — connect a service name to a stored credential. The supervisor hands the credential out only where policy allows it.

---

## 4. Security model

- **Kernel-level enforcement.** Every file access, system call and network connection is checked at runtime.
- **Agents never see real credentials.** OpenShell adds credentials only to requests bound for approved endpoints. A leaked prompt or a curious agent cannot exfiltrate a key it never had.
- **Deny by default.** Every outbound connection is denied unless a `network_policies` rule allows it — down to which binary may send which request (read from an API, but not write to it).
- **Risky changes wait for a human.** Before a policy change is approved, formal verification flags risky new access — reaching a new host with credentials, calling a new API method, touching cloud metadata endpoints. Any finding blocks auto-approval.

---

## 5. End to end: run an agent, approve access as it needs it

This flow runs OpenCode against a free OpenRouter model (from the official "Run Your First Agent" guide).

**1. Import a provider profile and create the provider** — set `OPENROUTER_API_KEY` first; OpenShell stores it and sends it only to `openrouter.ai`:

```shell
openshell profile import \
  --url https://raw.githubusercontent.com/NVIDIA/OpenShell/main/providers/openrouter.yaml

openshell provider create \
  --name openrouter \
  --type openrouter \
  --from-existing
```

**2. Create the sandbox with the agent as its main process:**

```shell
openshell sandbox create \
  --name my-agent \
  --from ghcr.io/anomalyco/opencode:latest \
  --provider openrouter \
  -- opencode -m openrouter/nvidia/nemotron-3.5-lightning:free
```

**3. Approve new access when the agent asks for it.** When the agent hits a destination the policy does not allow, OpenShell denies the request and drafts a narrow rule (the policy advisor). The gateway checks the draft with the prover; you decide:

```shell
openshell rule get my-agent --status pending                       # list pending proposals
openshell rule approve my-agent --chunk-id <chunk-id>               # approve one
openshell rule reject my-agent --chunk-id <chunk-id> \
  --reason "Not needed for this task."                              # reject one, with a reason
openshell rule approve-all my-agent                                # approve all pending
```

Approved rules hot-reload into the running sandbox — no restart, the agent just retries.

---

## 6. Extras

- **Teach your coding agent OpenShell:** `npx skills add NVIDIA/OpenShell` — skills for driving the CLI, writing policies and debugging gateways.
- **SDKs** (they connect to a gateway; they do not install the CLI): Python `uv add openshell` · TypeScript `npm install @nvidia/openshell-sdk` · Go `go get github.com/NVIDIA/OpenShell/sdk/go@latest` · Rust `cargo add openshell-sdk`.
- **Test the gateway API:** it serves gRPC reflection on `localhost:18080` — try `grpcurl -plaintext localhost:18080 list`.

---

*Docs: https://docs.nvidia.com/openshell/latest/index.html · Repo: https://github.com/NVIDIA/OpenShell · Questions: GitHub Discussions.*
