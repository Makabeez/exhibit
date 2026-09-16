# Exhibit

An admissibility standard for evidence in agent-to-agent disputes.

**[Live demo](https://makabeez.github.io/exhibit/)** · **[Demo video](https://youtu.be/Upe4DsQX2hc)** · Deployed on GenLayer Bradbury at [`0xa5bD1ff37eCFD280F4CBdA73576DA7c86DB2D275`](https://explorer-bradbury.genlayer.com/address/0xa5bD1ff37eCFD280F4CBdA73576DA7c86DB2D275)

## The idea

Two agents transact. One escrows payment, the other delivers. Somebody has to decide whether the deliverable met the terms.

The obvious answer is to ask an LLM jury. It is also the wrong answer for most terms, for two reasons we measured rather than assumed.

**A jury can be unanimously wrong.** In a [controlled study](https://github.com/Makabeez/webclaims) on this same network, five validators reached unanimous agreement on three answers that were false — they were correctly reading a truncated artifact that never contained the file list the claims referred to. Consensus guarantees agreement, not truth.

**A jury is the expensive tool.** It costs an LLM round per validator, adds minutes of latency, and its reliability varies by network — on one testnet the same judged term settled 1 time in 3, on another 3 times in 3. Measurements below, including the ones that contradict each other.

**And every dependency on model output is a liability.** One run showed this contract's own settlement breaking because it asked the jury to echo a term index back, and the model renumbered from zero.

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

## Measured

Deployed twice, on two networks, with the same terms and the same pinned artifact. The three runs disagree with each other, and that disagreement is the most useful thing here.

**Live deployment: [`0x8379eb6c8425b561cf56AD5104911Ae860268001`](https://explorer-studio-dev.genlayer.com/address/0x8379eb6c8425b561cf56AD5104911Ae860268001) on Studio Next (chain 61997).**

### Run one, Bradbury: judged terms settled 1 of 3

| Group | Settled |
| --- | --- |
| Mechanical | **3 of 3** |
| Judged | **1 of 3** — one `NO_MAJORITY`, one `DISAGREE`, neither wrote state |

### Run two, Bradbury: a bug in this contract, caught by its own validation

Three further judged cases all reverted. Not the jury — the prompt numbered judged terms by their **original index**, so with one judged term at index 1 the jury was asked about term `1` and answered `{"id": 0, "met": false}`. It renumbered from zero, which is the obvious thing to do when shown a single item.

Earlier cases had succeeded only because the model happened to echo the index back. **The contract's correctness depended on an LLM preserving a non-contiguous numbering.** The strict validation layer refused the settlement rather than acting on it — three reverts, zero mis-settlements, escrow untouched. [Fixed](https://github.com/Makabeez/exhibit/commit/4fac692) by numbering `0..n-1` and mapping back in code.

### Run three, Studio Next: judged terms settled 3 of 3

The same six cases against the ported contract on a newer runtime with a fresh validator set:

| Group | Settled | Jury |
| --- | --- | --- |
| Mechanical | **3 of 3** | never convened |
| Judged | **3 of 3** | convened on 2, gated out on 1 |

**This contradicts run one, and the honest reading is that the 1-of-3 failure rate was a property of Bradbury, not of LLM juries.** Bradbury was a testnet being retired; it also produced repeated `NOT_VOTED` transactions where no committee picked up a call at all. We are not going to claim a general reliability finding from a network that was on its way out.

What survives is narrower and still worth building around: a jury costs an LLM round per validator, adds minutes of latency, and its reliability is a property of the network you happen to be on. A mechanical term has none of those dependencies.

### The finding we did not expect

In run three, `sn-J3` evaluated `contains: web_claims.py` as **false** — against the same pinned, immutable URL where `sn-J1` and `sn-J2` had evaluated it **true** minutes earlier.

The term is a substring check. The artifact is content-addressed. Neither can vary. What varied was the bytes the fetch returned: eighteen transactions in quick succession, five validators fetching per settle, against an unauthenticated API with a 60-requests-per-hour limit. A rate-limited response is still valid JSON — it just contains an error message instead of a file list, and `contains:` correctly reported the string was absent from it.

> **A deterministic term is only as deterministic as the bytes it is given.** Mechanical evaluation removes jury variance. It does not tell "the file is absent" from "we were handed an error page instead of the artifact," and the contract settled a case against the wrong document without hesitating.

This is the truncation problem at a different layer, and it points at the same fix: validate that the fetch *looks like* the expected artifact — expected key present, minimum size, no error shape — and return `INADMISSIBLE` otherwise, exactly as the window guard already does. Not yet implemented; it is the next thing to build.

### Earlier single-case runs

Four cases on Bradbury covering all four mechanical kinds, with wall-clock settlement times:

| Case | Terms | Jury | Settlement |
| --- | --- | --- | --- |
| `json-1` | `array_min` + `json_exists` | no | 1.7 min |
| `gate-1` | failing `contains` + judged | no | 3.6 min |
| `mix-1` | passing `contains` + judged | yes | 9.6 min |
| `mech-1` | two `contains` | no | — |

`gate-1` is the gate working: it carries a judged term, but its mechanical term fails, so no jury runs. Timings are single runs — indicative, not averages.

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

Porting from GenVM v0.2 to v0.6 (Studio Next) needed exactly three changes: `TreeMap` moved to `gl.storage.TreeMap`, `prompt_comparative` takes its principle **positionally** rather than as a keyword, and transfers go through a `@gl.evm.contract_interface` class instead of `gl.get_contract_at`. Everything else — `Address`, `gl.message`, payable decorators, dict returns, `UserError`, `gl.nondet.web.get`, `exec_prompt` — was unchanged.

The fastest debugging tool on GenLayer is the `gen_getContractSchemaForCode` RPC method: POST a contract to it and it returns a real Python traceback with the failing line, in about a second, for free. It catches compile errors but not call-site errors inside method bodies — `prompt_comparative(fn, principle=...)` passes the schema check and reverts at runtime.


Undocumented behaviours found building this and its predecessor:

- **A contract cannot have non-deterministic blocks in two different methods.** Deployment fails with no readable diagnostic. Isolated with two probes: a second payable method deploys, a second method with bond logic and `emit_transfer` deploys, adding a `gl.eq_principle` block to it does not.
- **Do not put free text in an answer the equivalence principle compares.** Asking the jury for a `reason` string and telling the principle to ignore it produced four leader rotations and `UNDETERMINED` — with identical booleans from every leader.
- **No `try/except` in contract code.** GenVM rejects `except Exception` at schema generation though `genvm-lint` accepts it.
- **A settle call can finalize with `NOT_VOTED`** — no committee picked it up, nothing applied. Seen three times across two contracts; resubmitting the identical call worked every time. For unattended agent-to-agent settlement this needs a retry loop.
- **`rc.getBalance` and `waitForTransactionReceipt` are unreliable.** Poll `getTransaction` and read contract state.
- **The RPC rate-limits bursts.** Three transactions in quick succession returned `transaction gas rate limit exceeded`. Space them.
- **The RPC can refuse a deploy on size.** A 19.4 KB contract is currently rejected with `gas limit too high` where a 14.5 KB one deployed fine, with no client-side gas setting able to override it.
- **Writing a contract from scratch is the slow path.** Three from-scratch deploys of this contract failed with no diagnostic and neither suspect proved to be the cause. A mechanical rename of an already-deployed contract, extended one step at a time, worked first try.

## Files

- `exhibit.py` — the Intelligent Contract
- `index.html` — the demo: read cases, terms and per-term rulings

## License

MIT
