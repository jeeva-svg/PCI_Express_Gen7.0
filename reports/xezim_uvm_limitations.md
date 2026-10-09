# ChipSage Labs · Tool Assessment
## Xezim UVM capability, limitations and PCIe project suitability

**Version:** v1.1 | **Status:** Draft for client review | **Date:** 9 October 2026 | **Audience:** Engineering management, DV leads

All paths are relative to `chipsage/PCI_Express_Gen7.0/dv/eval/` unless stated otherwise. Every result in this report was produced on the evaluation host and can be reproduced with the commands given. Nothing is quoted from memory or from third parties. Conclusions are labelled as one of:
- **Fact:** a measured result, with its log.
- **Judgment:** an engineering conclusion drawn from facts.
- **Untested:** not examined in this assessment.

## 01 Executive summary

The question is not whether a UVM testbench can execute on Xezim. It can: the full PCIe UVM 1.2 environment runs to completion. The question is whether every UVM feature the PCIe project needs is supported and can be qualified. To answer it, each required feature was tested individually with a self-checking test, a positive control and, where meaningful, a negative control. Tests ran on the installed build and on the latest upstream build, against UVM 1.2, IEEE 1800.2-2017 and IEEE 1800.2-2020.

**What works (Passed).** 144 of 149 individual checks pass, identically on both Xezim builds and all three UVM libraries:
- phasing, objections, drain time and timeouts
- the factory, config_db and resource_db
- reporting and report catchers
- sequences, including virtual sequences, arbitration, lock, grab and the sequence library
- TLM 1 and TLM 2
- callbacks, events and barriers
- SVA, including local variables, `throughout`, goto repetition and assertion control from the top
- the register layer: frontdoor, backdoor, predictor, W1C, and the built-in reset, bit-bash, access and memory-walk sequences
- every `uvm_hdl_*` function
- DPI-C (open arrays, strings, chandles, context imports, exported time-consuming tasks)
- VPI (system functions, value-change and read-only-synch callbacks)
- two top modules, clocking blocks and mixed timescales

Same-seed runs are byte-identical and parallel runs match serial runs.

**What does not work (Confirmed limitations).** 31 limitations: 26 confirmed (16 found or rechecked in this assessment, 10 carried over from the earlier evaluation) and 5 conditional, plus 3 unverified observations. Each confirmed or conditional limitation has a reproducer that fails identically on all three Xezim builds tested: installed `83df0dd5`, upstream `23f3728c`, and upstream HEAD `f56bb4df`, current at the time of writing. Where an independent reference exists, Verilator 5.050 produces the expected result on the same code. Most of them return wrong values **silently**, with no error or warning. The ones with the widest reach:
- **XEZ-UVM-001:** a `fork ... join_none` child does not share the parent task's automatic variables (lost updates, missed handshakes).
- **XEZ-UVM-002:** a local variable in a package class can silently bind to a same-named top-level instance and write into it.
- **XEZ-UVM-006:** the constraint solver never selects a valid branch when an array size depends on a nested random object.
- **XEZ-UVM-009, XEZ-UVM-010:** per-instance functional coverage is lost in two common construction patterns.
- **XEZ-UVM-014:** `UVM_ERROR` and `UVM_FATAL` never change the exit status, even with `--error-exit`. A regression system must parse logs.
- **XEZ-UVM-016:** IEEE 1735 encrypted code is skipped without a diagnostic, which blocks encrypted commercial PCIe VIP.
- **XEZ-UVM-018:** there is no coverage merge, UCIS database, report tool, or condition or FSM coverage.

**Agreement with the project's trusted simulator.** Icarus Verilog 12.0 is the simulator named in `README.md` and `tb/Makefile`. On the repository's own Root Complex BFM bench (`tb/`), Xezim gives the same check results as Icarus, and the identical LTSSM timeline once a testbench reset race is removed. Xezim runs that bench about 4.2 times faster (11.8 s against 49.8 s). The full `dv/` regression gives identical results on all three Xezim builds (§09, §08).

**Impact on the current PCIe environment.** The `dv/` environment avoids every confirmed limitation, either by construction (it was checked for each one) or through the workarounds already in place. Its results are therefore trustworthy for the scope it covers. The risk is in **new code**: the defects are silent, so new testbench code needs an independent cross-check before its results are trusted.

**Suitability decision** (details and evidence in §12):

**Table 1. Suitability decision summary**

| Use case | Decision |
|---|---|
| Controller-level and Transaction Layer UVM verification of this in-house RTL, with in-house agents and BFMs, for development and regression | **Suitable with restrictions (GO)** |
| Sole simulator for verification signoff | **Not suitable (NO-GO)** |
| Verification with encrypted commercial PCIe Root Complex VIP | **Not suitable (NO-GO)** for encrypted models. No such VIP was available to test. |
| Full-chip or SoC-level verification | **Not verified** |

> **Recommendation:** Xezim is suitable today for the PCIe project's bring-up, block-level and regression work with the UVM features listed as Passed, under the workarounds in §10 and the risk controls in §11. It is not yet qualified as the sole signoff simulator: the exit-status behaviour, silent wrong-result defects, missing coverage merge and inability to run encrypted VIP each block signoff on their own. A second, independent simulator (the planned Cadence Xcelium flow) should be used for signoff and for periodic cross-checks. [confirm Xcelium availability date]

## 02 Scope, method and classification

**Scope.** The requirements given for this assessment:
- UVM 1.2 and IEEE 1800.2 compatibility
- factory, config_db, phases, objections and reporting
- sequences, sequencers, drivers, monitors and TLM
- constrained random
- functional and cross coverage
- SVA
- RAL frontdoor and backdoor, and `uvm_hdl_*`
- DPI-C, VPI and callbacks
- multiple top modules, virtual interfaces, clocking blocks and timing
- PCIe regression reliability, debug, waveforms, performance and scalability

**Method.**
1. Read the official documentation at the evaluated commit: `docs/uvm-guide.md`, `coverage-guide.md`, `dpi-guide.md`, `command-line.md`, `user-guide.md` and `features.md`.
2. Read the actual PCIe testbench in `dv/`.
3. Wrote one self-checking UVM test per requirement area (`uvm_suite/u01` to `u16`). Each check prints `CHECK <id> PASS|FAIL <evidence>`, and expected values are derived by hand from the IEEE standard (the arithmetic is in the source comments). Positive controls show a feature working; negative controls show that the test can fail (for example, a wrong reset value must be reported).
4. When a check failed, the test was first audited for testbench errors. Eleven of the first failures were bugs in the assessment's own tests and were fixed (see §05). Each remaining failure was reduced to a minimal reproducer in `cases2/`.
5. Each reproducer was run on both Xezim builds and, if it uses no UVM, on Verilator 5.050 as an independent reference.

**Classification.**

**Table 2. Classification**

| Class | Meaning |
|---|---|
| Passed | The feature works in every check, on both builds and all three UVM libraries, including the negative controls. |
| Confirmed limitation | A minimal reproducer gives a wrong result, crash or refusal on valid code; or the documentation states the feature is unsupported and the test confirms it. |
| Conditional limitation | Fails only in a specific, identified condition and works outside it. A workaround is verified. |
| Unverified | Not tested, or observed but not reproduced with retained evidence. |

**Separating causes.** Each failure was attributed to exactly one of:
- a tool limitation
- UVM source compatibility (the testbench uses an API that differs between UVM versions)
- a missing dependency
- a testbench bug
- a configuration error

Only tool limitations are entered in the issue register. The others are listed in §05.

**Assumptions.**
- **A1:** the RTL under test is the repository's `rtl/` at commit `dc1c690`, plus the optional bring-up patch set in `../rtl_patches/` where stated.
- **A2:** "Gen5/Gen7" refers to the protocol generation the RTL models. Xezim's limitations concern the language, UVM and tool infrastructure and do not depend on the generation (Judgment). No Gen5-specific environment was run (Untested).
- **A3:** the client's "existing Root Complex VIP" is taken to be the repository's in-house BFM `tb/pcie_rc_bfm.sv`. No commercial Root Complex VIP exists in the repository. [confirm]

## 03 Environment

**Table 3. Environment and tool versions**

| Item | Value |
|---|---|
| Tool | Xezim, open-source SystemVerilog simulator, https://github.com/aionhw/xezim |
| Build A (installed) | 0.11.1-45-g83df0dd5, commit `83df0dd53506f12f58eff8cb87687423c86ebd78` (2026-10-08), `~/xezim/target/release/xezim` |
| Build B (upstream recheck) | main at commit `23f3728c97d64c598e3b929e59c604936048da9d` (2026-10-09, 13 commits after A, same version number); binary preserved at `~/.cache/xezim_builds/xezim_23f3728c` |
| Build C (upstream HEAD) | main at commit `f56bb4df67d416c9f386f505431da038c295c093` (2026-10-09, 11 commits after B, mostly randomization fixes); `~/.cache/xezim_builds/xezim_f56bb4df`. The user documentation (`docs/`) is identical at A, B and C; only `NOTES.md` changed. Suite results: `uvm_suite/results/xezim_head_f56bb4df_<uvm>/`. |
| UVM libraries | Accellera UVM 1.2, IEEE 1800.2-2017, IEEE 1800.2-2020.3.1 from GitHub `nitronis/UVM` commit 65a3ded at `~/uvm`; selected with `-uvm`, `XEZIM_UVM_DIR=~/uvm`, `XEZIM_UVM_VERSION=<release>` |
| Latest release | 0.11.1, published 2026-10-07 (GitHub releases API) |
| Build configuration | `cargo build --release` from source, default features. Build A reported "cranelift init failed; interpreter only" in the earlier evaluation (XZ-23); the JIT state of builds B and C was not checked |
| Project's trusted simulator | Icarus Verilog 12.0 (stable), named in `README.md` and `tb/Makefile`; it does not support UVM |
| Reference simulator | Verilator 5.050 (`verilator --binary --timing`), used only for reproducers without UVM |
| Host | Ubuntu, kernel 7.0.0-1013-aws, Intel Xeon Platinum 8375C, 16 vCPU, 61 GB RAM, shared with other users |
| Not available | Cadence Xcelium, any commercial PCIe VIP, any waveform viewer |

**Documentation status.** `git diff 83df0dd5 f56bb4df -- docs/` is empty, so the user documentation is identical across builds A to C (Fact). Only `NOTES.md` (release notes) changed. Documentation references below are pinned to commit `83df0dd5` and apply equally to build C.

**Standard command** (the exact command and environment of every run are in its `.cmd` file):

```
XEZIM_UVM_DIR=~/uvm XEZIM_UVM_VERSION=1.2 XEZIM_COV_DB=<run>.cov.json \
  xezim -uvm --max-time 10ms -s top <test>.sv +UVM_TESTNAME=<test>
```

**Reproducing everything:**

```
cd uvm_suite && ./run_suite.sh all xezim xezim_head        # suite, controls, determinism, parallel, scaling
XEZIM_UVM_VERSION=1800.2-2017 RESULTS_TAG=_1800.2-2017 ./run_suite.sh tests xezim xezim_head
cd ../cases2 && ./run_cases.sh                             # every minimal reproducer on 3 simulators
```

Outputs:
- `uvm_suite/results/<build>[_<uvm>]/<run>.{cmd,log,exit,time,cov.json}`
- `cases2/results/<case>.<simulator>.{cmd,log,exit}`
- `cases2/results/summary.txt`
- `xissues_recheck/`: the earlier x01 to x10 reproducers on both builds

## 04 Support matrix

Each row maps a required capability to the checks that test it. The check IDs are listed in Appendix A, and every check ran on both builds.

**Table 4. Support matrix**

