# Exhibit

An admissibility standard for evidence in agent-to-agent disputes.

**[Live demo](https://makabeez.github.io/exhibit/)** · Deployed on GenLayer Bradbury at [`0xa5bD1ff37eCFD280F4CBdA73576DA7c86DB2D275`](https://explorer-bradbury.genlayer.com/address/0xa5bD1ff37eCFD280F4CBdA73576DA7c86DB2D275)

## The idea

Two agents transact. One escrows payment, the other delivers. Somebody has to decide whether the deliverable met the terms.

The obvious answer is to ask an LLM jury. It is also the wrong answer for most terms, for two reasons we measured rather than assumed.

**A jury can be unanimously wrong.** In a [controlled study](https://github.com/Makabeez/webclaims) on this same network, five validators reached unanimous agreement on three answers that were false — they were correctly reading a truncated artifact that never contained the file list the claims referred to. Consensus guarantees agreement, not truth.

**A jury often reaches no verdict at all.** Running the same judged term three times on the same artifact, only one settled. The other two finalized with `NO_MAJORITY` and `DISAGREE` — nothing decided, nothing paid, escrow stuck.

Exhibit's position: **most terms in an agent contract do not need judgment at all.** Whether a response contains a field, whether an array has ten entries, whether a status equals `ok` — these are decidable. Only the genuinely subjective ones need a jury.

So Exhibit splits the terms.

## Two classes of term

A buyer commits to terms before any work starts. A term with a **kind prefix** is mechanical; anything else is judged.

```
contains: Cargo.toml
json_equals: status = ok
json_exists: data.items
array_min: results >= 10
the summary accurately describes the linked article     ← judged
```

**Mechanical terms** are evaluated in deterministic Python after a single consensus fetch. Every validator computes the same answer from the same bytes. No LLM, no variance, and structurally impossible to be unanimously wrong in the way the study documented.

**Judged terms** go to the validator jury, which returns one boolean per term and must agree on every one.

Mechanical terms are evaluated **first**. If any fails, the deliverable is already out of spec and the jury is never convened — a cheap deterministic check gates the expensive probabilistic one.

## Measured on Bradbury

Six cases, one contract, one artifact, identical terms within each group. The only variable is whether the terms are mechanical or judged.

| Group | Terms | Settled on first attempt |
| --- | --- | --- |
| Mechanical | `contains: web_claims.py` + `contains: README.md` | **3 of 3** |
| Judged | `contains: web_claims.py` + *"the artifact lists a Rust project"* | **1 of 3** |

The two judged failures took different routes and neither wrote any state:

| Case | Outcome |
| --- | --- |
| `J1` | jury agreed, case `RULED` |
| `J2` | `FINALIZED / NO_MAJORITY` — the jury never reached a majority |
| `J3` | `UNDETERMINED / DISAGREE` — validators returned different booleans |

Both cases remain stuck at `SUBMITTED` with the escrow held. Nothing was decided and nothing was paid.

**This is the argument for the design.** A jury is a scarce and unreliable resource: it costs an LLM round per validator, it can be [unanimously wrong](https://github.com/Makabeez/webclaims), and on this evidence it fails to produce a verdict a substantial fraction of the time. Every term moved from judged to mechanical is a term that cannot fail either way.

Exhibit does not claim LLM juries are the wrong tool. It claims they are the *expensive* tool, and that most terms in an agent contract never needed one.

### Earlier single-case runs

Four cases covering all four mechanical kinds, with wall-clock settlement times:

| Case | Terms | Jury | Settlement |
| --- | --- | --- | --- |
| `json-1` | `array_min` + `json_exists` | no | 1.7 min |
| `gate-1` | failing `contains` + judged | no | 3.6 min |
| `mix-1` | passing `contains` + judged | yes | 9.6 min |
| `mech-1` | two `contains` | no | — |

`gate-1` is the gate working: it carries a judged term, but its mechanical term fails, so no jury runs. The deliverable is out of spec on a fact.

`json-1` exercised path-walking against live GitHub API JSON — `array_min: 0.name >= 1` correctly false (`name` is a string, not a list), `json_exists: 0.sha` correctly true.

`mix-1` ruled correctly on both halves: the artifact does contain `web_claims.py` and does not describe a Rust project. It also required a retry after finishing `NOT_VOTED` on the first attempt.

Timings are wall-clock from submission and single runs, so treat them as indicative. The settlement-rate result above is the one that replicates.

## Lifecycle

```
open_case(id, brief, terms)      payable — the value sent is the escrow
submit(id, artifact_url)         https, must pin a 40-char commit SHA
settle(id)                       one fetch → mechanical → jury only if needed
release(id)                      escrow to whoever the terms decided
appeal(id)                       payable — bond equals escrow, reopens the case
```

A ruling holds the escrow rather than paying it. That gap is what makes an appeal possible: the losing party stakes a bond equal to the escrow and a fresh jury rules on the same pinned artifact. Overturned, the appellant takes escrow plus bond. Upheld, both go to the other side.

## Why the artifact must be pinned

`submit` rejects any URL without a 40-character commit SHA. A branch reference can be rewritten between delivery and settlement, which would let the party being judged edit its own evidence after the fact. For agent-to-agent commerce this is not hypothetical — the seller controls the endpoint the contract reads.

## Why it refuses to rule on a partial view

The artifact is read into a 6,000-character window. If the fetch fills that window it was almost certainly truncated, and the contract returns `INADMISSIBLE` with the escrow held rather than ruling on evidence it cannot see the end of.

This is the direct consequence of the study: truncation is exactly the condition that produced unanimous false verdicts.

## Consensus design

`settle()` runs at most **two non-deterministic blocks**, both in the same method.

**Block 1 — read the artifact.** `prompt_comparative` with a tolerant principle. Even an immutable artifact differs in whitespace between fetches; `strict_eq` would deadlock the jury on noise.

**Block 2 — rule on judged terms.** `prompt_comparative` wrapping `gl.nondet.exec_prompt`, returning `{"rulings": [{"id": 0, "met": true}]}`. The principle: same ids, same booleans, nothing else matters. This block only runs when judged terms exist and every mechanical term passed.

Everything after that is deterministic: twelve validation checks on the jury's output, then the outcome computed from the agreed booleans.

## Prompt injection

The seller controls the artifact the contract reads, and that artifact decides whether they get paid. The ruling prompt states that anything inside the artifact block — instructions, role-play, claims of authority — is data to be judged, never instructions to follow, and the artifact is delimited with explicit tags.

## Verifying the contract

```bash
pip install genvm-linter
genvm-lint check exhibit.py
genvm-lint schema exhibit.py
```

Note that `genvm-lint` accepts several constructs GenVM rejects at deploy. See below.

## Notes for builders

Undocumented behaviours found building this and its predecessor:

- **A contract cannot have non-deterministic blocks in two different methods.** Deployment fails with no readable diagnostic. Isolated with two probes: a second payable method deploys, a second method with bond logic and `emit_transfer` deploys, adding a `gl.eq_principle` block to it does not.
- **Do not put free text in an answer the equivalence principle compares.** Asking the jury for a `reason` string and telling the principle to ignore it produced four leader rotations and `UNDETERMINED` — with identical booleans from every leader.
- **No `try/except` in contract code.** GenVM rejects `except Exception` at schema generation though `genvm-lint` accepts it.
- **A settle call can finalize with `NOT_VOTED`** — no committee picked it up, nothing applied. Seen three times across two contracts; resubmitting the identical call worked every time. For unattended agent-to-agent settlement this needs a retry loop.
- **`rc.getBalance` and `waitForTransactionReceipt` are unreliable.** Poll `getTransaction` and read contract state.
- **The RPC rate-limits bursts.** Three transactions in quick succession returned `transaction gas rate limit exceeded`. Space them.
- **Writing a contract from scratch is the slow path.** Three from-scratch deploys of this contract failed with no diagnostic and neither suspect proved to be the cause. A mechanical rename of an already-deployed contract, extended one step at a time, worked first try.

## Files

- `exhibit.py` — the Intelligent Contract
- `index.html` — the demo: read cases, terms and per-term rulings

## License

MIT
