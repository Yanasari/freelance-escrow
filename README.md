# FreelanceEscrow — AI-Verified Milestone Escrow for Freelance Work

Freelance marketplaces run on trust: a client does not want to pay before the work exists, and a freelancer does not want to work before payment is credible. Classic escrow solves this with a human middleman who inspects the delivery and releases the money. FreelanceEscrow replaces the middleman's *review* step with GenLayer's non-deterministic consensus, and records the whole agreement — job, milestones, proofs, verdicts, releases — as public, append-only on-chain state.

## How it works

1. **Create a job.** The client calls `create_job(title, freelancer, currency, milestones_json)`. Each milestone has a title, a written requirement, and an amount. A job holds 1–10 milestones.
2. **Deliver.** The freelancer calls `submit_proof(job_id, milestone_index, proof_url, summary)` with a public proof page — a repository, a published page, a live demo.
3. **Verify by consensus.** Anyone calls `verify_milestone(job_id, milestone_index)`. A leader validator fetches the proof page (`gl.nondet.web.render`) and an LLM judges whether the page demonstrates the milestone requirement (`gl.nondet.exec_prompt`, JSON response). Every other validator independently repeats the fetch and the judgement and accepts the leader's result only if its own verdict matches it exactly (`gl.vm.run_nondet_unsafe` with a custom validator) — GenLayer's equivalence principle applied to the verdict (`COMPLETED` / `INCOMPLETE`), never to the reasoning text. The verdict and the decisive evidence are stored on the milestone.
4. **Release.** Once a milestone is `VERIFIED`, the client calls `release_milestone(job_id, milestone_index)`, which records the release authorization on-chain. When every milestone is released, the job becomes `COMPLETED`.

Milestone lifecycle: `AWAITING_PROOF → PROOF_SUBMITTED → VERIFIED / REJECTED → RELEASED` (a rejected milestone can receive a new proof). Job lifecycle: `OPEN → COMPLETED`.

## Methods

| Method | Type | Who | Purpose |
| --- | --- | --- | --- |
| `create_job(title, freelancer, currency, milestones_json)` | write | client | Create a job with 1–10 milestones |
| `submit_proof(job_id, milestone_index, proof_url, summary)` | write | freelancer | Deliver a milestone with a public proof URL |
| `verify_milestone(job_id, milestone_index)` | write (non-det.) | anyone | AI-consensus judgement of the submitted proof |
| `release_milestone(job_id, milestone_index)` | write | client | Record the release of a verified milestone |
| `get_job(job_id)` / `get_milestone(job_id, i)` | view | anyone | Read a job (with milestones) or one milestone |
| `get_jobs()` / `get_history()` / `get_config()` | view | anyone | List jobs, the full event log, contract info |

## Live demo (GenLayer Studio / Studionet)

- Contract: `0xb1CEF7B50f926e2DA63757C0CC54dc8f3DEf6398`
- Explorer: https://explorer-studio.genlayer.com/address/0xb1CEF7B50f926e2DA63757C0CC54dc8f3DEf6398
- Deployed from: `0x0d1e8B737614bCb563A4BaB329bC257D200B42B4` (creator verified on the explorer)
- Deploy tx: `0x59bd6dad94b181d8a18103fef292a7c175463d9a8c48175b000f20afc571bed3` — FINALIZED, 2026-11-26

Demo run — job `job-1`, "Demo job: GenLayer price-oracle repository" (client = deployer wallet above; freelancer = a Studio local account, `0x8590e584E02D17D22B58D2f54f6E13f8089934eb`; proof for both milestones: https://github.com/Yanasari/hargapangan). Every step FINALIZED:

| Step | Tx | Result |
| --- | --- | --- |
| `create_job` (2 milestones: 60 + 40 USDC) | `0x6d8730d5f457452444fb85a89062aaa4c4c26c8452adfaf7d408879f3a23712f` | job-1 OPEN |
| `submit_proof` milestone 0 | `0xa691a485043217abb167d00f497817f9210cb5dafa10ea675898e9ec9e345e1c` | PROOF_SUBMITTED |
| `submit_proof` milestone 1 | `0x7f7101bc0051063a11f570d0e1a598dc38d4aeeba7d3bff4a8eaca28f3e501e4` | PROOF_SUBMITTED |
| `verify_milestone` milestone 0 | `0x6ad9d7b59ee48d989417c857dd2ca491a4a7ee261322bdf837fce3193fe9ac1e` | verdict **COMPLETED** → VERIFIED |
| `verify_milestone` milestone 1 | `0xb6da4d92bded20c3dadffcfdb372e27fcf616ad2f72e06678210ea4e85b66aad` | verdict **COMPLETED** → VERIFIED |
| `release_milestone` milestone 0 | `0x8b63d7dc72cb79697a8e18e096f3f2b473269bf40b10bff5a0faf0d538594fb6` | RELEASED (60 USDC) |
| `release_milestone` milestone 1 | `0x008cf14e057374aba6554135ab24fe41ab3716bcdd082d4ed4afea0e6b32eeeb` | RELEASED (40 USDC) → job **COMPLETED** |

On-chain evidence recorded by the validators — milestone 0: "The public GitHub repository 'hargapangan' contains the required 'harga_pangan.py' file and documentation describing it as a GenLayer Intelligent Contract price oracle." Milestone 1: "Public GitHub repo Yanasari/hargapangan lists README.md and harga_pangan.py. README explains the contract (GenLayer oracle for food prices), a 'How it works' section, and a 'Contract API' table of methods." `get_history()` shows all 7 events.

## Scope — stated honestly

Version 1 is the record and verification layer. It records jobs, milestones, proofs, AI-consensus verdicts and release authorizations in a public history. It does **not** custody tokens: moving funds against a `VERIFIED` / `RELEASED` record is the documented next step (token escrow), exactly like a paper escrow ledger precedes the vault.

## Files

- `freelance_escrow.py` — the Intelligent Contract (GenLayer v0.2.16 header, `py-genlayer` 1jb45aa8…). No constructor arguments.