| Capability | Checks | Result | Issues |
|---|---|---|---|
| UVM 1.2 | all 149 | Passed (144 pass; the 5 failures are tool defects below) | 005, 006, 009, 010 |
| IEEE 1800.2-2017 and 1800.2-2020.3.1 | all 149, rerun per library | Passed: identical results to UVM 1.2. Two tests needed version guards for API differences (§05). | as above |
| Phasing, phase jumps, drain time, timeout | U01-01 to U01-07, U01-N1 | Passed. Drain time fails under `-timescale` (U01-07). | 004 (conditional) |
| Objections | U01, all tests | Passed | |
| Reporting, catchers, verbosity, quit count | U02-01 to U02-08 | Passed | 014 (exit status) |
| Factory: type, instance, name overrides, plusarg | U03-01 to U03-06, U03-N1 | Passed | 003 (naming) |
| config_db, resource_db, wildcards, regex, plusarg, wait_modified | U04-01 to U04-09, U04-N1, N2 | Passed | |
| Sequences, items, responses, p_sequencer, virtual sequences | U05-01 to U05-04 | Passed | 003 (naming) |
| Arbitration FIFO, STRICT_FIFO, lock, grab, library, stop_sequences | U05-05 to U05-10 | Passed (the docs list lock and grab as limited; they work) | |
| Drivers, monitors, analysis ports, TLM FIFOs, put/get, TLM 2 | U06-01 to U06-07, U06-N1, U11, U15 | Passed | |
| Callbacks, uvm_event, event pool, barrier | U07-01 to U07-08 | Passed | 001, 008 (conditional) |
| Object copy, clone, compare, print | U08-01, 02, 04, 05, N1 | Passed | |
| Object pack/unpack with strings | U08-03 | Confirmed limitation | 005 |
| Constrained random: size, foreach, unique, sum, implication, solve-before, soft, dist, override, pre/post, local::, modes, stability | U16-01, 03, 04, 06 to 10; `cases/c_rand`, `c_seed` | Passed | |
| Constrained random: distribution and nested-object array sizes | U16-02, U16-05; `cases/c_dist` | Confirmed limitation | 006, 007 |
| Functional coverage: bins, ignore, illegal, transitions, iff, at_least, stop/start | U09-01 to 05, 09, 10, 13, 14 | Passed | |
| Cross coverage: auto, ignore_bins, named select bins | U09-06 to U09-08 | Passed | |
| Per-instance and merged coverage | U09-11, 11b, 11c, 12 | Conditional limitation | 009, 010, 011 |
| Coverage database content (automatic and cross bins) | documentation, U09 JSON | Confirmed limitation (documented) | 033 |
| Non-ASCII text in `$display` | `cases2/e_utf8_string_literal.sv` | Confirmed limitation | 034 |
| Coverage merge, UCIS, reports, condition/FSM coverage | documentation | Confirmed limitation (documented) | 018 |
| SVA: ranges, throughout, goto, $rose/$fell/$past/$stable, local variables, covers, named sequences | U10-01 to U10-07, U10-09 | Passed | |
| SVA assertion control and UVM reporting of failures | U10-08, U10-10 | Passed (from the top module) | 012 (conditional) |
| RAL model, adapter, predictor, frontdoor, backdoor, mirror, W1C, update | U11-01, U11-04 to U11-10 | Passed | 002 (naming) |
| RAL built-in sequences: hw_reset, bit_bash, access, mem_walk | U11-02, 03, 11 to 13 | Passed | |
| uvm_hdl check_path, read, deposit, force, release, release_and_read, force_time | U12-01 to U12-11, U11-14, U14-07 | Passed | |
| uvm_hdl partial force | U12-12 | Confirmed limitation (documented) | 019 |
| DPI-C: scalars, strings, open arrays, vectors, chandle, context, exports | U13-01 to U13-08, neg controls | Passed | |
| VPI: systf, arguments, cbValueChange, cbReadOnlySynch, put_value | U13-09 to U13-11 | Passed | |
| Multiple top modules, virtual interface across tops | U14-01, U14-07 | Passed | |
| Clocking blocks: output skew, #1step sampling, ##N, modports | U14-02 to U14-04, U14-08 | Passed | 029 (2-D element select) |
| Timing: fractional delays, mixed `timescale | U14-05, U14-06 | Passed | 004 (packages) |
| Regression: determinism, seeds, parallel runs | determinism, parallel | Passed | |
| Regression: exit status | neg controls | Confirmed limitation | 014 |
| Debug | documentation, earlier evaluation | Passed with limits (no interactive debugger) | |
| Waveforms | earlier evaluation, rechecked on build B | Conditional limitation | 015 |
| Encrypted IP (IEEE 1735) | `cases/c_protect` | Confirmed limitation | 016 |
| Performance and scalability | scale runs | Passed, with superlinear growth in component count | §07 |
| Commercial PCIe VIP | none available | Unverified | |

### Official documentation against verified results

The table compares what the official documentation claims with what was verified. `<base>` = `https://github.com/aionhw/xezim/blob/83df0dd53506f12f58eff8cb87687423c86ebd78/`.

**Table 5. Official claims against verified results**

| Area | Official documentation | Verified result (builds A, B, C) | Classification |
|---|---|---|---|
| UVM libraries | 1800.2-2017 and 1800.2-2020.3.1 run end to end (`<base>/docs/features.md#L47-L51`) | 1.2, 1800.2-2017 and 1800.2-2020 give identical suite results (144/149) | Verified supported |
| UVM reference examples | "32 of 35 UVM 1800.2-2017 example testbenches pass" (`<base>/docs/features.md#L52-L56`) | Not re-run; a public example claim, not project evidence | Not verified for this project |
| Phases, factory, config_db, reporting | listed as supported (`<base>/docs/uvm-guide.md#L169-L182`) | U01 to U04 pass; drain time fails only under `-timescale` (XEZ-UVM-004) | Verified supported (conditional on 004) |
| Sequences, lock/grab, RAL | lock/grab and RAL "beyond the common path" listed as limitations (`<base>/docs/uvm-guide.md#L191-L192`) | U05 (lock, grab, library, arbitration) and U11 (frontdoor, backdoor, predictor, W1C, hw_reset, bit_bash, access, mem_walk) all pass | Verified supported. The documentation understates support. |
| TLM, analysis, scoreboards, objections | supported (`<base>/docs/uvm-guide.md#L175-L176`) | U06 pass; `dv/` scoreboard and reference model run | Verified supported |
| Constrained random, seeds | `+seed` gives byte-identical output (`<base>/docs/command-line.md#L29`) | U16 8/10 pass; same seed byte-identical, different seed differs | Partially supported or restricted (XEZ-UVM-006, 007) |
| Functional and cross coverage | supported features (`<base>/docs/coverage-guide.md#L15-L30`) | U09 exact values pass for bins, crosses, ignore and illegal bins; per-instance failures | Partially supported or restricted (009, 010, 011) |
| Coverage database and merge | not supported (`<base>/docs/coverage-guide.md#L848-L862`, `#L416`) | confirmed by test (`cases2/e_cov_system_functions.sv`) and JSON inspection | Confirmed limitation (018, 033) |
| SVA | concurrent assertions and assertion control (`<base>/docs/features.md#L13-L27`) | U10 pass incl. local variables, throughout, goto, $assertoff from the top | Verified supported (012 conditional) |
| uvm_hdl backdoor | supported except partial force (`<base>/docs/uvm-guide.md#L147-L155`, `#L189-L190`) | U12 pass; partial force refused as documented | Verified supported; partial force: Confirmed limitation (019) |
| DPI-C | `--dpi-lib` (`<base>/docs/dpi-guide.md#L275-L320`) | U13-01 to 08 pass; imports in a package work although the guide says a module is required (L319-L320) | Verified supported |
| VPI | `--vpi-lib`, callbacks (`<base>/docs/dpi-guide.md#L378-L470`) | U13-09 to 11 pass (systf, cbValueChange, cbReadOnlySynch) | Verified supported (subset tested) |
| Multiple tops, interfaces, clocking | multiple tops supported (`<base>/docs/features.md#L50-L51`) | U14 pass; x08 2-D clocking element select fails (XEZ-UVM-029) | Verified supported (restricted by 022, 023, 029) |
| Timescales | `-timescale` applies to elements without one (`<base>/docs/command-line.md#L120-L121`) | package delays mis-scaled under `-timescale` / `-override_timescale` | Confirmed limitation, conditional (004) |
| Exit status | `--error-exit`: nonzero on `$error` (`<base>/docs/command-line.md#L55`) | UVM_ERROR/UVM_FATAL exit 0 in every mode | Confirmed limitation (014) |
| Waveforms | VCD and FST (`<base>/docs/user-guide.md#L70`) | VCD correct; FST aborts on the full UVM environment | Partially supported or restricted (015) |
| Encrypted IP | no statement found | envelopes skipped silently | Confirmed limitation (016) |
| Commercial VIP | no statement | no VIP available | Not verified for this project |

