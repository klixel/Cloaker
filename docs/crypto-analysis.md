# Cryptographic Analysis of Cloaker (`klixel/Cloaker`)

_Date:_ 2026-04-20  
_Repository:_ `klixel/Cloaker`  
_Analyzed branch reference:_ `master` at commit `2ae8f66624ae6fa53f5d07d6ecc550f8d3cdf8c8`  
_Prepared for:_ cryptosecurity research

## Executive Summary

Cloaker appears to implement password-based file encryption using libsodium via the Rust `sodiumoxide` bindings, with modern authenticated streaming encryption in current versions and backward-compatible legacy decryption support.

Based on reviewed code paths:

- Current/modern files are distinguished by a signature and decrypted through the primary decrypt routine.
- Legacy-format files are handled in `core/src/legacy.rs`.
- The legacy routine uses:
  - `pwhash::derive_key(...)` with interactive limits,
  - `secretstream::xchacha20poly1305` for chunked authenticated decryption.

**Recovery implication:** If a target file was encrypted correctly with the latest format and the password is unknown, practical plaintext recovery is typically infeasible absent:
1. a password guess/candidate source,
2. an implementation flaw,
3. side-channel/artifact leakage,
4. or user operational mistakes.

---

## Scope and Method

This assessment is static code review of surfaced repository files and code-search hits, focusing on cryptographic architecture and recoverability.

### Repository components (high level)

- `core/` — cryptographic core logic (encrypt/decrypt, legacy handling, OS interface)
- `cli/` — command-line wrapper and argument handling
- `gui/` — Qt interface and mode detection helpers
- `adapter/` — Rust/C++ bridge layer

---

## Observed Cryptographic Design

## 1) Primitive and API choices

The code references libsodium through `sodiumoxide`:

- Password hashing / key derivation (`sodiumoxide::crypto::pwhash`)
- Authenticated streaming encryption (`secretstream::xchacha20poly1305`)

In `core/src/legacy.rs`, decryption flow:

1. Parse salt (special handling depending on legacy signature behavior),
2. Parse `secretstream` header,
3. Derive key from password + salt using `pwhash::derive_key`,
4. Initialize pull stream with header + key,
5. Decrypt authenticated chunks.

This is structurally aligned with safe, modern patterns for file encryption.

## 2) File identification and version routing

In `core/src/os_interface.rs`, decrypt mode:

- Reads first four bytes and compares to current signature.
- If current signature matches, goes to modern decrypt path.
- Otherwise routes to legacy handler (with additional first-four-byte handling).

GUI layer also includes:
- `FILE_SIGNATURE = 0xC10A6BED`
- `LEGACY_FILE_SIGNATURE = 0xC10A4BED`
for mode inference and UX behavior.

## 3) Key derivation cost profile (legacy observed)

Legacy uses:

- `pwhash::OPSLIMIT_INTERACTIVE`
- `pwhash::MEMLIMIT_INTERACTIVE`

This favors usability and responsiveness; it is usually weaker against offline cracking than “sensitive”/higher-cost profiles, but still substantially increases brute-force cost versus raw hashing.

---

## Recoverability Analysis

## What is realistically possible?

### If password is unknown and strong:
- Direct cryptanalytic break is unlikely.
- Recovery is generally not feasible.

### If password is weak/reused/patterned:
- Offline dictionary/targeted candidate attacks may succeed.

### If implementation or operational flaws exist:
- Recovery may become possible via non-cryptanalytic vectors (artifacts, memory, incorrect format handling, etc.).

---

## Risk and Attack Surface Checklist

## A) Cryptographic misuse risk (code-level)

- [ ] Confirm modern encrypt path uses unique per-file random salt.
- [ ] Confirm modern path uses fresh secretstream header/nonce material.
- [ ] Confirm no key/nonce/header reuse across files.
- [ ] Confirm no unauthenticated plaintext release before tag verification.
- [ ] Confirm decryption error paths do not leak oracle-quality distinctions.

## B) KDF hardness and brute-force economics

