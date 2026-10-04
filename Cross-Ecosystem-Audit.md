# Where the Vulnerabilities Actually Live: A Cross-Ecosystem Audit of AI-Agent Security

*Shiqiang Chen — independent researcher (2026-10-04)*

I've spent months auditing the AI-agent security ecosystem across three distinct surfaces: Model Context Protocol (MCP) servers, execution-control gateways, and AI-agent × DeFi wallets. The pattern across all three is consistent and, I think, useful to anyone building or deploying agent tooling: **the language and maturity of a repository predict its vulnerability rate better than its name does.**

This is a companion to my [papers](https://github.com/shunfeng8421/mcp-agent-security-papers) and [detection toolkit](https://github.com/shunfeng8421/mcp-security-toolkit).

## Three audits, three different density of findings

### 1. Python MCP / execution-control servers — 4 confirmed vulnerabilities

In the Python ecosystem (62+ freshly-published repos), I runtime-confirmed **4 vulnerabilities**, all in the "execution control" class — components named *gateway*, *sandbox*, *policy engine*:

- **agent-exec-gateway**: unauthenticated arbitrary command execution + policy/approval bypass
- **agent-sandbox-runtime**: unauthenticated arbitrary Python execution on 0.0.0.0
- **governed-policy-engine**: unauthenticated policy rewrite + approval self-grant
- **e2b-mcp-server**: hardcoded API key + unauthenticated services bound to 0.0.0.0

Characteristic of Python 0-star repos: FastAPI/FastMCP with no `Depends`, `shell=True` + f-string command building, hardcoded secrets, `0.0.0.0` bind in the own Dockerfile. The thing named "sandbox" ships as a reachable code-execution primitive.

### 2. Rust/Go MCP servers — 0 vulnerabilities (deep-audited 8)

I deep-audited skgate (Go OAuth MCP gateway), fastexec-mcp (Rust bash-exec), esxi-mcp (Go), pydeno (Rust V8 sandbox), and others. **Zero exploitable vulnerabilities.**

Why:
- Static typing + `Command::new().args()` (argv list, no shell parsing) make command injection structurally hard.
- Authentication is taken seriously: skgate implements textbook OAuth (PKCE S256, single-use codes, constant-time compare, redirect strict-validate).
- Engineering culture: SECURITY.md, CI, acceptance tests are common.

### 3. AI-agent × DeFi wallets — 0 easy wins (deep-audited new projects)

I applied the 8-vector AI-Agent × DeFi methodology (tool injection, cross-contract chains, oracle poisoning, MCP MITM, timing, multi-agent, context poisoning, autonomous-signature theft) to new projects like Ink-OmniVault-AI (ERC-4337 smart account) and signo-shield.

Ink-OmniVault-AI's `_execute` enforces: whitelisted targets, enabled modules, session spending limits, a circuit breaker (pauses after consecutive failures), and per-agent policies. This is a genuinely hardened design — the simple vulnerabilities (drain, no checks) have been engineered out.

## The insight: trust the engineering, not the label

The honest takeaway across all three surfaces is that *the density of easy-to-find vulnerabilities tracks the language/engineering culture, not the security-soundingness of the name*:

| Surface | Easy vulns | Why |
|---|---|---|
| Python 0-star MCP/gateway | High (4 confirmed) | Fast prototyping, `shell=True`, hardcoded secrets, no auth by default |
| Rust/Go MCP | Nil | Static types, argv-list exec, auth taken seriously |
| AI-Agent DeFi (new) | Nil | Hardened designs: whitelists, session limits, circuit breakers |

A repository named "sandbox" in Python is more likely to be an unauthenticated exec primitive than a Rust binary with the same name. A well-funded AI wallet (ERC-4337) is hardened; a 0-star Python "gateway" is not.

This mirrors what I found in the protein-design world: **a self-reported confidence metric (pLDDT / repository name) is not evidence of the property it claims (binding / security).** Validate against the real gate, not the flattering metric.

## What this means for defenders

1. **If you deploy agent tooling, treat Python 0-star "control" components as untrusted bounds** — verify auth is wired to every executable endpoint, and never expose on `0.0.0.0`.
2. **If you are securing, prioritize Python MCP/gateway repos** over Rust/Go — that's where the density is.
3. **If you are building**, adopt the Rust/Go pattern even in Python: `subprocess` argv lists (never `shell=True`), fail-closed auth, no hardcoded secrets, bind loopback by default.

All findings are runtime-confirmed on isolated loopback instances (no credentials, no real assets), disclosed to owning repos, and reproducible. Scanner + paper + toolkit are open-source.

*Next rounds: the paper is being extended with this cross-language density result; the detection toolkit is being updated to prioritize Python "control-plane" naming classes.*