**Upstream issue tracker** (https://github.com/aionhw/xezim/issues, 155 issues reviewed, numbers 4 to 290):
- **Open:** two issues relate to this project.
  - #255, nested random class handles: same area as XEZ-UVM-006.
  - #290, DPI-export waits hang on macOS: not applicable on Linux.
- **Closed:** related issues whose fixes do not cover the variants found here:
  - #271, interface instance name collision in class-member comparisons: same mechanism as XEZ-UVM-002, which still fails on build C
  - #16, automatic variables in looped `fork..join`: XEZ-UVM-001 still fails
  - #107, exit code 0 on errors: fixed for `$error`, not for UVM reports (014)
  - #68 and #77, `.triggered`: XEZ-UVM-008 still fails
- **Not filed upstream:** no issue was found for XEZ-UVM-004, 005, 009, 010, 013, 015, 016 or 033. They should be reported. [confirm owner]

## 05 Failures that are not Xezim limitations

These were found during the assessment and are listed so they are not mistaken for tool issues.

**Table 6. Failures that are not Xezim limitations**

| Cause | Finding | Resolution |
|---|---|---|
| Testbench bug | U02-05: expected UVM_COUNT to gate the severity count; UVM 1.2 counts severities always and UVM_COUNT gates only the quit count (`uvm_report_server.svh` line 653) | expectation corrected |
| Testbench bug | U04-03: regex scope with a non-null context; config_db prefixes the context path (`uvm_config_db.svh` lines 103 to 105) | null context |
| Testbench bug | U04-06: `+uvm_set_config_int` stores `uvm_bitstream_t`, read with `uvm_config_db#(uvm_bitstream_t)` (`uvm_root.svh` line 844) | type corrected |
| Testbench bug | U05-10: `stop_sequences` while the driver holds an item produces a legitimate UVM_FATAL | driver drops the item |
| Testbench bug | U07-08: barrier release time compared to an absolute time | compared to the start time |
| Testbench bug | U09: `small` used as a bin name (IEEE 1800 reserved keyword) | renamed |
| Testbench bug | U10: `wait ($time >= ...)` has no event to wake on | computed delay |
| Testbench bug | U11-03: RO fields configured volatile; UVM skips volatile fields in mirror checks | not volatile |
| Testbench bug | U11-14: backdoor peek in the same time step as the frontdoor write's clock edge, before the DUT's nonblocking update | settle delay |
| Testbench bug | U12: hierarchical references inside a package (forbidden by IEEE 1800-2017 26.2) | `uvm_hdl_read` |
| Testbench bug | U14-03/04: read before the NBA update; `##3` started 1 ns after an edge ends 29 ns later, not 30 | expectations corrected |
| UVM source compatibility | `uvm_do_with` exists in 1800.2 only under `src/deprecated`; `uvm_comparer::show_max` is `local` in 1800.2 (`uvm_comparer.svh` line 119) | `ifdef UVM_VERSION_1_2` guards in U05 and U08 |
| UVM source compatibility | `uvm_top` is removed from 1800.2 unless `UVM_ENABLE_DEPRECATED_API` is set (found in the PCIe env earlier) | `uvm_root::get()` |
| Reference tool limits | Verilator 5.050 reports zero assertion failures even for `assert property (1'b0)` and returns 0.00 coverage on correct covergroup code | Verilator not used as a reference for SVA or coverage; those results rely on the IEEE standard |
| Missing dependency | none | |
| Configuration error | none | |
| Host | `/tmp` is full (other users' files); builds and Verilator output were moved to `~/.cache` and `~/xezim-eval-head` | not a tool issue |

**Project findings in the repository benches** (Fact):

**Table 7. Project findings in the repository benches**

| ID | Finding | Evidence | Applies to |
|---|---|---|---|
| PF-01 | `uvm_tb/` sets virtual interfaces with scope `"uvm_test_top.*"`, which does not match the test component itself (UVM config_db glob semantics). `pcie_smoke_test` stops with `UVM_FATAL [NOVIF] pcie_ctrl_if not found in config db` at time 0. | `pcie_suitability/logs/uvm_tb.xezim.log`; fix `pcie_suitability/uvm_tb_scope_fix.diff` | any simulator (testbench bug) |
| PF-02 | With PF-01 fixed, `uvm_tb/` reports `link_up observed after 0 cycles` at time 0: `while (!ctrl_vif.link_up ...)` exits on an X value (IEEE 1800-2017 12.4). The smoke sequence then waits until the 50 ms limit. | `pcie_suitability/logs/uvm_tb_scopefix.xezim.log` (wall 3:07, exit 0) | testbench weakness |
| PF-03 | `tb/tb_pcie_top.sv` line 441 releases `rst_n` with a blocking assignment right after `@(posedge core_clk)`, racing the RTL flops at the same edge (IEEE 1800-2017 4.7). Icarus and Xezim legitimately resolve it one cycle apart. With a nonblocking release, both give identical LTSSM timelines. | `pcie_suitability/logs/tb.*.log`, `tb_racefix.*.log`, `tb_reset_race_fix.diff` | testbench race |
| PF-04 | `README.md` states the `tb/` bench passes 10 of 10 tests. On Icarus 12.0, the documented flow, it stops at TEST 5 (AXI write) and the 5 ms watchdog ends the run with 7 checks passed and 0 failed. Xezim gives the same. | `pcie_suitability/logs/tb.iverilog.log` | DUT AXI path, consistent with `../docs/verification_plan.md` §6 |
| PF-05 | 28 RTL defects, found by the `dv/` environment | `../docs/verification_plan.md` §6 | RTL |

## 06 Issue register

Common to every entry:
- **Tool versions:** xezim 0.11.1, build A `0.11.1-45-g83df0dd5` and build B `23f3728c` (both print `=== xezim 0.11.1 ===` and their git hash in every log). Reference: Verilator 5.050. UVM: Accellera 1.2 unless stated.
- **Builds:** reproduced on build A (`83df0dd5`), build B (`23f3728c`) and build C (`f56bb4df`). Build C logs: `cases2/results/<case>.xezim_f56bb4df.log`, `xissues_recheck/*.xezim_f56bb4df.*`.
- **Reproduction steps:** `cd dv/eval/cases2`, then run the command given in the entry (or `./run_cases.sh <case>.sv` to run it on build A, build B and Verilator and store the logs).
- **Command:** `XEZIM_UVM_DIR=~/uvm XEZIM_UVM_VERSION=1.2 xezim [-uvm] -s <top> <file>`, unless stated; exact commands are in `cases2/results/<case>.xezim.cmd`.
- **Logs:** the complete logs are in `cases2/results/<case>.<simulator>.log`. The lines quoted below are the complete relevant output: the version banner (`=== xezim 0.11.1 ===`, `git <hash>`) and the closing `Simulation finished` line are omitted.
- **Exit status:** 0 unless stated. Every silent defect below exits 0.
- **Official references:** pinned to commit `83df0dd5`, base URL `https://github.com/aionhw/xezim/blob/83df0dd53506f12f58eff8cb87687423c86ebd78/`, written below as `<base>/path`.
- **Source references:** files in `src/` at the same commit.

**Table 8. Issue register**

| ID | Title | Class | Severity | Silent |
|---|---|---|---|---|
| XEZ-UVM-001 | `fork/join_none` children do not share the parent task's automatic variables | Confirmed | High | yes |
| XEZ-UVM-002 | Package-class local binds to a same-named top-level instance | Confirmed | High | yes |
| XEZ-UVM-003 | Static-method call through a handle binds to a same-named class | Confirmed | Medium | partly |
| XEZ-UVM-004 | Delays in packages without a timescale are mis-scaled under `-timescale` / `-override_timescale` | Conditional | High | yes |
| XEZ-UVM-005 | Character writes to a string grown in a loop are lost (breaks `uvm_packer` strings) | Confirmed | Medium | yes |
| XEZ-UVM-006 | Array size tied to a nested random object: the branch is never chosen | Confirmed | High | yes |
| XEZ-UVM-007 | `dist` produces exact weight quotas on every seed | Confirmed | Medium | yes |
| XEZ-UVM-008 | `.triggered` misses a trigger earlier in the same time step | Conditional | Medium | yes |
| XEZ-UVM-009 | Several instances of a non-embedded covergroup created in class code lose hits | Confirmed | Medium | yes |
| XEZ-UVM-010 | Embedded covergroup created in a UVM `run_phase` after a delay drops an instance | Conditional | Medium | yes |
| XEZ-UVM-011 | `cg_type::type_option` in a class method does not compile | Confirmed | Low | no |
| XEZ-UVM-012 | `$assertoff` naming an assertion inside a module instance or interface aborts the run | Conditional | Low | no |
| XEZ-UVM-013 | A dynamic-array member of an unpacked struct cannot be sized | Confirmed | Medium | yes |
| XEZ-UVM-014 | `UVM_ERROR` / `UVM_FATAL` do not change the exit status, even with `--error-exit` | Confirmed | High | yes |
| XEZ-UVM-015 | FST dump of the full UVM environment aborts the simulator | Conditional | Medium | no |
| XEZ-UVM-016 | IEEE 1735 protected envelopes are skipped without a diagnostic | Confirmed | High | yes |
| XEZ-UVM-017 | `.name()` on an enum returned by a UVM method prints the wrong name | Confirmed | Low | yes |
| XEZ-UVM-018 | No coverage merge, UCIS, report tool, condition or FSM coverage | Confirmed (documented) | High for signoff | n/a |
| XEZ-UVM-019 | Partial force through `uvm_hdl_force` is unsupported | Confirmed (documented) | Low | no |
| XEZ-UVM-020 to 029 | Ten language defects from the earlier evaluation (x01 to x10), still present on build B | Confirmed | Medium | yes |
| XEZ-UVM-030 to 032 | Three project-level observations not reduced | Unverified | | |
| XEZ-UVM-033 | Coverage database omits automatic bins of coverpoints and crosses, and never-hit bins | Confirmed (documented) | High for signoff | n/a |
| XEZ-UVM-034 | Non-ASCII characters in `$display` strings are re-encoded | Confirmed | Low | yes |

**PCIe relevance and adoption effect.** The table below adds, for each issue, the PCIe feature it affects and its effect on adoption. All issues reproduce on builds A, B and C (Fact: `cases2/results/*.xezim*.log`, `xissues_recheck/*.xezim_f56bb4df.log`).

Severity scale:
- **Critical:** blocks a use case outright.
- **High:** silent wrong result likely in PCIe code, or blocks regression/signoff without a workaround.
- **Medium:** silent wrong result in less common code, or a workaround is needed.
- **Low:** cosmetic, or not silent with a simple workaround.

Adoption effect:
- **Blocks:** prevents a use case.
- **Restricts:** usable with the stated workaround or coding rule.
- **Qualification:** needs more testing before relying on the feature.

**Table 9. Issue summary with PCIe relevance**

| ID | Severity | Required feature and why PCIe verification needs it | Class | Adoption effect |
|---|---|---|---|---|
| XEZ-UVM-001 | High | `fork/join_none` with shared locals: per-lane and per-tag monitor and driver processes | Confirmed | Restricts (coding rule; current env not affected) |
| XEZ-UVM-002 | High | Name resolution in packages: agents and adapters with short local names | Confirmed | Restricts (naming rule plus CI grep) |
| XEZ-UVM-003 | Medium | `uvm_do*` macros: every sequence | Confirmed | Restricts (naming rule) |
| XEZ-UVM-004 | High | Delays and drain times in packages under `-timescale`: timeouts, completion-timeout models | Conditional | Restricts (`timeunit` in every package) |
| XEZ-UVM-005 | Medium | `uvm_packer` with strings: transaction pack/unpack | Confirmed | Restricts (no string packing) |
| XEZ-UVM-006 | High | Array size from a nested random object: TLP length in a header object sizing the payload | Confirmed | Restricts (flat TLP classes) |
| XEZ-UVM-007 | Medium | `dist` randomness: TLP type mixes, error-injection rates | Confirmed | Qualification (distribution quality) |
| XEZ-UVM-008 | Medium | `.triggered` handshakes between components | Conditional | Restricts (levels or mailboxes) |
| XEZ-UVM-009 | Medium | Several covergroup instances: per-lane, per-VC, per-function coverage | Confirmed | Restricts (embedded covergroups) |
| XEZ-UVM-010 | Medium | Coverage objects created during `run_phase` | Conditional | Restricts (create in `build_phase`) |
| XEZ-UVM-011 | Low | Merged instance coverage options set at run time | Confirmed | Restricts |
| XEZ-UVM-012 | Low | Assertion control inside an SVA interface | Conditional | Restricts (control from top) |
| XEZ-UVM-013 | Medium | Structs with dynamic arrays: descriptor or TLP structs | Confirmed | Restricts (use classes) |
| XEZ-UVM-014 | High | Exit status: CI regression pass/fail | Confirmed | Restricts (log-based pass/fail, already in `../sim/regress.py`) |
| XEZ-UVM-015 | Medium | FST waveforms of the full environment | Conditional | Restricts (VCD or scoped FST) |
| XEZ-UVM-016 | Critical for VIP | IEEE 1735 decryption: encrypted commercial Root Complex VIP | Confirmed | **Blocks** encrypted-VIP use |
| XEZ-UVM-017 | Low | Enum `.name()` in messages | Confirmed | Restricts |
| XEZ-UVM-018 | High for signoff | Coverage merge, UCIS, code-coverage signoff | Confirmed (documented) | **Blocks** signoff coverage closure on Xezim alone |
| XEZ-UVM-019 | Low | Partial force: bit-level fault injection | Confirmed (documented) | Restricts |
| XEZ-UVM-020 to 029 | Medium | Queues, structs, vifs, clocking blocks: core agent code | Confirmed | Restricts (workarounds in `dv/`) |
| XEZ-UVM-030 to 032 | n/a | Project observations not reduced | Unverified | Qualification |
| XEZ-UVM-033 | High for signoff | Coverage database with automatic and cross bins: cross-coverage closure across a regression | Confirmed (documented) | **Blocks** merged cross-coverage closure on Xezim alone |
| XEZ-UVM-034 | Low | Non-ASCII text in `$display`: log readability, log parsing | Confirmed | Restricts (ASCII-only messages) |

### XEZ-UVM-001 · `fork/join_none` children do not share the parent task's automatic variables

- **Classification:** Confirmed limitation. Severity High. Silent.
- **Expected:** IEEE 1800-2017 9.3.2: processes spawned by `fork` share the automatic variables of the enclosing task. Eight children each incrementing a parent `int` leave it at 8. A child waiting on a parent variable sees the parent's later write.
- **Observed:**
  - the parent counter ends at 1 of 8
  - two children incrementing at different times leave 1 instead of 2
  - a child never sees the parent's write after the fork
  - class and module tasks behave the same
- **Reproducers:** `cases2/e_fork_parent_auto_var.sv`, `e_fork_two_children_share.sv`, `e_fork_parent_auto_read.sv`, `e_fork_parent_auto_module_task.sv`, `e_fork_parent_auto_read_module.sv`, `e_fork_loop_events_task.sv`.
- **Command:** `xezim -s t e_fork_parent_auto_var.sv`
- **Log (build A, exit 0):** `RESULT parent automatic counter: 1/8, class member counter: 8/8 (expect 8 8)`
- **Log (build A, exit 0):** `RESULT two children, one increment each: 1 (expect 2)` / `RESULT child saw parent write after fork: 0 (expect 1)`
- **Verilator (exit 0):** `8/8 ... 8/8`, `2`, `1`.
- **Reference:**
  - upstream commit `50999fbd` "Live-share fork-child writes to a suspended parent's automatic locals" states the design: "A fork/join_none child is given a COPY of the parent's locals, and its writes are merged back ... only when the child finishes".
  - Source: `src/compiler/simulator.rs` line 49859 (`merge_fork_writes`) and the design comment at line 7383.
  - The upstream regression test `tests/fork_child_blocking_share.sv` passes on builds A and B; it covers one child, one write.
  - Related upstream issue #16 (closed, "automatic Variables in Looped fork..join Blocks", https://github.com/aionhw/xezim/issues/16); this variant still fails on build C.
- **Root cause:** copy-in/merge-out semantics. A child's change is mirrored to the parent only when the child's copy differs from its fork-time snapshot. A second child, still holding the stale copy, writes back its own increment (lost update). Parent writes after the fork never reach the child's copy.
- **PCIe impact:**
  - Any driver, monitor or scoreboard that forks per-lane or per-tag processes sharing a task-local counter, flag or handle gets wrong results without warning.
  - The current `dv/` environment has one `join_none` (`agents/pipe/pipe_driver.svh` line 43). It shares no automatic variables, so it is not affected.
  - The UVM library uses this idiom in places. Upstream fixed one (`m_safe_select_item`), and the suite's sequencer, objection and event tests pass, but other library sites were not audited. [confirm]
- **Workaround:** share state between forked processes through class members, module variables or mailboxes, never through task-local automatic variables. Verified: the class-member counter reads 8/8 in the same reproducer, and U07-07 passes once it uses a member.

### XEZ-UVM-002 · Package-class local binds to a same-named top-level instance

- **Classification:** Confirmed limitation. Severity High. Silent.
- **Expected:** IEEE 1800-2017 26.3 and 23.9: code in a package cannot see module-scope names; `b.wr` inside a class method refers to the local handle `b`.
- **Observed:**
  - A local handle `b` in a package class, with the top module containing an interface instance also named `b`: writes to `b.wr` and `b.addr` go into the top-level interface.
  - Only members the interface does not have (`data`) reach the local object.
- **Reproducer:** `cases2/e_pkg_local_vs_top_instance.sv`. Command `xezim -s top e_pkg_local_vs_top_instance.sv`.
- **Log (build A, exit 0):** `RESULT item wr 0 addr 00 data 0000f00d (expect 1 0c f00d); top.b wr 1 addr 0c (expect 0 00)`
- **Verilator (exit 0):** `RESULT item wr 1 addr 0c data 0000f00d (expect 1 0c f00d); top.b wr 0 addr 00 (expect 0 00)`
- **Reference:** no documentation statement. Source location not isolated. Related upstream issue #271 (closed 2026-10-08, "Interface instance name collision makes scalar class-member equality comparisons incorrect", https://github.com/aionhw/xezim/issues/271): the same mechanism for a comparison; its fix predates build A and does not cover this write case, which still fails on build C.
- **Root cause:** name resolution of `<id>.<member>` prefers a top-level instance over a local variable. Inferred from behaviour; the code path was not identified.
- **PCIe impact:**
  - Found during the RAL test (U11): the bus adapter and monitor locals named `b` drove the top-level bus interface. Every register access went to address 0 until they were renamed.
  - The current `dv/` environment was checked: no local or member name matches a top-level instance name (`pipe_if`, `sb_if`, `s_axi_if`, `m_axi_if`, `probe_if`, `dut`, `u_sva`).
- **Workaround:** a naming rule: no class local or member may share a name with a top-level instance. Enforce it with a lint grep in CI. Verified: renaming the locals fixed U11 (14/14 pass).

### XEZ-UVM-003 · Static-method call through a handle binds to a same-named class

- **Classification:** Confirmed limitation. Severity Medium.
- **Expected:** `item t; t.who()` calls `item::who` (IEEE 1800-2017 8.10, 8.23).
- **Observed:** when a class named `t` exists, `t.who()` calls `t::who`.
  - In UVM this breaks `uvm_do*` and `uvm_create`, which call `SEQ_OR_ITEM.get_type()`.
  - The sequence logs `RNDFLD Randomization failed in uvm_do_with action` and drives nothing.
- **Reproducers:** `cases2/e_static_call_via_handle.sv`; UVM form `e_u05_member_n_original.sv` (U05 as first written, test class named `t`).
- **Command:** `xezim -s top e_static_call_via_handle.sv`; UVM form `xezim -uvm -s top --max-time 1ms +incdir+../uvm_suite e_u05_member_n_original.sv +UVM_TESTNAME=t`
- **Log (build A, exit 0):** `RESULT t.who() with 'item t' in scope: class t (expect class item)`. UVM form: `UVM_WARNING ... [RNDFLD] Randomization failed in uvm_do_with action`.
- **Verilator (exit 0):** `RESULT t.who() with 'item t' in scope: class item (expect class item)`
- **Reference:** none in the docs; source not isolated.
- **Root cause:** a class name takes precedence over a variable of the same name in method-call resolution. Inferred.
- **PCIe impact:** low in practice, because UVM code rarely names a class like a variable. It is silent, though, and was hit in this assessment. `dv/` does not use one-letter class names.
- **Workaround:** never give a class the same name as a variable in scope. Verified: renaming the test class to `u05_test` fixed it.

### XEZ-UVM-004 · Delays in packages without a timescale are mis-scaled under `-timescale` / `-override_timescale`

- **Classification:** Conditional limitation. Severity High when the condition holds. Silent.
- **Condition:** a package with no `timescale` directive or `timeunit` declaration, compiled with `-timescale 1ns/1ps` (the PCIe flow uses this) or `-override_timescale 1ns/1ps`. Without these options, or with a `timescale` directive or `timeunit` in the package, it works.
- **Expected:** a package delay `#100` lasts 100 ns, as on Verilator. `$time` in the package reads 100.
- **Observed:**
  - With `-timescale 1ns/1ps`, `#100` lasts 100 ps and `$time` reads 0.
  - With `-override_timescale 1ns/1ps`, `#100` lasts 100 us and `$time` reads 100000.
  - In UVM, `set_drain_time(this, 50)` becomes 50 ps: U01-07 fails under `-timescale 1ns/1ps` (`drain time 50 honoured (dropped 0, run ended 0)`) and passes without it.
- **Reproducers:** `cases2/e_pkg_time.sv` (controls `e_pkg_time_tsdir.sv`, `e_pkg_time_unit.sv`); `uvm_suite/results/timescale/u01_test_timescale_1ns_1ps.log`.
- **Command:** `xezim -timescale 1ns/1ps -s top e_pkg_time.sv`
- **Log (build A, exit 0):** `package class after #100: $time=0  $realtime=0.100000  %t=0` / `module after the package task:  $time=0  $realtime=0.100000  %t=0` / `module after its own #100:      $time=100  %t=100000`
- **Log, `-override_timescale 1ns/1ps` (exit 0):** `package class after #100: $time=100000  $realtime=100000.000000  %t=100000000`
- **Verilator (both options, exit 0):** `package class after #100: $time=100  $realtime=100.000000  %t=100000`
- **Reference:**
  - `<base>/docs/command-line.md#L120-L121`: `-timescale` sets "Default timescale for design elements without one"; `-override_timescale` sets "One timescale for every design element, package and compilation unit".
  - `<base>/docs/user-guide.md#L181`: the module-timescale extension applies to module definitions.
  - Source not isolated.
- **Root cause:** packages do not receive the command-line timescale consistently. Delays and time reads use different units. Inferred from the three variants.
- **PCIe impact:**
  - `dv/` compiles every package with `-timescale 1ns/1ps` and none declares a timescale, so the condition holds.
  - Today it has no effect: all class-level timing in `dv/` is clock-based, and it sets no drain time or phase timeout. The default UVM timeout is a time literal (`9200s`, `uvm_global_defines.svh` line 57), which scales correctly.
  - Any future `#` delay, `set_drain_time` or `set_timeout` in a package would be 1000 times too short.
- **Workaround:** put `timeunit 1ns; timeprecision 1ps;` in every testbench package. Verified: `e_pkg_time_unit.sv` gives the expected 100 ns. The UVM package itself cannot be edited; set drain times and timeouts with time literals (`50ns`) or avoid `-timescale` and declare timescales in source. [confirm the literal form for drain time]

### XEZ-UVM-005 · Character writes to a string grown in a loop are lost

- **Classification:** Confirmed limitation. Severity Medium. Silent.
- **Expected:** `r = {r, " "}; r[i] = src[i];` in a loop builds a copy of `src` (IEEE 1800-2017 6.16).
- **Observed:** the copy holds the wrong characters (`" " ( "` for `first`). Straight-line character writes work (`e_string_char_write.sv`).
- **UVM effect:** UVM 1.2 `uvm_packer::unpack_string` (`uvm_packer.svh` lines 1010 to 1035) and 1800.2 (lines 1161 to 1184) use exactly this loop. Unpacked strings come back corrupted, and U08-03 fails on all three UVM libraries.
- **Reproducers:** `cases2/e_fn_return_string_index.sv`, `e_packer_string.sv` (UVM).
- **Command:** `xezim -s t e_fn_return_string_index.sv`; UVM form `xezim -uvm -s top --max-time 1ms e_packer_string.sv`
- **Log (build A, exit 0):** `RESULT return-variable: " " ( "  local variable: " " ( "  class method return-variable: " " ( " (expect first first first)`; UVM: `RESULT unpack_string=" " ( " (len 5), next int=000000000000cafe (expect "first" len 5, cafe)`
- **Verilator (exit 0):** `RESULT return-variable: "first"  local variable: "first"  class method return-variable: "first"`
- **Reference:** none; source not isolated.
- **Root cause:** an index write to a string after the string was reassigned in the same loop iteration. Inferred.
- **PCIe impact:** any `pack`/`unpack` of a transaction with string fields, or `uvm_packer` use in a scoreboard or a VIP, is corrupted. `dv/` does not pack strings.
- **Workaround:** do not pack strings; pack byte arrays, or build strings with `$sformatf` / concatenation only.

### XEZ-UVM-006 · Array size tied to a nested random object: the branch is never chosen

- **Classification:** Confirmed limitation. Severity High. Silent.
- **Expected:** with `kind dist {0 := 1, 1 := 1}` and `kind == 1 -> arr.size() == h.n` (where `h` is a `rand` nested object), both kinds are feasible and each appears about half the time (IEEE 1800-2017 18.5.4, 18.5.8.1, 18.4).
- **Observed:**
  - `kind == 1` is never chosen (0 of 400), and `randomize()` reports success.
  - Forcing `kind == 1` inline fails on every call, with `[xezim][error] randomize budget exhausted in class c_nested_len; last failed constraints: __inline__, k`.
  - The same constraint with the size in a field of the same class works (200 of 400).
  - In the TLP-style test U16, MWR, CFGWR and CPLD (the kinds with a payload) were never generated in 500 randomizations (U16-02), and every one of 200 calls to `randomize() with { h.len <= local::lim; kind == MWR; }` failed (U16-05).
- **Reproducer:** `cases2/e_rand_size_implication.sv`. Command `xezim -s top e_rand_size_implication.sv`.
- **Log (build A, exit 0):** `RESULT nested length: kind==1 in 0/400 (expect about 200), violations/failures 0 (expect 0)` / `RESULT nested length, kind == 1 forced inline: 50/50 randomize failures, 0 violations (expect 0 0)` / `RESULT flat: kind==1 in 200/400 (expect about 200), violations/failures 0 (expect 0)`
- **Verilator (exit 0):** `RESULT nested length: kind==1 in 210/400 (expect about 200), violations/failures 0`. Verilator also mishandles the forced variant (29 failures, 12 violations), so only the distribution result is cross-checked.
- **Reference:** no documentation statement; source not isolated. Related open upstream issue #255 ("Issues with Randomizing Nested rand Class Handles ...", https://github.com/aionhw/xezim/issues/255). Build C contains randomization fixes for receiver-qualified `size()` in inline constraints (`NOTES.md`, PR #287), but this case still fails on build C.
- **Root cause:** the solver does not treat an array size that depends on a nested object's random variable as solvable. Inferred.
- **PCIe impact:**
  - A TLP generator whose length sits in a header sub-object would never produce payload-carrying TLPs, and nothing would report it.
  - Coverage would expose the gap, but only if the corresponding bins exist.
  - `dv/` keeps all TLP fields flat (`common/pcie_tlp.svh`), so it is not affected.
- **Workaround:** keep fields that size arrays in the same class as the array. Verified by the flat case and U16-01 (500 TLPs, 0 violations).

### XEZ-UVM-007 · `dist` produces exact weight quotas on every seed

- **Classification:** Confirmed limitation. Severity Medium. Silent.
- **Expected:** `d dist { 0 := 1, [1:10] :/ 9 }` draws 0 with probability 0.1, so the count of zeros in 1000 draws varies with the seed (IEEE 1800-2017 18.5.4).
- **Observed:** exactly 100 zeros on every seed tried (1 to 6 on build A, 1 to 3 on build B). The flat case in `e_rand_size_implication.sv` also gives exactly 200 of 400.
- **Reproducer:** `cases/c_dist.sv`, command `xezim c_dist.sv +seed=<n>`. Logs: `cases/c_dist.result`, `xissues_recheck/c_exit_dist.xezim_head.txt`.
- **Log (build B, exit 0):** `seed 1: zeros in 1000: 100` / `seed 2: zeros in 1000: 100` / `seed 3: zeros in 1000: 100`
- **Verilator (exit 0):** 89, 105, 102 on seeds 1 to 3.
- **Reference:** none; source not isolated.
- **Root cause:** quota-based (deterministic) weighting instead of random sampling. Inferred.
- **PCIe impact:** random stimulus is less random than the testbench author intends. Rare-event combinations that depend on runs of the same value occur less often than on other simulators. Bugs that need such runs may go unfound.
- **Workaround:** none in the tool. For critical distributions, draw with `$urandom_range` and explicit thresholds. [confirm with upstream whether this is intended]

### XEZ-UVM-008 · `.triggered` misses a trigger earlier in the same time step

- **Classification:** Conditional limitation. Severity Medium. Silent.
- **Condition:** the trigger runs before the waiting process in the same time step. In the other order it works. U07-06 (`uvm_event::wait_ptrigger`) passes with its scheduling order.
- **Expected:** `wait(e.triggered)` returns if `e` was triggered earlier in the same time step (IEEE 1800-2017 15.5.3).
- **Observed:** when the trigger comes first, the waiter does not wake (0 of 1).
- **Reproducers:** `cases2/e_event_ptrigger_race.sv`; UVM form `e_uvm_event_ptrigger.sv` (the run ends at time 5 without the waiter's message, exit 0).
- **Command:** `xezim -s t e_event_ptrigger_race.sv`
- **Log (build A, exit 0):** `RESULT trigger-first woke=0, wait-first woke=1 (expect 1 1)`
- **Verilator (exit 0):** `RESULT trigger-first woke=1, wait-first woke=1 (expect 1 1)`
- **Reference:** related upstream issues #68 and #77 (closed, `.triggered` in event controls); this variant still fails on build C. The trigger-time state is `event_triggered_time` in `src/compiler/simulator.rs` line 7412. The failing comparison was not isolated.
- **PCIe impact:** `uvm_event::wait_ptrigger` and `@(e.triggered)` handshakes between components can miss events depending on process order. That would hang or skip a check. `dv/` does not use `wait_ptrigger` (its completion tracker polls).
- **Workaround:** use a level (a status bit or `uvm_event::is_on()` polled at the clock) or a mailbox instead of `.triggered` across processes.

### XEZ-UVM-009 · Several instances of a non-embedded covergroup created in class code lose hits

- **Classification:** Confirmed limitation. Severity Medium. Silent.
- **Expected:** each `new()` of a covergroup is a separate instance with its own hits (IEEE 1800-2017 19.8). Two instances sampled with 0 and 15 report 50% each.
- **Observed:**
  - With a package-level covergroup created twice inside a class method, the first instance reports 0%.
  - Held in a queue, both instances read 0%.
  - The same code in a module works (U09-11 fails).
- **Reproducers:** `cases2/e_cg_two_inst_class.sv`, `e_cg_two_inst_variants.sv`, `e_cg_pkg_noargs_class.sv`.
- **Command:** `xezim -s top e_cg_pkg_noargs_class.sv`; suite form U09-11 in `uvm_suite/u09_coverage.sv` (`./run_suite.sh tests xezim`)
- **Log (build A, exit 0):** `RESULT package cg (no args) in class method: a 0.00 b 50.00 (expect 50.00 50.00)` and `RESULT class method, 2 instances: a 0.00 b 50.00 type 0.00 (expect 50.00 50.00 50.00)`
- **Verilator (exit 0):** `RESULT package cg (no args) in class method: a 50.00 b 50.00`. Verilator's covergroup support is partial elsewhere, so the classification rests on the IEEE standard.
- **Reference:** `<base>/docs/coverage-guide.md#L270-L320` ("Every `new()` creates a separate instance with its own hits"). Source not isolated.
- **PCIe impact:** a covergroup declared in a package and instantiated per lane, per function or per virtual channel from a UVM component loses per-instance data. Coverage closure numbers are then wrong. `dv/` uses embedded covergroups with one instance each (`env/pcie_coverage.svh`), which are not affected.
- **Workaround:** declare covergroups inside the coverage class (embedded) and create one collector object per instance, held in a class member. Verified: U09-11c passes.

### XEZ-UVM-010 · Embedded covergroup created in a UVM `run_phase` after a delay drops an instance

- **Classification:** Conditional limitation. Severity Medium. Silent.
- **Condition:**
  - Two objects with an embedded covergroup are created inside a UVM `run_phase` after any delay (`#0` is enough) and held in task-local handles.
  - Without the delay, or with the handles held in class members, it works.
  - It was not reproduced without UVM.
- **Expected:** both instances record their own hits (50%, 50%).
- **Observed:**
  - The second instance reports 0%.
  - The coverage database contains only one instance of the type.
  - U09-11b fails and U09-11c (class members) passes.
- **Reproducers:** `cases2/e_uvm_cg_task_local_after_delay.sv`; controls `e_uvm_cg_task_local_after_delay_members.sv` and `e_uvm_cg_task_local_nodelay.sv`.
- **Command:** `xezim -uvm -s top --max-time 1ms e_uvm_cg_task_local_after_delay.sv`
- **Log (build A, exit 0):** `RESULT uvm run_phase locals: 50.00 0.00`; controls: `RESULT uvm run_phase locals: 50.00 50.00`. Coverage JSON: one `col::cg` entry, `samples 1`.
- **Reference:** none. Root cause not isolated; possibly related to XEZ-UVM-001, because `run_phase` executes in a forked process. [confirm]
- **PCIe impact:** coverage objects created during the run (per stream or per transaction class) can lose data. Collectors built in `build_phase` as component members, the usual UVM style and the style in `dv/`, are not affected.
- **Workaround:** create covergroup holders in `build_phase` or as class members.

### XEZ-UVM-011 · `cg_type::type_option` in a class method does not compile

- **Classification:** Confirmed limitation. Severity Low. Not silent: compile error, exit 1.
- **Expected:** `val_cg::type_option.merge_instances = 1;` is legal in any procedural code that can see `val_cg`. The documentation shows it as supported.
- **Observed:** in a class method, `error: Undeclared identifier 'val_cg'`. In a module it works.
- **Reproducers:** `cases2/e_cg_type_option_scope.sv`, control `e_cg_type_option_module.sv`.
- **Command:** `xezim -s top e_cg_type_option_scope.sv`
- **Log (build A, exit 1):** `e_cg_type_option_scope.sv:12:7: error: Undeclared identifier 'val_cg'`
- **Reference:** `<base>/docs/coverage-guide.md#L292` ("`cg_type::type_option.merge_instances = 1;` at run time").
- **PCIe impact:** small; merged coverage across instances must be configured another way.
- **Workaround:** set `type_option.merge_instances = 1;` in the covergroup declaration. Verified: U09-12 passes.

### XEZ-UVM-012 · `$assertoff` naming an assertion inside a module instance or interface aborts the run

- **Classification:** Conditional limitation. Severity Low. Not silent: `Simulation error`, exit 1.
- **Condition:** the call uses a bare assertion label from inside a non-top module or an interface. From the top module, a bare name in the top or a hierarchical name into an instance (`$assertoff(0, u.a_off)`) works.
- **Expected:** IEEE 1800-2017 20.12 allows naming assertions in the list.
- **Observed:** the run aborts at time 0 before any assertion is evaluated.
- **Command:** `xezim -s top e_assertoff_in_module.sv`
- **Log (builds A and B, exit 1):** `Simulation error: Undeclared identifier 'a_off' at e_assertoff_in_module.sv:7:25`. Control `e_assertoff_in_top.sv` (exit 0): `RESULT bare_t0: a_on 4 a_off 0 (expect 4 0)`.
- **Reproducers:** `cases2/e_assertoff_in_module.sv`, `e_assertoff_in_interface.sv`; controls `e_assertoff_in_top.sv`, `e_assertoff_hier.sv`.
- **Reference:** `<base>/docs/features.md#L23-L27` (assertion control with "a list of scopes or assertion names"). Entry point `src/compiler/simulator/assert_ctl.rs` line 327. Failing lookup not isolated.
- **PCIe impact:** assertion control inside the SVA interface or bind module is not possible; control must come from the top. Verilator cannot be used as a reference here (it reports no assertion failures).
- **Workaround:** issue `$assertoff`/`$asserton` from the top module with hierarchical names. Verified: U10-08 passes.

### XEZ-UVM-013 · A dynamic-array member of an unpacked struct cannot be sized

- **Classification:** Confirmed limitation. Severity Medium. Silent.
- **Expected:** `c.slices = new[1];` on a struct member gives `c.slices.size() == 1`, and struct copies carry the array (IEEE 1800-2017 7.2, 7.5).
- **Observed:** size 0 after `new[1]`, and 0 in every copy.
- **Reproducer:** `cases2/e_struct_dynarray_copy.sv`.
- **Command:** `xezim -s top e_struct_dynarray_copy.sv`
- **Log (build A, exit 0):** `RESULT sizes: c 0, d=c 0, q[0] 0, decl-init e=q[0] 0, f=q[0] 0 (expect 1 1 1 1 1)`
- **Verilator (exit 0):** `RESULT sizes: c 1, d=c 1, q[0] 1, decl-init e=q[0] 1, f=q[0] 1`
- **Reference:** none; source not isolated.
- **PCIe impact:** a TLP or descriptor modeled as a struct with a dynamic payload array would carry no payload. UVM's own `uvm_hdl_path_concat` is a class (`uvm_reg_model.svh` line 346), so the UVM backdoor is not affected (U11 and U12 pass). `dv/` models TLPs as classes.
- **Workaround:** use a class, or a fixed-size array, for structures with variable-length data.

### XEZ-UVM-014 · `UVM_ERROR` / `UVM_FATAL` do not change the exit status, even with `--error-exit`

- **Classification:** Confirmed limitation. Severity High for regression. Silent at the process level.
- **Expected:** the docs say `--error-exit` makes the run "Exit nonzero if any `$error` was reported (`$fatal` always does)". A UVM regression needs failing tests to exit nonzero.
- **Observed:**
  - exit 0 for `uvm_error` and for `uvm_fatal`, with or without `--error-exit`
  - exit 0 for an unknown `+UVM_TESTNAME` (`UVM_FATAL ... [INVTST]`)
  - exit 0 for U10 with 5 `UVM_ERROR`s under `--error-exit`
  - `$error` with `--error-exit` and `$fatal` do exit 1
- **Reproducers:** `cases/c_exit.sv` (`+KIND=uvm_error|uvm_fatal|sv_error|sv_fatal`); `uvm_suite/results/<build>/neg_unknown_test.*`, `neg_uvm_error_exit.*`.
- **Log (build B):** `KIND=uvm_error --error-exit -> exit 0` / `KIND=uvm_fatal --error-exit -> exit 0` / `KIND=sv_error --error-exit -> exit 1` / `KIND=sv_fatal (default) -> exit 1`; `neg_unknown_test.log`: `UVM_FATAL @ 0: reporter [INVTST] Requested test from command line +UVM_TESTNAME=no_such_test not found.`, exit 0.
- **Reference:** `<base>/docs/command-line.md#L55`. Related upstream issue #107 (closed, "Exit code is 0 even when errors are reported"): fixed for `$error` under `--error-exit`, not for UVM reports. Unchanged on build C (`xissues_recheck/c_exit_dist.xezim_f56bb4df.txt`).
- **Root cause:** UVM reports are not `$error`/`$fatal` calls. After a fatal, UVM 1.2 calls `$finish` (`uvm_root.svh` line 135, in `die()`), which xezim treats as a normal end.
- **PCIe impact:** a regression system that trusts exit codes reports every failing test as passing. The PCIe flow already parses `[RESULT]` lines and the report summary (`sim/regress.py`).
- **Workaround:** pass or fail from the log, using the `UVM_ERROR :` and `UVM_FATAL :` summary counts and a test-printed result line. Verified in `sim/regress.py`. Alternatively, call `$fatal` from `report_phase` when the counts are nonzero. [confirm whether a `report_phase` `$fatal` is preferred]

### XEZ-UVM-015 · FST dump of the full UVM environment aborts the simulator

- **Classification:** Conditional limitation. Severity Medium. Not silent: abort, exit 134.
- **Expected:** the run completes and writes an FST file, as it does with VCD (`--wave`) on the same test.
- **Condition:** `--fst` on the whole PCIe UVM environment. VCD works, and FST limited to the DUT or a single interface works (earlier evaluation, `logs/fst_scope_dut`, `logs/fst_pipe_if` and others).
- **Observed (build B, rechecked):** `thread 'xezim-fst' panicked at .../fst-writer-0.3.1/src/buffer.rs:117:17: Failed to parse four state value: 00000000000000000000000011101110 for signal of size 1`, exit 134.
- **Reproducer:** the full PCIe environment (`dv/sim`, test `pcie_link_up_test`, seed 1). Not reduced to a minimal case.
- **Command:** in `dv/sim`, `xezim -uvm -F rtl.f -F tb.f -s pcie_tb_top -timescale 1ns/1ps --max-time 50ms +UVM_TESTNAME=pcie_link_up_test +seed=1 --fst r1.fst`
- **Logs:** `xissues_recheck/fst_full.xezim_head.log`; build A: `logs/fst_repro_1/`.
- **Reference:** the source frame is `src/compiler/fst_sink.rs` line 390. A signal declared 1 bit wide is written with a 32-bit value.
- **PCIe impact:** waveform debug of the full testbench needs VCD (larger files) or scoped FST.
- **Workaround:** `--wave` (VCD), or `--fst` with `--fst-scope` on the DUT or interfaces.

### XEZ-UVM-016 · IEEE 1735 protected envelopes are skipped without a diagnostic

- **Classification:** Confirmed limitation. Severity High for VIP. Silent.
- **Expected:** a simulator either decrypts a `pragma protect` envelope or reports that it cannot.
- **Observed:** the envelope's contents are dropped, the rest of the file runs, no message is printed, and the exit status is 0.
- **Reproducer:** `cases/c_protect.sv`. Command: `xezim c_protect.sv`. Log (build B): `before envelope` / `after envelope`, exit 0. Log: `xissues_recheck/c_protect.xezim_head.log`.
- **Reference:** no documentation statement found. [confirm]
- **PCIe impact:** encrypted commercial PCIe VIP cannot run, and a partially encrypted model would run with parts silently missing.
- **Workaround:** none in the tool. Use unencrypted or open-source models, or run VIP-based tests on a commercial simulator.

### XEZ-UVM-017 · `.name()` on an enum returned by a UVM method prints the wrong name

- **Classification:** Confirmed limitation (UVM-level reproducer only). Severity Low. Silent.
- **Expected:** `.name()` returns the name of the value in its own enum type: `UVM_WARNING` (IEEE 1800-2017 6.19.5.6).
- **Observed:** in a report catcher, `get_severity().name()` prints `UVM_PHASE_DORMANT` for `UVM_WARNING`. The same value in a typed variable prints correctly. Equivalent code without UVM works (`e_enum_name_min.sv`, `e_enum_name_pkg.sv`).
- **Reproducer:** `cases2/e_enum_name.sv`. Command: `xezim -uvm -s top --max-time 1ms e_enum_name.sv`. Log (exit 0): `RESULT get_severity().name()=UVM_PHASE_DORMANT  local var .name()=UVM_WARNING  value=1 (expect UVM_WARNING UVM_WARNING 1)`
- **Root cause:** `.name()` on a call result resolves against the wrong enum type, likely the first enum with that value. Not isolated.
- **PCIe impact:** misleading log messages only.
- **Workaround:** assign to a typed variable before calling `.name()`.

### XEZ-UVM-018 · No coverage merge, UCIS, report tool, condition or FSM coverage

- **Classification:** Confirmed limitation (documented). Severity High for signoff.
- **Statement:** `<base>/docs/coverage-guide.md#L848-L862`: "Not supported at all: Condition, expression and FSM coverage ... Coverage databases in other formats (such as UCIS), text or HTML coverage reports, and merging results across runs". The `$coverage_*` system functions are ignored.
- **Related:** XEZ-UVM-033 (content of the coverage database).
- **Verified:**
  - Reproducer `cases2/e_cov_system_functions.sv`, command `xezim -s top e_cov_system_functions.sv`.
  - Log (builds A and B, exit 0): `Warning: unknown system task '$coverage_merge' ignored (first use at e_cov_system_functions.sv:6:25)` / `RESULT $coverage_merge returned 0` / `RESULT $coverage_save returned 0`.
  - Per-run JSON files are written. `dv/sim/cov_merge.py` merges functional coverage across runs as a project-side workaround.
  - Condition, expression and FSM coverage were not tested beyond this documentation statement.
- **PCIe impact:** coverage closure for signoff needs a merged database with exclusions and reports. Today that is project scripting, not a tool feature.
- **Workaround:** project merge scripts for functional coverage; a commercial simulator for code-coverage signoff.

### XEZ-UVM-019 · Partial force through `uvm_hdl_force` is unsupported

- **Classification:** Confirmed limitation (documented, verified). Severity Low. Not silent.
- **Statement:** `<base>/docs/uvm-guide.md#L189-L190`.
- **Observed:** `uvm_hdl_force("top.d.r[3:0]", 0)` returns 0 and prints `[DPI] error: uvm_hdl_force("top.d.r[3:0]"): forcing part of a signal is not supported; nothing was forced` (U12-12). Whole-signal force, release and deposit work (U12-06 to U12-10).
- **Reproducer:** check U12-12 in `uvm_suite/u12_hdl.sv`. Command: `XEZIM_UVM_DIR=~/uvm XEZIM_UVM_VERSION=1.2 xezim -uvm -s top --max-time 10ms u12_hdl.sv +UVM_TESTNAME=u12_test`. Log: `uvm_suite/results/<build>/u12_test.log` (exit 0), line `CHECK U12-12 PASS documented limitation: partial force returns 0`.
- **PCIe impact:** fault injection on a field of a wider register (for example one bit of an LTSSM state vector) cannot be forced from the testbench; the whole signal must be forced.
- **Workaround:** force the whole signal with the bits to keep re-read and merged.

### XEZ-UVM-020 to XEZ-UVM-029 · Language defects carried from the earlier evaluation

These ten defects are documented in full in `xezim_evaluation.md` §07 (XZ-01 to XZ-10). Each has a minimal reproducer in `dv/docs/xezim_issues/` and a Verilator cross-check in `cases/verilator_crosscheck.txt`. All ten were re-run on build B (`xissues_recheck/<case>.xezim_head.log`) and still fail identically. Each is worked around in `dv/` with a comment at the workaround.

**Table 10. Language defects carried from the earlier evaluation**

| ID | Earlier ID | Defect | Build B result (exit 0) | Workaround |
|---|---|---|---|---|
| XEZ-UVM-020 | x01 | `q[$] != v` in an `always` condition is always false | `size=1 (expect 3)` | `q[q.size()-1]` |
| XEZ-UVM-021 | x02 | Part-select write to a member of a packed struct declared in a loop body is lost | `struct in loop: 00` vs plain vector `fb` | build in plain vectors |
| XEZ-UVM-022 | x03 | Class specialized with a virtual-interface type differs when spelled through a typedef: config_db set/get miss | `set_ok=0 (expect 1)` | one vif typedef package |
| XEZ-UVM-023 | x04 | Signals read through a local virtual-interface variable read 0 | `assigned: w=0...0` | vifs as class members |
| XEZ-UVM-024 | x05, x05b | Derived member resolved to a base `local static` (a `uvm_reg_block` property named `id` stays null) | `derived.id null=1 (expect 0)` | rename (`id_reg`) |
| XEZ-UVM-025 | x06 | Unpacked-array `output` argument of a class method not copied back | `h[0]=xxxxxxxx (expect 00000011)` | return a packed value |
| XEZ-UVM-026 | x07 | Queue-returning call passed directly as a queue argument arrives empty | `sz(mk()) = 0 (expect 2)` | assign to a variable first |
| XEZ-UVM-027 | x09 | Left-hand-side concatenation of class properties mis-assigns | `a=0001 ... (expect 1234 56 7 8)` | slice by slice |
| XEZ-UVM-028 | x10 | Class function returning a queue of structs returns an empty queue | `0 chunks (expect 4)` | fill a member queue |
| XEZ-UVM-029 | x08 | Element select of a packed 2-D clocking-block input returns 0 | `cb[1]=0 (expect 22222222)` | read the whole vector |

Commands: `xezim <case>.sv` (x05 also `-uvm`). PCIe impact: all are silent, and all were hit while building `dv/`. x03, x04 and x08 sit on the virtual-interface and clocking-block path every agent uses.

### XEZ-UVM-033 · The coverage database omits automatic bins, cross bins and never-hit bins

- **Classification:** Confirmed limitation (documented, verified). Severity High for signoff coverage closure. Not silent (documented).
- **Required feature:** coverage closure across a regression needs every bin, hit or not, of every coverpoint and cross in a mergeable database. PCIe cross coverage (for example TLP type x length class x byte-enable pattern) is mostly automatic cross bins.
- **Statement:** `<base>/docs/coverage-guide.md#L409-L416`: "Things the file does not hold for functional coverage: ... Bins that were never hit ... Automatic bins, of a coverpoint or a cross."
- **Expected (for signoff use):** a per-run database holding all bins, so that a merge can compute cross coverage over the regression.
- **Observed:** in `uvm_suite/results/xezim/u09_test.cov.json`, the covergroup `cov::cg` holds `crosses: {'x_named': 5, 'x_op_mode': 5}` (sample counts only) and one cross bin, `x_named.rd_any` (a named bin). The eight automatic bins of `x_op_mode`, which U09-06 reports in-run as 3/8 covered, are absent.
- **Reproducer and command:** `cd uvm_suite && ./run_suite.sh tests xezim` (U09), then `python3 -c "import json; d=json.load(open('results/xezim/u09_test.cov.json')); print([g['crosses'] for g in d['covergroups'] if g['name']=='cov::cg'])"`. Exit 0.
- **Log:** `{'x_named': 5, 'x_op_mode': 5}`; bins with prefix `x_`: `{'x_named.rd_any': 3}`.
- **PCIe impact:** in-run coverage queries are correct (U09-06, U09-07), but merged cross coverage over a regression cannot be computed from Xezim's files. `../sim/cov_merge.py` merges coverpoint bins only and reports "crosses not merged" (`../docs/results/coverage_original.txt`).
- **Workaround:** declare the important cross combinations as named cross bins (they are written to the database), or report per-run percentages from the testbench with the query functions. Signoff cross coverage needs another tool.
- **Adoption effect:** blocks merged cross-coverage signoff on Xezim alone; does not affect development use.

### XEZ-UVM-034 · Non-ASCII characters in `$display` strings are re-encoded

- **Classification:** Confirmed limitation. Severity Low. Silent.
- **Required feature:** log messages and parsers that match message text. The repository's own bench `tb/tb_pcie_top.sv` prints the non-ASCII characters U+2013 (en dash) and U+2192 (right arrow).
- **Expected:** a string literal is a sequence of bytes (IEEE 1800-2017 5.9); `$display` writes the bytes unchanged. Icarus 12.0 and Verilator 5.050 print `e2 80 93` for U+2013.
- **Observed:** Xezim prints `c3 a2 c2 80 c2 93`: each byte is re-encoded as if it were a Latin-1 character.
- **Reproducer:** `cases2/e_utf8_string_literal.sv`. Command: `xezim -s t e_utf8_string_literal.sv | od -An -tx1`.
- **Log (builds A, B, C, exit 0):** bytes `52 45 53 55 4c 54 20 30 c3 a2 c2 80 c2 93 35 0a`. Verilator (exit 0): `52 45 53 55 4c 54 20 30 e2 80 93 35 0a`. Icarus: `e2 80 93`.
- **Project evidence:** `pcie_suitability/logs/tb.xezim.log` shows `[PASS] MPS field valid (0â5)` where Icarus prints the original U+2013 character between 0 and 5.
- **Reference:** no documentation statement; source not isolated.
- **PCIe impact:** cosmetic, but a log parser that matches non-ASCII text gives different results on Xezim and other simulators.
- **Workaround:** keep testbench messages ASCII.

### XEZ-UVM-030 to XEZ-UVM-032 · Project-level observations not reduced (Unverified)

**Table 11. Project-level observations not reduced**

| ID | Earlier ID | Observation | Status |
|---|---|---|---|
| XEZ-UVM-030 | XZ-12 | In the RAL frontdoor, a member named `timeout` read in a loop condition took a different value; renaming it to `max_wait` fixed it | Reproduced in the project at the time; no minimal reproducer. Possibly the same mechanism as XEZ-UVM-024 (name resolution against a base or library member). [confirm] |
| XEZ-UVM-031 | XZ-13 | Assigning a property of a freshly created sequence in `pcie_host_model` was lost | No minimal reproducer. The code forks per-request processes; XEZ-UVM-001 is a plausible cause. [confirm] |
| XEZ-UVM-032 | XZ-14 | Creating or resetting `uvm_event`s held in an associative array appeared to corrupt unrelated objects | Not reproduced: U07-07 (64 `uvm_event`s in an associative array) passes. The first version of U07-07 failed because of XEZ-UVM-001, so the original observation was probably that defect. |

## 07 Regression reliability, debug, waveforms, performance and scalability

**Determinism (Passed).**
- Seed 7 twice gives byte-identical logs and coverage JSON; seed 8 changes the random stream (`results/<build>/determinism.txt`).
- This matches the documentation: `<base>/docs/command-line.md#L29`.
- The earlier evaluation found two full PCIe baselines identical on all 26 common runs.

**Parallel runs (Passed).** Eight suite tests run concurrently produce the same CHECK lines and exit status as the serial runs on both builds (`results/<build>/parallel.txt`).

**Negative controls.**

**Table 12. Negative controls**

| Control | Result | Exit |
|---|---|---|
| Missing `--dpi-lib` file | `Simulation error: --dpi-lib ... could not be loaded: ... No such file or directory` | 1 (as documented) |
| DPI import with no implementation | `** Fatal: DPI import 'u13_not_defined_anywhere' has no implementation` | 1 (as documented). The statement after the call still printed `value 0` before the run ended. |
| Syntax error | `error: expected Semicolon, found KwEnd 'end'` with file, line and caret | 1 |
| Unknown `+UVM_TESTNAME` | `UVM_FATAL ... [INVTST]` | 0 (XEZ-UVM-014) |
| 5 `UVM_ERROR` with `--error-exit` | report summary `UVM_ERROR : 5` | 0 (XEZ-UVM-014) |

**Debug.**
- Diagnostics carry file, line and a caret.
- `XEZIM_VALUE_TRACE` (a per-signal change trace with the writer's file and line) was the most effective debug aid in the PCIe project.
- There is no interactive debugger; the `-do` language supports run and quit only (earlier evaluation, XZ-24).

**Waveforms.** VCD is correct. FST works on scoped dumps and aborts on the full UVM environment (XEZ-UVM-015).

**Performance** (build A, UVM 1.2, single runs on a shared host; wall time and peak resident memory from `/usr/bin/time -v`):

**Table 13. Performance and scalability**

| Workload | Wall time | Peak memory |
|---|---|---|
| Sequence items through sequencer, driver, analysis port: 1,000 | 2.5 s | 140 MB |
| 5,000 items | 8.8 s | 184 MB |
| 20,000 items | 32.9 s | 332 MB |
| Components with config_db lookup and objection: 100 | 2.2 s | 154 MB |
| 500 components | 9.9 s | 243 MB |
| 1,000 components | 24.1 s | 363 MB |
| 2,000 components | 70.3 s | 573 MB |
| RAL suite (U11, 14 checks incl. bit-bash) | 4.9 s | 173 MB |
| PCIe `pcie_link_up_test` (earlier evaluation) | 7.0 s simulation, 0.25 s compile | 191 MB |

Build B is within about 6% of build A on every row (`results/xezim_head/scale_*`).

Conclusions:
- Item throughput is linear at about 1.6 ms per item.
- Memory grows about 10 KB per item. The cause was not investigated. [confirm]
- Component count grows faster than linearly: doubling from 1,000 to 2,000 components multiplies wall time by 2.9.
- The PCIe environment has a few dozen components, far below where this matters. A large SoC-level environment with thousands of components would feel it.
- No commercial-simulator numbers exist for comparison, so no relative performance claim is made.

## 08 PCIe-specific compatibility assessment

**Verification levels.**

**Table 14. Verification levels**

| Level | Description | Benches |
|---|---|---|
| Transaction Layer only | Checks TLP formation and decode at the TL to DLL hand-off, without depending on link behaviour | The `dv/` probe agent observes this hand-off directly (white-box). The reference model and the `pcie_tlp` class self-test (200 random 3DW/4DW round trips) run at this level. |
| Full controller | Drives the PIPE lanes through a link-partner model (LTSSM, DLL framing, LCRC, sequence numbers) and the AXI interfaces | `dv/` environment, `tb/` with the Root Complex BFM, `uvm_tb/` |
| Full chip | Controller integrated in an SoC with interconnect, firmware and PHY | Not available in the repository: Untested |

**Table 15. PCIe scenarios on Xezim**

The "Runs on Xezim" column is about the tool; the RTL result column is about the design. A FAIL there is a DUT defect found by the checks, listed in `../docs/verification_plan.md` §6.

| Scenario | RTL present | Bench and tests | Runs on Xezim | Checks effective on Xezim | RTL result (original RTL) |
|---|---|---|---|---|---|
| TLP transmit (AXI write/read to MWr/MRd) | yes | `dv` axi_write, axi_read, axi_random x5; `tb` TEST 5/6 | Yes, A, B, C identical | Yes: mutation M11 killed, ref-model self-test pass | FAIL: DUT defects (W never accepted, MWr sent as MRd). `tb` TEST 5 stalls on Icarus and Xezim alike. |
| TLP receive (inbound MWr/MRd, CplD) | yes | `dv` rx_mem, stress | Yes | Yes | FAIL: no AXI write, no CplD |
| 3DW/4DW headers, payload, byte enables, tags | yes | `pcie_tlp` pack/unpack self-test; reference model MWr32/MWr64 selection; completion tag matching | Yes | Yes: tb_selftest 200 round trips PASS | blocked or FAIL behind the TX defects |
| Ordering rules | partial | none | n/a | n/a | Untested |
| Completion handling and error responses | yes | `dv` cfg_rw, rx_ur, rx_malformed | Yes | Yes | FAIL: no completions, UR not reported |
| Malformed TLPs | yes | `dv` rx_malformed (injection) | Yes | Yes | FAIL |
| DLLP and Data Link Layer (ACK/NAK, LCRC, sequence) | yes | `dv` rx_dll, dll_checker, pipe agent injection | Yes | Yes: mutations M7, M8 killed | FAIL: no NAK; spurious ACK |
| Replay, flow-control credits | yes (RTL modules) | not planned (VP-DLL-06) | n/a | n/a | Untested |
| LTSSM and link training | yes | `dv` link_up, link_stability, reset_retrain; `tb` TEST 1/2 | Yes; identical timeline to Icarus | Yes: mutations M1, M2, M6 killed | Detect to L0 PASS; Recovery and low-power states not reachable |
| DMA H2D/D2H data integrity and status | yes | `dv` dma, stress; `tb` TEST 7/8 | Yes | Yes | FAIL: no dma_done |
| Reset, configuration, timeout | yes | `dv` reset_retrain, cfg_defaults, cfg_rw; completion timeout not planned | Yes | Yes: mutations M4, M5 killed | FAIL: config reset values |
| Retry and error injection | yes | LCRC, sequence and EDB injection in the pipe agent | Yes | Yes | FAIL |
| Assertions | yes | `tb/pcie_dv_sva.sv` | Yes | Yes: mutations M3, M12 killed by SVA | PASS on AXI/status properties |
| Functional coverage | n/a | `env/pcie_coverage.svh`, 8 covergroups | Yes; embedded single instances, not affected by 009/010 | per-run; crosses not merged (033) | 47.2% merged coverpoint bins |
| Scoreboards and constrained-random regression | n/a | ref model and scoreboard; axi_random x5, stress x3 | Yes; deterministic per seed | Yes | FAIL behind DUT defects |
| Root Complex BFM (`tb/pcie_rc_bfm.sv`) | n/a | `tb/` | Yes; same results as Icarus | Yes | TEST 1 to 4 PASS, TEST 5 stalls |
| RAL | n/a | `dv` RAL model, frontdoor and backdoor | Yes (frontdoor after a naming workaround) | Yes | FAIL: DUT never completes CfgRd0 |
| External dependencies | n/a | UVM library (Accellera, open source), Xezim's built-in UVM DPI | Yes | n/a | n/a |
| Commercial PCIe Root Complex VIP | n/a | none in the repository | n/a | n/a | Untested; blocked if encrypted (016) |

**Judgment.** Xezim executes every PCIe scenario the repository's benches exercise, on all three builds. The checks are effective on Xezim: they kill 12 of 12 injected defects and agree with Icarus on the shared bench. The failing results come from the RTL. No PCIe scenario was found that Xezim cannot run. The gaps are coverage closure, signoff infrastructure, encrypted VIP and the scenarios no bench exercises yet (ordering, replay, flow control, completion timeout, low-power states).

### Project impact

**Applicability to PCIe Gen5 and Gen7.** All limitations in this report concern SystemVerilog language semantics, the UVM library, coverage, assertions or tool infrastructure. None depends on the PCIe generation, link speed or encoding. They apply equally to a Gen5 or a Gen7 UVM environment that uses the affected construct. Only the Gen7 project in this repository was exercised; no Gen5 environment and no commercial PCIe VIP was run, so generation-specific effects (for example of FLIT-mode data structures) are Unverified.

**Table 16. PCIe project impact**

| Area | Impact today | Impact as the project grows |
|---|---|---|
| Current `dv/` environment | Not affected by any confirmed limitation (checked individually for 001, 002, 004, 006, 008, 009, 010, 013) or worked around (020 to 029). Its results stand. | |
| New testbench code | | Silent defects (001, 002, 003, 005, 006, 007, 009, 010, 013, 020 to 029) can produce wrong results with exit status 0. Every new component needs a self-check or a second-simulator cross-check. |
| Regression infrastructure | Pass/fail from logs, not exit codes (014). Already done in `sim/regress.py`. | |
| Coverage closure and signoff | Functional coverage merge by project script; no code-coverage merge or exclusions (018). | Signoff coverage needs a commercial tool. |
| Commercial PCIe VIP | Cannot run encrypted VIP (016). | Blocks any VIP-based compliance testing on Xezim. |
| Timing in packages | No current effect (004). | Any package delay or drain time under `-timescale` is 1000 times too short. |
| Random stimulus quality | `dist` is quota-based (007); nested-object sizing is broken (006). | Lower randomness than intended; can hide corner cases. |

## 09 Comparison with the project's existing trusted simulator

**Table 17. Same bench, same RTL** (`tb/` with the Root Complex BFM; logs in `pcie_suitability/logs/`)

| Measure | Icarus Verilog 12.0 | Xezim build A | Xezim build B | Xezim build C |
|---|---|---|---|---|
| Compile | `iverilog -g2012 -Wall`, exit 0; 20 "sorry: constant selects in always_* processes are not currently supported (all bits will be included)" messages, one missing-time-unit warning and two "$display in always_ff" warnings (`tb.iverilog_compile.log`) | no errors; 13 warnings for modules without a timescale | same | same |
| Result | 7 passed, 0 failed, watchdog at TEST 5 | 7 passed, 0 failed, watchdog at TEST 5 | same | same |
| Check sequence | reference | identical check names and order | identical | identical |
| LTSSM timeline (original bench) | reference | every transition one `core_clk` cycle (4 ns) later, from the first transition after reset | same as A | same as A |
| LTSSM timeline (reset race fixed, PF-03) | reference | **identical, all 12 transitions** | not run | not run |
| Wall time | 49.8 s to 50.4 s | 11.8 s to 11.9 s | 11.7 s | 12.0 s |
| Exit status | 0 | 0 | 0 | 0 |
| Text output | UTF-8 bytes as written | non-ASCII characters re-encoded (XEZ-UVM-034) | same | same |

**Interpretation.**
- **Fact:** on the only bench both simulators can run, Xezim reproduces Icarus exactly, once a testbench race is removed.
- **Judgment:**
  - The one-cycle offset seen first was a testbench race (PF-03), not a simulator defect. Both outcomes are legal under IEEE 1800-2017 4.7.
  - Icarus cannot run `uvm_tb/` or `dv/`, because it has no UVM support. For the UVM environments there is no trusted reference on this host, so the planned Xcelium flow remains the only route to a second opinion. [confirm Xcelium availability]
- **Untested:** Xcelium, Questa, VCS.

## 10 Workarounds

These rules are mandatory for Xezim use; each was verified to avoid the corresponding issue.

1. Share state between forked processes through class members, module variables or mailboxes, never task-local automatics (001).
2. Naming rules, enforced by a CI grep:
   - no class local or member named like a top-level instance (002)
   - no class named like a variable (003)
3. Put `timeunit 1ns; timeprecision 1ps;` in every testbench package. Use time literals for UVM drain times and timeouts (004).
4. Do not pack strings with `uvm_packer` (005).
5. Keep variables that size arrays in the same class as the array (006).
6. Use `$urandom_range` thresholds where distribution quality matters (007).
7. Use levels or mailboxes instead of `.triggered` across processes (008).
8. Embed covergroups in collector classes created in `build_phase`, held in members. Set `merge_instances` in the declaration (009, 010, 011).
9. Issue assertion control from the top module with hierarchical names (012).
10. Use classes, not structs, for variable-length data (013).
11. Decide pass or fail from the log's UVM summary and an explicit result line (014).
12. Use VCD or scoped FST (015).
13. Use open-source or unencrypted models; run VIP tests on a commercial simulator (016).
14. Merge functional coverage with `dv/sim/cov_merge.py`; use a commercial tool for code-coverage signoff (018).
15. Force whole signals (019).
16. The x01 to x10 workarounds already in `dv/` (020 to 029).

## 11 Residual risks and unverified items

- **Unknown silent defects.** 25 confirmed silent or partly silent defects were found while building about 11,000 lines of PCIe testbench code and a 149-check suite. More are likely in code paths not exercised here. Mitigation: periodic cross-runs of the PCIe regression on a second simulator, and self-checking tests with negative controls.
- **UVM library internals.** XEZ-UVM-001 affects a pattern the UVM library itself uses. Upstream fixed one site, but no audit of the library was done. [confirm with upstream]
- **Untested features:**
  - UVM transaction recording and `uvm_tr_database` beyond the default text file
  - `uvm_reg_fifo`, `uvm_vreg`, user-defined frontdoors other than the PCIe one
  - multi-map RAL
  - `uvm_heartbeat`
  - `uvm_hdl` signal polling (1800.2-2020)
  - VPI `cbForce`/`cbRelease`/`cbStmt`
  - `$past` with gating clocks, multi-clock assertions, `expect`
  - `covergroup` `with function sample` on class arguments of struct type
  - JIT mode (unavailable in this build)
  - UPF
  - SDF
  - Windows or macOS hosts
- **Commercial PCIe VIP:** Unverified (none available).
- **Root causes** marked "not isolated" are inferred from behaviour; fixes may reveal different mechanisms.
- **Upstream pace:** the tool changes daily (24 commits between builds A and C). Results apply to the three builds tested. Re-run `run_suite.sh` and `run_cases.sh` on every tool update before adopting it.

### Risk register

**Table 18. Risk register**

| # | Risk | Basis | Likelihood | Impact | Mitigation |
|---|---|---|---|---|---|
| R1 | New testbench code hits a silent Xezim defect and a test passes wrongly | 25 silent or partly silent confirmed defects (§06) | Medium | High | Coding rules (§10); self-checking tests with negative controls; periodic cross-run on a second simulator |
| R2 | A failing test is reported as passing by CI | XEZ-UVM-014 | High without mitigation | High | Log-based pass/fail (already in `../sim/regress.py`) |
| R3 | Coverage closure cannot be demonstrated from Xezim data | XEZ-UVM-018, 033 | Certain for signoff | High | Functional coverage merge by script (`../sim/cov_merge.py`, coverpoints only); signoff coverage on a commercial simulator |
| R4 | Encrypted commercial VIP is required later | XEZ-UVM-016 | Unknown [confirm] | Critical for that use | Run VIP-based tests on a commercial simulator; keep Xezim for in-house benches |
| R5 | Random stimulus less random than intended | XEZ-UVM-007 | Certain | Medium | `$urandom_range` thresholds for critical distributions; coverage-driven review |
| R6 | Tool changes daily (24 commits between builds A and C) | upstream history | Certain | Medium | Pin a build; run `uvm_suite/run_suite.sh` and `cases2/run_cases.sh` before adopting a new build |
| R7 | Full-chip scale | components scale superlinearly (2,000 components 70 s, 573 MB) | Low for this IP | Medium | Qualify at the target scale before committing |
| R8 | Unknown defects in untested areas (§11) | not exercised | Medium | Medium | Extend the suite before relying on those features |

## 12 Suitability decision and qualification

**Table 19. Decision by use case**

| Use case | Decision | Evidence for | Evidence against |
|---|---|---|---|
| U1. Controller-level UVM verification of this RTL with in-house agents and BFMs: development and regression | **Suitable with restrictions: GO** | §08 every scenario runs; checks effective (mutation 12/12); identical across three builds; agrees with Icarus on `tb/`; 4.2x faster than Icarus | silent defects require the §10 workarounds; exit status needs log parsing |
| U2. Transaction Layer-only verification | **Suitable with restrictions: GO** (subset of U1) | TL hand-off checks, reference model and TLP self-test run | same as U1 |
| U3. Sole simulator for signoff | **Not suitable: NO-GO** | | no coverage merge or automatic-bin database (018, 033); no independent reference for UVM results; silent-defect risk (R1) |
| U4. Encrypted commercial Root Complex VIP | **Not suitable: NO-GO** for encrypted models; unencrypted VIP Untested | | 016 |
| U5. Full-chip or SoC | **Not verified** | | no full-chip bench; superlinear component scaling (R7) |

The decision follows the measured results. Xezim being open source plays no part in it, and production readiness is not inferred from public UVM examples.

### Qualification criteria and results

**Table 20. Qualification criteria for U1**

| # | Criterion | Result | Met |
|---|---|---|---|
| Q1 | Every UVM feature the project uses passes or has a verified workaround | 144/149 suite checks pass; the 5 failures have verified workarounds | Yes |
| Q2 | Same results as the project's trusted simulator on a shared bench | `tb/`: identical results; identical timeline with the race fixed | Yes |
| Q3 | Results stable across tool builds | dv regressions A = B = C on 27 runs (original RTL) and A = C on 26 runs (patched RTL); suite identical on A, B, C | Yes |
| Q4 | Deterministic and parallel-safe | same seed byte-identical; parallel equals serial | Yes |
| Q5 | Checkers detect real defects on this tool | mutation analysis 12/12 (`../docs/results/mutation_report.txt`, build A) | Yes |
| Q6 | Failing tests visible to CI | exit status 0 on UVM errors; log parsing in place | With workaround |
| Q7 | Waveforms available for debug | VCD yes; full FST no | With workaround |
| Q8 | Coverage closure evidence | per-run JSON; coverpoint merge by script; no cross or code-coverage merge | No (signoff only) |
| Q9 | Independent cross-check of the UVM environment | Xcelium unavailable | Not done |
| Q10 | Commercial VIP | none available | Not done |

## 13 Recommendation

1. **GO for controller-level and Transaction Layer work (U1, U2).** Adopt Xezim for development and regression of the PCIe controller environment, with the features marked Passed in §04 and the workarounds in §10. Pin build C (`f56bb4df`) or a later build that passes both suites. Keep the current log-based pass/fail flow.
2. **NO-GO as the sole signoff simulator (U3) and with encrypted VIP (U4).** Four independent blockers exist:
   - exit status ignores UVM errors (014)
   - silent wrong-result defects (§11)
   - no coverage merge, automatic-bin database or code-coverage signoff (018, 033)
   - no encrypted VIP (016)
3. **Bring up the Xcelium flow** (`dv/sim/run_xrun.sh`, already written). Use it for signoff and encrypted VIP, and run the full PCIe regression on both simulators at each milestone. Treat any result difference as a bug in the testbench, the RTL or one of the tools until explained. [confirm Xcelium licence and date]
4. **Fix the project findings PF-01 to PF-04** (§05) in the repository benches; they affect every simulator.
5. **Re-qualify on every tool update.** Run `uvm_suite/run_suite.sh`, `cases2/run_cases.sh`, `pcie_suitability/run_project_tbs.sh` and the `dv/` regression. A change from FAIL to PASS on a reproducer is the signal to remove the matching workaround.
6. **Report the defects upstream** with the reproducers in `cases2/` (aionhw/xezim issues), starting with XEZ-UVM-001, 002, 004 and 014. Most have no upstream issue yet (§04). [confirm who owns the upstream contact]

## 14 References and evidence

Official project and documentation, pinned to the evaluated commit (`<base>` = `https://github.com/aionhw/xezim/blob/83df0dd53506f12f58eff8cb87687423c86ebd78/`):
- Project: https://github.com/aionhw/xezim
- UVM guide, supported features and known limitations: `<base>/docs/uvm-guide.md#known-limitations--out-of-scope`, backdoor access `<base>/docs/uvm-guide.md#L147-L155`
- Supported features, UVM section: `<base>/docs/features.md#uvm`
- Coverage guide, limits: `<base>/docs/coverage-guide.md#limits-and-unsupported-features`
- DPI guide, running and VPI: `<base>/docs/dpi-guide.md#running-with-xezim`, `<base>/docs/dpi-guide.md#L378`
- Command line, `--error-exit`, `+seed`, `-timescale`: `<base>/docs/command-line.md#L29`, `#L55`, `#L120-L121`
- User guide, module-timescale extension: `<base>/docs/user-guide.md#module-timescale-extension`
- Upstream fork-sharing fix: commit `50999fbde727c742c578dbc87c6a8b3027fe40da`
- Source: `src/compiler/simulator.rs` (lines 7383, 7412, 49859), `src/compiler/simulator/assert_ctl.rs` (line 327), `src/compiler/fst_sink.rs` (line 390)

Documentation statements that do not match observed behaviour (documentation findings, not limitations):
- `uvm-guide.md` line 191 lists "RAL ... and sequence lock/grab arbitration beyond the common path" as limitations. Lock, grab, the RAL built-in sequences, the predictor and W1C all pass (U05-07, U05-08, U11).
- `dpi-guide.md` lines 319 to 320 say "xezim today requires [`import "DPI-C"`] inside a module". Imports declared in a package work (U13-01 to U13-06).
- `features.md` line 47 names 1800.2-2017 and 1800.2-2020.3.1. UVM 1.2 is not listed but gives identical results.

Standards: IEEE 1800-2017 (SystemVerilog), IEEE 1800.2-2017 and 1800.2-2020 (UVM), Accellera UVM 1.2.

Upstream issue tracker: https://github.com/aionhw/xezim/issues (issues #16, #68, #77, #107, #255, #271, #290 cited).

**Table 21. Evidence files** (all exist; verified when this document was produced)

| Evidence | Location | Reproduce with |
|---|---|---|
| Project benches on Icarus and Xezim A/B/C | `pcie_suitability/logs/{tb,tb_racefix,uvm_tb,uvm_tb_scopefix}.*.{cmd,log,exit,time}`, `summary.txt` | `pcie_suitability/run_project_tbs.sh` (`XEZIM_C=<binary>` for a third build) |
| Testbench fixes used in experiments (repository unchanged) | `pcie_suitability/tb_reset_race_fix.diff`, `uvm_tb_scope_fix.diff` | applied to scratch copies by the runner |
| `dv/` regression, original RTL | `logs/baseline_final` (A), `logs/baseline_23f3728c` (B), `logs/baseline_f56bb4df` (C), each with `summary.md` | `XEZIM=<binary> ./run_one.sh <test> <seed> <tag>` |
| `dv/` regression, patched RTL | `../docs/results/regress_xezim_patched.txt` (A), `logs/patched_f56bb4df` (C) | `RTL_F=rtl_patched.f XEZIM=<binary> ./run_one.sh ...` |
| Mutation analysis | `../docs/results/mutation_report.txt` | `../sim/mutate.py` |
| UVM capability suite | `uvm_suite/results/<build>[_<version>]/` | `uvm_suite/run_suite.sh` |
| Minimal reproducers | `cases2/results/<case>.<simulator>.*`, `summary_with_f56bb4df.txt` | `cases2/run_cases.sh` |
| Earlier reproducers x01 to x10, exit status, `dist`, IEEE 1735, FST | `xissues_recheck/` | commands in each `.cmd` file |

## Appendix A · Feature-to-test map

Each ID is one `CHECK` line in `uvm_suite/results/<build>/<test>.log`. All pass on both builds and all three UVM libraries except the five marked FAIL.

**Table 22. Feature-to-test map**

| Test file | Checks |
|---|---|
| `u01_phases.sv` | U01-01 build top-down; 02 connect bottom-up; 03 end_of_elaboration before start_of_simulation; 04 run-time phase order; 05 phase jump; 06 extract/check/report/final order; 07 drain time; N1 phase timeout fatal |
| `u02_reporting.sv` | U02-01 error count; 02 id count; 03 severity override; 04 catcher THROW/CAUGHT; 05 quit count; 06 +UVM_VERBOSITY; 07 set_report_verbosity_level; 08 cmdline tool name |
| `u03_factory.sv` | U03-01 instance override; 02 type override; 03 override by name; 04 create_object_by_name; 05 parameterized class; 06 +uvm_set_type_override; N1 unknown type |
| `u04_config_db.sv` | U04-01 field auto-config; 02 build-time precedence; 03 regex scope; 04 object handle; 05 virtual interface; 06 +uvm_set_config_int; 07 exists; 08 wait_modified; 09 resource_db; N1 unset name; N2 wrong type |
| `u05_sequences.sv` | U05-01 items and responses; 02 pre/post_body; 03 p_sequencer; 04 virtual sequence; 05 FIFO; 06 STRICT_FIFO; 07 lock; 08 grab; 09 sequence library; 10 stop_sequences |
| `u06_tlm.sv` | U06-01 analysis broadcast; 02 analysis FIFO; 03 FIFO order; 04 try_put full; 05 blocking put; 06 port/export/imp; 07 TLM 2 b_transport; N1 unconnected port |
| `u07_callbacks_events.sv` | U07-01 instance and type-wide callbacks; 02 delete; 03 callback_mode; 04 event data; 05 event pool; 06 wait_ptrigger; 07 64 events in an associative array; 08 barrier |
| `u08_objects.sv` | U08-01 copy; 02 clone/compare; **03 pack/unpack FAIL (005)**; 04 pack_bytes; 05 sprint; N1 compare detects difference |
| `u09_coverage.sv` | U09-01 subscriber; 02 ignore_bins; 03 array bins; 04 transitions; 05 iff; 06 cross ignore_bins; 07 cross select bin; 08 group average; 09 at_least; 10 negative control; **11 FAIL (009)**; **11b FAIL (010)**; 11c members; 12 merge_instances; 13 stop(); 14 illegal bin error (log) |
| `u10_sva.sv` | U10-01 ##[1:3] with disable iff; 02 throughout and [->1]; 03 $fell/$past; 04 local variable; 05 covers incl. named sequence; 06 $rose cover; 07 $stable; 08 $assertoff; 09 negative control; 10 UVM report counts |
| `u11_ral.sv` | U11-01 model; 02 hw_reset_seq; 03 negative wrong reset; 04 frontdoor with predictor; 05 backdoor; 06 needs_update; 07 update; 08 W1C; 09 mirror check; 10 negative deposit; 11 bit_bash; 12 access_seq; 13 mem_walk; 14 mem peek |
| `u12_hdl.sv` | U12-01 check_path; 02 read whole/bit/part/$root; 03 ascending range; 04 memory word and struct member; 05 net; 06 deposit; 07 force/release variable; 08 force net; 09 release_and_read; 10 force_time; 11 negative bad paths; 12 partial force (documented) |
| `u13_dpi_vpi.sv` | U13-01 int; 02 string; 03 open array in; 04 open array out; 05 64-bit vector; 06 chandle; 07 context import with exported function and task; 08 svGetScope; 09 cbValueChange; 10 cbReadOnlySynch; 11 vpi_get/vpi_put_value |
| `u14_tops_clocking.sv` | U14-01 vif across two tops; 02 output skew; 03 #1step sampling; 04 ##N; 05 fractional delay; 06 1us module; 07 uvm_hdl through the other top; 08 vector drive |
| `u15_scale.sv` | U15-01 N items delivered; 02 N components configured |
| `u16_random.sv` | U16-01 size/foreach/unique/implication/nested; **02 dist over kinds FAIL (006)**; 03 pre/post_randomize; 04 derived override; **05 inline with local:: FAIL (006)**; 06 sum(); 07 constraint_mode; 08 infeasible; 09 srandom stability; 10 different seed |