- [ ] Confirm exact pwhash algorithm and parameters in **modern** path (opslimit, memlimit, algorithm id).
- [ ] Benchmark guesses/sec on realistic hardware.
- [ ] Estimate attack time for probable password distributions.

## C) Version/format confusion

- [ ] Fuzz first bytes/signature handling.
- [ ] Ensure no downgrade confusion from modern to legacy parser that weakens guarantees.
- [ ] Validate handling of malformed short files and mixed-format inputs.

## D) Operational leakage

- [ ] CLI `-p` password exposure via shell history/process list.
- [ ] Password file handling (`--password-file`) and accidental persistence.
- [ ] Temporary files, crash dumps, swap files, autosave artifacts.
- [ ] GUI memory lifetime of password strings.
- [ ] Logs/telemetry that may include file paths or errors with sensitive context.

---

## Suggested Research Workflow (Practical)

## Phase 1 — Format and implementation mapping

1. Document exact on-disk format for both modern and legacy files:
   - signature bytes,
   - salt length and location,
   - stream header position/length,
   - chunk framing and overhead.

2. Create parser scripts to:
   - detect format version,
   - extract metadata fields (non-secret only),
   - validate structural integrity.

## Phase 2 — Security property verification

1. Build test corpus:
   - tiny, medium, huge files,
   - random data and known plaintext patterns,
   - cross-version generated ciphertexts.

2. Add negative tests:
   - wrong password,
   - bit-flipped header/chunks,
   - truncated ciphertext,
   - modified salt/signature.

3. Verify consistent failure semantics and absence of partial plaintext output.

## Phase 3 — Recovery feasibility study

1. Use known test files with controlled passwords to calibrate cracking speeds.
2. Model candidate attack sets (rules, masks, targeted dictionaries).
3. Quantify expected time-to-success under realistic assumptions.

## Phase 4 — Side-channel and artifact forensics

1. Examine system-level residue:
   - shell history,
   - temp dirs,
   - file-system snapshots,
   - editor backups/cloud sync versions.

2. Memory forensics (if legally authorized):
   - process memory capture during active decrypt operations,
   - string and key-material scavenging windows.

---

## Guidance for Attempting File Recovery (Ethical/Legal)

Only proceed where you have legal authorization and ownership rights.

### Priority order

1. **Acquire probable password candidates** (highest ROI).
2. **Check for backups/artifacts** (often overlooked).
3. **Run targeted, not blind, cracking** using model-driven candidates.
4. **Investigate implementation weaknesses** only if evidence suggests one.

### Avoid

- Blind exhaustive brute force over full high-entropy space.
- Assuming cryptographic break is plausible without strong evidence.

---

## Notable Repository Observations

- README explicitly warns that forgotten passwords mean data loss.
- GUI enforces a minimum 12-character password for encryption prompts.
- CLI supports password via argument or password-file; docs warn about command-history risk.
- A brute-force experiment file (`cli/src/brute_force.rs`) reinforces impracticality of exhaustive search.

---

## Limitations of This Assessment

- This report is based on a partial static review and surfaced code-search results.
- Full assurance requires complete inspection of all encryption/decryption source paths, dependency versions, and runtime behavior.
- No dynamic tracing, fuzzing, or side-channel measurement has yet been executed in this report.

---

## Recommended Next Deliverables

1. **Byte-level format specification** (`docs/file-format.md`)
2. **Threat model matrix** (`docs/threat-model.md`)
3. **KDF benchmark report** (`docs/kdf-benchmarks.md`)
4. **Recovery decision tree** for incident response (`docs/recovery-playbook.md`)
5. **Hardening recommendations** with severity and effort estimates

---

## Appendix: Initial File References Reviewed

- `README.md`
- `core/src/os_interface.rs`
- `core/src/legacy.rs`
- `cli/src/main.rs`
- `cli/src/brute_force.rs`
- `gui/cloaker/adapter.cpp`
- `gui/cloaker/adapter.h`
- `adapter/src/lib.rs`
- `gui/cloaker/mainwindow.cpp`
