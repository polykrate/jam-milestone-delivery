# JAM Milestone Delivery


## Company/Team details

- Company/Team's name: Carge
- Company/Team's GitHub: https://github.com/polykrate
- Programming language and language set: Common Lisp (Set 5 — novel/functional)
- Link/s to previous delivery/ies: N/A (first milestone)


## Documentation checklist

We declare that:

- [x] we have completed **the Web3 Foundation KYC/KYB process**.
- [x] we used **a clear and permissive open-source license**. GPL-3.0 (OSI-approved).
- [x] we submitted **a clear Git history and public, credibly timestamped commits**.
- [x] we used third party libraries for: **cryptographic primitives** (Blake2b, Bandersnatch VRF, Ed25519 via Rust FFI using `ark-vrf` 0.2.1).
- [x] we provided **Gas, trie/DB, signature-verification, and availability (EC/DB) performance tests** to be run on standard hardware.
- [x] we viewed the following **JAM implementation code** during our implementation: **Strawberry** (Go, Eiger — [github.com/eigerco/strawberry](https://github.com/eigerco/strawberry)) for Merkle trie bit-ordering and node format; **TypeBerry** (TypeScript, FluffyLabs — [github.com/FluffyLabs/typeberry](https://github.com/FluffyLabs/typeberry)) for AccumulateItem field order (Operand.Codec) and erasure coding stride-64 layout.
- [x] we have **not** had private conversations with other implementers.
- [x] we have **not** had concerns about collusion.
- [x] we agree to a recorded interview by the *Polkadot Technical Fellowship* on any matter arising from this milestone submission.
- [x] we understand that this milestone submission will need to be ratified with an on-chain remark by the *Polkadot Technical Fellowship* before it can be merged.



## Context

JOTL (JAM On The Lisp) is a clean-room implementation of the JAM state transition function Υ(σ, B) → σ' in **Common Lisp**, targeting the [Gray Paper](https://graypaper.com) v0.7.2.

This first milestone delivers a **fully conformant STF implementation** that passes all public test vectors (1000/1000 blocks across 8 trace categories), the polkajam-fuzz traces (760/760 steps), and the minifuzz protocol (forks and no_forks).

The implementation includes:
- A complete **PVM interpreter** (GP Appendix A) in pure Common Lisp with AOT precomputed tables and lazy instruction cache
- Full **host call** support (GP Appendix B) including all Ω functions
- All **17 state components** (α β γ δ η ι κ λ ρ τ ϕ χ ψ π ω ξ θ) as sovereign let-over-lambda closures
- The **4-wave dependency graph** orchestrator (GP §4.2.1) in `upsilon.lisp`
- **Accumulation** orchestration (GP §12) with PVM execution and privilege resolution
- **Merkle trie** state root computation (GP Appendix D)
- A **fuzz-v1 protocol** server for conformance fuzzing

Cryptographic primitives (Blake2b, Bandersnatch VRF, Ed25519, erasure coding) are implemented in Rust and accessed via CFFI.


## Deliverables

- [x] 1. Validating Node Path

- **Milestone:** 1

| Number | Deliverable | Link | Notes |
|--------|-------------|------|-------|
| 1. | Source code | [JOTL @ `4c82161`](https://github.com/polykrate/JOTL/tree/4c82161e66acde57903fbd246b46ff399b02554a) | Full STF implementation in Common Lisp (SBCL). ~13,500 lines of code + ~4,800 lines of comments. |
| 2. | Static test vectors conformance | [tests/conformance.lisp](https://github.com/polykrate/JOTL/blob/4c82161e66acde57903fbd246b46ff399b02554a/tests/conformance.lisp) | 1000/1000 blocks passing across all 8 traces (fallback, safrole, storage, storage_light, preimages, preimages_light, fuzzy_light, fuzzy). Run with `./scripts/test.sh`. |
| 3. | Fuzz traces conformance | [tests/polkajam-traces.lisp](https://github.com/polykrate/JOTL/blob/4c82161e66acde57903fbd246b46ff399b02554a/tests/polkajam-traces.lisp) | 205 polkajam-fuzz traces, 760 steps: 735 pass + 25 correct rejects, 0 failures. Run with `./scripts/test-reports.sh`. |
| 4. | Minifuzz conformance | [scripts/fuzz-target.sh](https://github.com/polykrate/JOTL/blob/4c82161e66acde57903fbd246b46ff399b02554a/scripts/fuzz-target.sh) | fuzz-v1 protocol: no_forks 102/102 ✓, forks 102/102 ✓. |
| 5. | PVM interpreter | [src/jamvm/](https://github.com/polykrate/JOTL/tree/4c82161e66acde57903fbd246b46ff399b02554a/src/jamvm) | Pure Common Lisp PVM (GP Appendix A). AOT precomputed tables (basic-block starts, skip-distance LUT) + lazy instruction cache, zero-allocation inner loop. |
| 6. | Host calls | [src/jam-host/](https://github.com/polykrate/JOTL/tree/4c82161e66acde57903fbd246b46ff399b02554a/src/jam-host) | All Ω host functions (GP Appendix B) including accumulate, on-transfer, privileged services. |
| 7. | Performance benchmarks | [tests/conformance.lisp (scoring)](https://github.com/polykrate/JOTL/blob/4c82161e66acde57903fbd246b46ff399b02554a/tests/conformance.lisp) | Parity scoring methodology. Per-trace P50, P90, Mean, P99, StdDev. Score ~16.5 on local hardware (Intel i7-10610U). |
| 8. | Crypto FFI (Rust) | [crypto/jam-crypto/](https://github.com/polykrate/JOTL/tree/4c82161e66acde57903fbd246b46ff399b02554a/crypto/jam-crypto) | Blake2b-256, Bandersnatch VRF, Ed25519 signature verification, erasure coding. |


## Additional Information

### Architecture

JOTL uses a **closure-based architecture** where every sigma state component is a `let-over-lambda` closure generated by a single macro (`define-state-closure`). Each closure receives messages (`:transition`, `:encode`, `:decode`) and returns a new version of itself. No CLOS, no mutation — pure functional state transitions.

The orchestrator σ (`sigma.lisp`) is a pure byte store that lazily decodes components on demand. Υ (`upsilon.lisp`) orchestrates the 4-wave dependency graph. `accumulate.lisp` handles §12 (R*, PVM execution, privilege resolution).

### Build & Run

```bash
# Build crypto FFI
cargo build --manifest-path crypto/jam-crypto/Cargo.toml --release

# Run all conformance tests (1000 blocks)
./scripts/test.sh

# Run polkajam fuzz-reports (205 traces)
./scripts/test-reports.sh

# Launch minifuzz target
./scripts/fuzz-target.sh /tmp/jam_target.sock
```
