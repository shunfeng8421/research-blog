# Two Sides of the Same Factory: What Building 292 Proteins and Hunting MCP Bugs Taught Me About Trust

*Shiqiang Chen — independent researcher (2026-10-04)*

I spent the last two weeks operating two very different "factories." One is an automated protein-design pipeline that has synthesized **292 candidate binders** and leveled a competition track to **20/20 submissions**. The other is a security-audit pipeline that has runtime-confirmed **4 vulnerabilities** in AI-agent "execution control" repositories and published **3 papers** plus **9 detection tools** to open source.

They look unrelated. But they converge on one lesson: **in any fast-moving, tool-driven field, the thing that names a guarantee is not the same as the guarantee.**

## The protein factory: fold score ≠ binding

Our design pipeline uses RFdiffusion (generate skeletons) → ProteinMPNN (design sequences) → ColabFold (verify folding). The naive success metric is **pLDDT** — the model's own confidence that a sequence folds. Our library is full of binders with pLDDT 92–97. Excellent folded proteins.

Then we ran them through **multimer ipTM** — AlphaFold's metric for *binding* to a target. The result was humbling: **pLDDT 92–97 single-chain collapsed to ipTM < 0.2** against the EGFR domain. High fold confidence said nothing about whether the protein would actually bind its target. Every binder that scored high on the "internal" metric (the model confidence in itself) failed the "external" metric (does the thing work with the thing it's meant to interact with?).

**The lesson:** a self-reported confidence metric is not evidence of function. You must validate against the real interaction, not against the metric the generator optimizes.

## The security factory: the name is not the guarantee

In parallel, I audited the AI-agent ecosystem — the repositories that are supposed to *control* execution: things named "gateway," "sandbox," "policy engine," "approval service." An operator deploying an "execution gateway" reasonably assumes it gates execution. That assumption is the attack.

I runtime-confirmed 4 repositories where the name advertised a security property that did not exist:
- **agent-exec-gateway**: named an execution gateway, shipped as unauthenticated arbitrary command execution + policy/approval bypass
- **agent-sandbox-runtime**: named a sandbox, shipped as unauthenticated arbitrary Python execution on 0.0.0.0
- **governed-policy-engine**: named for governance, shipped unauthenticated policy rewrite + approval self-grant
- **e2b-mcp-server**: hardcoded API key + unauthenticated services bound to 0.0.0.0

Meanwhile, a control group of 7 implementations enforced authentication and failed closed. The difference was never stars (both classes had 0-star members) — it was whether authentication was **wired to the executable surface**.

I named this pattern **zero-star trust inversion**: in the recent-0-star population, the name is the only assurance, and the name is wrong.

## The convergence

Both factories taught the same thing:

| Dimension | Protein design | Agent security |
|---|---|---|
| Cheap internal metric | pLDDT (fold confidence) | Repository name / star count |
| What actually matters | ipTM (binding to target) | Auth wired to execution surface |
| Failure mode | High pLDDT, no binding | Trusted name, no auth |
| Fix | Validate against target, not generator metric | Wire auth to every executable path, fail closed |

## What I'm publishing

This work is open, reproducible, and runtime-verified:

- **Papers** (3): `shunfeng8421/mcp-agent-security-papers` — execution control-planes, MCP trust labels, zero-star trust inversion
- **Toolkit** (9 detection tools): `shunfeng8421/mcp-security-toolkit` — read-only gate, SSRF, command injection, trust-label spoof scanners
- **Disclosures**: all 4 vulnerabilities reported to owning repos (all OPEN, runtime-verified)

No AI-generated fluff, no exaggerated claims. Every finding is backed by a reproduced runtime effect on an isolated loopback instance. Papers and code are cited/reproducible.

---

*The two factories keep running. The protein one is prepping for the next competition round; the security one is scanning the next wave of AI-tooling repositories. The takeaway, whether you're designing a binder or deploying a gateway: **validate against the thing that matters, not the metric that flatters.***