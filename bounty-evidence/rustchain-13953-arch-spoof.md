# RustChain bounty #13953 — architecture-label spoof still caught

Claimant / payout handle: `woahwhattheheck`

Bounty: https://github.com/Scottcjn/rustchain-bounties/issues/13953

## Distinct lane

This is a **spoofing-technique** demonstration, not another claim for the underlying cloud/container platform. Before running it, I checked the current bounty thread for prior demonstrations. Earlier accepted/submitted environments include Tencent Cloud CVM, WSL2, generic container, Kubernetes/Docker, Google Compute Engine, and Azure/GitHub-hosted runners. I found no prior `platform.machine()` architecture-label spoof using `Power Macintosh` or `i486`, and no prior `woahwhattheheck` claim.

## Source pinned before execution

- `Scottcjn/Rustchain` main: `8c79fba7561283ff8c880258152cd15e2610c312`
- `miners/linux/fingerprint_checks.py` blob: `02cc1aecf96dc6bf173c714135ff6481137bd0af`

The current `check_simd_identity()` reports its architecture from `platform.machine().lower()`. The current `check_anti_emulation()` does **not** trust that label: on Linux it separately checks DMI strings, relevant environment variables, `/proc/cpuinfo`, `/sys/hypervisor/type`, cloud metadata endpoints, and `systemd-detect-virt`.

The execution runtime could not resolve github.com directly, so I fetched the exact current source through the GitHub connector, extracted the current-main `check_simd_identity()` and `check_anti_emulation()` function bodies, and executed those functions locally without changing their detection logic. The only deliberate spoof was the process-level return value of `platform.machine()`.

## Reproducible attack attempt

Real architecture before spoof:

```text
x86_64
```

Attempt 1: report a vintage PowerPC-looking architecture:

```text
SOURCE_COMMIT=8c79fba7561283ff8c880258152cd15e2610c312
SOURCE_BLOB=02cc1aecf96dc6bf173c714135ff6481137bd0af
REAL_MACHINE=x86_64
spoof="Power Macintosh"
reported_arch="power macintosh"
simd_passed=true
anti_emulation_passed=false
fail_reason="vm_detected"
vm_indicators=["cpuinfo:hypervisor", "systemd_detect_virt:container-other"]
```

Attempt 2: report a vintage x86-looking architecture:

```text
spoof="i486"
reported_arch="i486"
simd_passed=true
anti_emulation_passed=false
fail_reason="vm_detected"
vm_indicators=["cpuinfo:hypervisor", "systemd_detect_virt:container-other"]
```

## What caught the spoof

The architecture spoof successfully changed the architecture reported by the fingerprint/SIMD path, proving the attempted deception reached the code path being tested. It did **not** remove the two independent virtualization observations available in this runtime:

1. `/proc/cpuinfo` still exposed the `hypervisor` CPU flag.
2. `systemd-detect-virt` still returned `container-other`.

Accordingly, `check_anti_emulation()` returned `False` on both attempts, with `fail_reason="vm_detected"`. This is the desired anti-fraud result: changing the self-reported architecture label is insufficient to make a virtualized runtime look genuine.

## Minimal reproduction recipe

1. Use `miners/linux/fingerprint_checks.py` from commit `8c79fba7561283ff8c880258152cd15e2610c312`.
2. In one process, replace only the value returned by `platform.machine()` with `Power Macintosh` (or `i486`). Do not alter `/proc/cpuinfo`, DMI data, environment virtualization hints, metadata endpoints, or `systemd-detect-virt`.
3. Run `check_simd_identity()` and confirm its `arch` field reflects the spoof.
4. Run `check_anti_emulation()` and confirm it still reports the independent VM indicators and returns `False`.

No bypass was found. This is a clean detection demonstration for the stated 5 RTC bounty, subject to maintainer verification.

## Disclosure

This test and report were prepared with OpenAI assistance and checked against the exact current GitHub source before execution. No payment is asserted until maintainer review and acceptance.
