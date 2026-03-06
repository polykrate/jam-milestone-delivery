# JAM Milestone Delivery


## Company/Team details

- Company/Team's name: JOTL
- Company/Team's GitHub: [https://github.com/polykrate](https://github.com/polykrate)
- Programming language and language set: Common Lisp / Set 5 (novel/functional)
- Link/s to previous delivery/ies: _none_


## Documentation checklist

We declare that:

- [x] we have completed **the Web3 Foundation KYC/KYB process**.
- [x] we used **a clear and permissive open-source license**. GPL-3.0 (OSI-approved).
- [x] we submitted **a clear Git history, credibly timestamped commits in a public GitHub repository**.
- [x] we used third party libraries for:
  - **cryptographic primitives**: `ark-vrf` 0.2.1, `ark-ed-on-bls12-381-bandersnatch`, `blake2`, `ed25519-zebra` (Rust FFI)
  - **encoding**: hand-rolled codec (no third-party serialization)
- [x] we provided **Gas, trie/DB, signature-verification, and availability (EC/DB) performance tests** to be run on standard hardware.
- [x] we viewed the following **JAM implementation code** during our implementation:
  - **Strawberry** (Go, Eiger — [github.com/eigerco/strawberry](https://github.com/eigerco/strawberry)) for Merkle trie bit-ordering and node format.
  - **TypeBerry** (TypeScript, FluffyLabs — [github.com/FluffyLabs/typeberry](https://github.com/FluffyLabs/typeberry)) for AccumulateItem field order and erasure coding stride-64 layout.
- [x] we have **not** had private conversations with other implementers.
- [x] we have **not** had concerns about collusion.
- [x] we agree to a recorded interview by the _Polkadot Technical Fellowship_ on any matter arising from this milestone submission.
- [x] we understand that this milestone submission will need to be ratified with an on-chain remark by the _Polkadot Technical Fellowship_ before it can be merged.


## Context

I am a former military officer who started into software engineering, then moved into administration, logistics, and finance. Having seen firsthand how blockchain is underutilised in administrative and logistics workflows, I decided to acquire deep protocol-level skills by building something real.

**JOTL** (JAM On The Lisp) is a Common Lisp implementation of the JAM state transition function Υ(σ, B) → σ', targeting [Gray Paper](https://graypaper.com) v0.7.2. I chose Common Lisp over Rust because I believe a functional, homoiconic language is better suited to describe a complex STF: every transition reads as a direct transcription of the Gray Paper equations.

This milestone delivers a fully conformant STF:
- **1000/1000** static test vectors across 8 trace categories
- **760/760** polkajam-fuzz steps (735 pass + 25 correct rejects)
- **204/204** minifuzz inputs (no\_forks + forks)
- **16** specification-level bugs found and resolved, each traced to a specific GP section

The codebase is ~17k lines of Common Lisp (26% comments, 33% on PVM) plus ~2k lines of Rust for cryptographic FFI. The Bandersnatch SRS is bundled (577KB) so the implementation is fully self-contained.


## Deliverables

- [x] 1. Validating Node Path
- [ ] 2. Non-PVM Validating Node Path
- [ ] 3. Light Node Path

- **Milestone:** 1

| Number | Deliverable | Link | Notes |
|--------|-------------|------|-------|
| 1. | JOTL source code | [polykrate/JOTL][jotl] | Full STF in Common Lisp (SBCL). 17 state components as immutable closures, 4-wave dependency orchestrator, PVM interpreter, all host calls. |
| 2. | Static conformance | [tests/conformance.lisp][conformance] | 1000/1000 blocks across 8 traces (fallback, safrole, storage, storage\_light, preimages, preimages\_light, fuzzy\_light, fuzzy). |
| 3. | Fuzz traces | [tests/polkajam-traces.lisp][fuzz-traces] | 205 polkajam-fuzz traces, 760 steps: 735 pass + 25 correct rejects, 0 failures. |
| 4. | Minifuzz (fuzz-v1) | [scripts/fuzz-target.sh][fuzz-target] | no\_forks 102/102 ✓, forks 102/102 ✓. Fork-aware state manager with ancestry support. |
| 5. | PVM interpreter | [src/jamvm/][pvm] | Pure Common Lisp (GP Appendix A). AOT basic-block analysis, skip-distance LUT, lazy instruction cache. |
| 6. | Host calls | [src/jam-host/][hostcalls] | All Ω functions (GP Appendix B) including accumulate, on-transfer, privileged services. |
| 7. | Performance | [tests/conformance.lisp][conformance] | Per-trace P50/P90/P99/Mean. Periodic GC for bounded heap growth over long sessions. |
| 8. | Crypto FFI | [crypto/jam-crypto/][crypto] | Rust: Blake2b-256, Bandersnatch Ring VRF, Ed25519. SRS bundled in `data/`. |
| 9. | Docker image | [Dockerfile][dockerfile] | Multi-stage build. Non-root user. Self-contained fuzz target. |

[jotl]: https://github.com/polykrate/JOTL
[conformance]: https://github.com/polykrate/JOTL/blob/master/tests/conformance.lisp
[fuzz-traces]: https://github.com/polykrate/JOTL/blob/master/tests/polkajam-traces.lisp
[fuzz-target]: https://github.com/polykrate/JOTL/blob/master/scripts/fuzz-target.sh
[pvm]: https://github.com/polykrate/JOTL/tree/master/src/jamvm
[hostcalls]: https://github.com/polykrate/JOTL/tree/master/src/jam-host
[crypto]: https://github.com/polykrate/JOTL/tree/master/crypto/jam-crypto
[dockerfile]: https://github.com/polykrate/JOTL/blob/master/Dockerfile


## Additional Information

### Architecture

Every state component is an immutable `let-over-lambda` closure generated by `define-state-closure`. Closures encapsulate data, codec, and transition behind a message-passing interface (`:transition`, `:encode`, `:decode`). They are fork-safe by construction: no defensive copying is required for branching.

σ (`sigma.lisp`) is a pure byte store that lazily decodes components on demand. Υ (`upsilon.lisp`) implements the 4-wave dependency graph (GP §4.2.1). `accumulate.lisp` orchestrates §12 (R\* extraction, per-service PVM execution, privilege resolution). `import-block` is the trust boundary: below it, all computation is pure and deterministic.

δ is stored as a flat key-value list (`delta-kvs`) in Merkle trie format (GP Appendix D), avoiding decode/encode round-trips. Services are indexed lazily via `build-sid-index` for O(1) lookups.

### Build & test

**Docker** (self-contained — no local dependencies):

```bash
docker build -t jotl .
docker run --rm -v /tmp:/tmp jotl /tmp/jam_target.sock
```

**Native** (requires SBCL ≥ 2.3, Quicklisp, Rust stable):

```bash
# Crypto FFI
cargo build --manifest-path crypto/jam-crypto/Cargo.toml --release

# Test data (sibling repos)
git clone https://github.com/w3f/jamtestvectors.git ../jamtestvectors
git clone https://github.com/w3f/jam-conformance.git ../jam-conformance

# Conformance (1000 blocks, 8 traces)
./scripts/test.sh

# Polkajam fuzz-reports (205 traces)
./scripts/test-reports.sh

# Fuzz-v1 target
./scripts/fuzz-target.sh /tmp/jam_target.sock
```

### Performance roadmap (M2)

M1 prioritises conformance correctness. The following optimisations are identified and planned:

- **Merkle trie**: currently full O(n log n) recompute per block → incremental trie O(k log n) for k modified entries.
- **δ state**: `copy-alist` O(n) per block for fork safety → persistent data structure with structural sharing.
- **PVM hot loop**: the interpreter is pure CL; a tighter dispatch loop is the main single-block bottleneck on heavy traces.
