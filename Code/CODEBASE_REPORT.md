# Dilithium Codebase Report

## 1. Executive summary

This repository contains a hardware-oriented implementation of CRYSTALS-Dilithium, a lattice-based digital signature scheme, plus a separate C++ model of its 256-point Number Theoretic Transform (NTT) engine. The hardware top module is combined_top in rtl_src/combined_top.v. It accepts 64-bit input words through a valid/ready streaming interface and supports key generation, signing, and verification at parameter levels 2, 3, and 5.

The RTL is mixed-language: most modules are Verilog, while the SHAKE/Keccak implementation is VHDL. combined_top instantiates three Keccak cores. In the tested Windows flow, GHDL analyzed and synthesized the VHDL hierarchy into Verilog, then Icarus compiled that netlist with the Verilog RTL and testbench.

**Verification status:** GHDL analysis/synthesis and Icarus compilation succeeded. A complete functional key-generation KAT run has not been confirmed. The simulation emitted $readmemh warnings and ran for about ten minutes without printing its first KG2 completion line; it was stopped before producing a pass/fail result. This report therefore does not claim that the complete RTL is functionally correct.

## 2. Repository contents

| Path | Purpose |
|---|---|
| rtl_src/ | Main hardware implementation; mostly Verilog, with VHDL Keccak/SHAKE sources. |
| rtl_tb/ | Testbenches for key generation, signing, and verification. |
| KAT/ | Hexadecimal known-answer-test input and expected-output data. |
| zetas.txt | Twiddle-factor ROM initialization data for the NTT. |
| dilithium-256/reference_code/ | Reference NTT and 2×2 NTT code with randomized comparison tests. |
| dilithium-256/hardware_code/ | C++ model of the hardware-style NTT datapath and test driver. |
| dilithium-256/params.h | Shared C++ arithmetic parameters. |
| README.md | Project overview and paper citation. |

There is no general-purpose signature API or command-line app for producing keys and signatures. The RTL is exercised through testbenches. The C++ programs test NTT arithmetic rather than full Dilithium key generation, signing, and verification.

## 3. Parameters and interface

The shared C++ parameters in dilithium-256/params.h are:

| Parameter | Value | Meaning |
|---|---:|---|
| DILITHIUM_Q | 8,380,417 | Prime modulus q = 2^23 − 2^13 + 1 |
| DILITHIUM_N | 256 | Polynomial degree and NTT coefficient count |
| DILITHIUM_LOGN | 8 | log2(N) |
| data_t | signed 32-bit | Coefficient storage type |
| data2_t | signed 64-bit | Wider intermediate arithmetic type |

The hardware selects matrix dimensions K and L according to sec_lvl:

| Code level | K | L | Interpretation |
|---:|---:|---:|---|
| 2 | 4 | 4 | Four matrix rows and four secret-vector polynomials |
| 3 | 6 | 5 | Six matrix rows and five secret-vector polynomials |
| 5 | 8 | 7 | Eight matrix rows and seven secret-vector polynomials |

combined_top uses a 64-bit input/output stream. valid_i/ready_i coordinate input transfers; valid_o/ready_o coordinate output transfers. The supplied testbenches use mode 0 for key generation, mode 1 for verification, and mode 2 for signing. sec_lvl selects 2, 3, or 5.

Coefficients are represented in 23-bit lanes. NTT memories pack four 24-bit lanes into 96-bit words. The design uses two NTT operation engines and multiple banks for intermediate polynomial values.

## 4. Hardware architecture and module responsibilities

### 4.1 Top-level controller

combined_top.v orchestrates the operation-specific state machines and moves data among samplers, encoders/decoders, memory banks, Keccak cores, and NTT engines. It implements the external stream and sequences key generation, signing, and verification.

The top declares seven inferred dual-port RAMs:

- BRAM_0: 96 bits × 4096 words.
- BRAM_1 through BRAM_6: each 96 bits × 1024 words.

Their aggregate declared capacity is 983,040 bits, or 120 KiB, before registers and ROMs. This is a source-level calculation; actual FPGA resource mapping depends on the synthesis tool and target device.

The design instantiates two operation_module engines and three Keccak cores. It routes Keccak traffic among matrix generation, secret sampling, challenge sampling, and message/transcript hashing.

### 4.2 NTT and modular arithmetic

| Module(s) | Responsibility |
|---|---|
| operation_module.v | Sequences forward/inverse NTT, pointwise multiplication, addition, and subtraction; controls addresses, memory writes, and twiddle loading. |
| address_unit.v, address_resolver.v, twiddle_resolver.v | Generate and map RAM/twiddle addresses for transform stages and layouts. |
| butterfly2x2.v | Coordinates a 2×2 butterfly datapath and instantiates four butterfly units. |
| butterfly.v | Performs modular butterfly arithmetic for the supported NTT modes. |
| Barrett_8380417.v | Implements pipelined Barrett reduction for modulus 8,380,417. |
| ntt_fifo.v, ntt_fifo_piso.v, ntt_pipo.v | Delay, reorder, and serialize/parallelize coefficients and twiddle values for the pipeline. |
| dual_port_ram.v, dual_port_rom.v | Inferred dual-port storage/ROM; initialization uses $readmemh. |

For N=256, a radix-2 transform has eight stages with N/2 butterflies per stage: 128 × 8 = 1,024 butterfly operations per polynomial transform. The 2×2 unit computes several operations in parallel, so cycle counts depend on scheduling, pipeline latency, memory accesses, address mapping, and stalls. No hardware clock rate or synthesis timing report is present.

### 4.3 Sampling and coefficient processing

| Module group | Responsibility |
|---|---|
| gen_s.v, sampler_s.v, rejection_s.v | Generate secret polynomials from SHAKE output using rejection sampling. |
| gen_a_ext.v, sampler_a_ext.v, rejection_a.v | Generate public matrix polynomials from a seed; extended modules coordinate multiple sampler lanes. |
| expandmask_ext.v, sampler_y_ext.v, rejection_y.v | Expand the signing mask vector and rejection-sample its coefficients. |
| gen_c.v | Generate the sparse challenge polynomial from transcript/hash input. |
| norm_check.v | Check coefficient bounds and signal rejection where required. |
| encoder.v, decoder.v, zero_strip.v, uncenter_coeff.v | Pack/unpack coefficient vectors for the 64-bit external stream. |
| decomposer_unit.v, coeff_decomposer.v, decomp_map1.v | Decompose coefficients into high/low parts used by signing and verification. |
| makehint.v, usehint.v | Create and apply hint data used in signature equations. |

These blocks use ready/valid handshakes so data can be processed incrementally.

### 4.4 Keccak/SHAKE

The VHDL hierarchy under rtl_src includes keccak_top.vhd, keccak_control.vhd, keccak_datapath.vhd, keccak_round.vhd, keccak_fsm1.vhd, keccak_fsm2.vhd, sha3_fsm3.vhd, keccak_pkg.vhd, sha3_pkg.vhd, keccak_cons.vhd, keccak_bytepad.vhd, and storage/counter helpers sipo.vhd, piso.vhd, regn.vhd, sr_reg.vhd, and countern.vhd.

Keccak is included in the repository, not an external crypto library. Its VHDL language does require a mixed-language simulator or conversion flow when used with Verilog.

## 5. Operation flows

### Key generation

The key-generation FSM expands the input seed, samples secret material, computes public matrix/vector products using the NTT engines, forms public polynomial values, and encodes outputs. The testbench reads seed files from KAT and compares outputs including rho, K, s1, s2, t1, t0, and tr against KAT values.

### Signing

The signing FSM accepts secret-key and message information, hashes transcript data, samples mask vector y, computes the matrix product, derives a challenge, and forms response and hint values. Rejection checks can restart work with a new mask. The testbench checks values including z, h, and c. Since rejection sampling is data-dependent, exact cycle counts depend on sampled values and retries.

### Verification

The verification FSM decodes public key/signature material, computes NTT-domain products, combines/decomposes polynomial values, applies hints, hashes the reconstructed transcript, and compares with the supplied challenge. Its testbench reports completion and rejection information.

## 6. C++ NTT implementation

The C++ subtree models and tests the NTT datapath independently from the complete RTL controller.

### Reference implementation

reference_code/ref_ntt.cpp and ref_ntt2x2.cpp implement standard and 2×2-reference transforms. ref_test_ntt_ntt2x2.cpp creates randomized 256-coefficient vectors and compares forward/inverse results modulo q. Its TESTS constant is 100,000.

### Hardware-style implementation

hardware_code/ntt2x2_fwdntt.cpp, ntt2x2_invntt.cpp, and ntt2x2_mul.cpp emulate hardware memory layout, rolling FIFOs, address mapping, butterfly operations, and multiplication. ntt2x2_test.cpp compares these operations with reference routines. It tests standalone multiply, forward transform, inverse transform, and polynomial multiplication; TESTS is 1,000,000.

hardware_code/gen_table.py prints a generated twiddle-factor table. It uses the Python standard library and is not needed to compile/run the C++ programs.

## 7. Time-complexity discussion

These are algorithmic operation-count estimates, not measured runtime or FPGA latency. Assume N varies while a security level fixes K and L. This repository uses N=256 and fixed supported levels.

### NTT

A radix-2 NTT performs N/2 butterflies at each of log2(N) stages:

- Operations per polynomial: (N/2) × log2(N).
- Asymptotic work: O(N log N).

For N=256, this is 1,024 butterfly operations per forward or inverse transform, excluding pipeline fill/drain, memory accesses, modular reductions, and inverse scaling. NTT polynomial multiplication uses two forward transforms, N pointwise products, and one inverse transform, so it is O(N log N).

The 2×2 design changes parallelism and data movement, not asymptotic work. If all four butterfly units were continuously utilized, 1,024/4 = 256 issue groups is a compute-only lower bound per transform. This is not a measured or guaranteed cycle count.

### Matrix-vector work

A×s or A×y performs K×L polynomial products and accumulates the results. With NTT-based polynomial multiplication, the dominant arithmetic term is approximately O(K L N log N). This omits vector additions, decomposition, encoding, hashing, and rejection checks.

K×L is 16, 30, and 56 for levels 2, 3, and 5. Thus the level-5 matrix-product term is 3.5 times the level-2 term before accounting for vector lengths and packing. Hardware cycle growth can differ due to multiple engines and level-specific schedules.

### Whole operations and test-driver cost

For fixed K and L, key generation, signing, and verification are dominated by a constant number of matrix-vector products and scale roughly as O(N log N). If K and L grow too, the dominant term is O(K L N log N). Hashing and packing are linear in the number of bytes/words. Rejection sampling adds data-dependent work; signing therefore has no fixed runtime derivable from the source without bounding retries.

For T randomized tests, the C++ drivers are approximately O(T N log N), plus array initialization and comparison. The reference test uses T=100,000; the hardware-style test uses T=1,000,000. They are stress tests, not quick smoke tests.

## 8. Build and simulation notes

### C++ on MinGW

The C++ Makefiles use Linux-specific absolute compiler/remove paths. On Windows, compile directly with MinGW g++ or adapt the Makefiles.

Reference test, from the repository root:

```powershell
Set-Location .\dilithium-256\reference_code
g++ -O3 -Wall ref_ntt.cpp ref_ntt2x2.cpp ..\consts.cpp ref_test_ntt_ntt2x2.cpp -o ref_test_ntt_ntt2x2.exe
if ($LASTEXITCODE -eq 0) { .\ref_test_ntt_ntt2x2.exe }
```

Hardware-style test:

```powershell
Set-Location ..\hardware_code
g++ -O3 -Wall -Wpedantic ..\consts.cpp ..\reference_code\ref_ntt.cpp ..\reference_code\ref_ntt2x2.cpp address_encoder_decoder.cpp util.cpp ram_util.cpp ntt2x2_fwdntt.cpp ntt2x2_invntt.cpp ntt2x2_mul.cpp consts_hw.cpp ntt2x2_test.cpp -o ntt2x2_test.exe
if ($LASTEXITCODE -eq 0) { .\ntt2x2_test.exe }
```

The hardware-style C++ test runs one million iterations. Reducing TESTS in a local test copy can make an experiment faster but lowers randomized coverage.

### RTL on Windows

The testbenches load files by basename, so run vvp with KAT/ as the current working directory. Icarus alone cannot compile the VHDL Keccak hierarchy; the tested local route is GHDL analysis/synthesis of that VHDL to Verilog, followed by Icarus compilation. The zeta ROM path must resolve from the simulation working directory; ../zetas.txt works from KAT/.

The KAT files contain 100 vectors, while each checked-in testbench declares NUM_TV = 5. Icarus warns that the files contain more words than fit in the arrays. These benches load only the first five entries, not all 100. They use $display for mismatches, so inspect output for Error lines and completion messages. A zero exit code alone is not sufficient evidence of a KAT pass.

## 9. Verification findings and limitations

1. **VHDL analysis succeeded.** GHDL analyzed the included Keccak hierarchy using VHDL-2008 mode and Synopsys compatibility.
2. **VHDL synthesis succeeded.** GHDL converted keccak_top to a Verilog netlist. It warned that sel_cin has no assignment in keccak_control.vhd. The keccak_top.vhd instance does not map this output, so it is unused at that top level.
3. **Icarus compile/elaboration succeeded** when the generated netlist was plain ASCII Verilog and included with the RTL and testbench.
4. **The full key-generation simulation produced no verified result.** It emitted $readmemh warnings and ran for roughly ten minutes without a KG2 completion line, then was stopped. This does not establish whether it would eventually complete or is stuck.
5. **No full functional pass is claimed.** README.md says the original design was verified against 100 KATs, but checked-in benches cover five cases per level and the local run did not finish. Reproducing the README verification requires expanding bench coverage and completing the runs.
6. **No FPGA timing/area report was found.** Memory totals and complexity estimates are source-based and do not demonstrate that a target FPGA meets resource or frequency requirements.

## 10. Overall assessment

This research implementation centers on a streaming Dilithium controller and a 2×2 NTT engine. It includes RTL for keygen/sign/verify, KAT fixtures and testbenches, and separate C++ NTT models that compile with MinGW. The NTT has the expected O(N log N) work; matrix-vector operations scale with K×L polynomial products.

The main Windows build challenges are the mixed Verilog/VHDL Keccak hierarchy and the runtime of a complete event-driven key-generation simulation. The source compiles through the tested conversion flow, but a completed RTL KAT run and FPGA timing/resource results are still needed before concluding that the local build is functionally correct or meets a performance target.

## 11. Project reference

README.md cites the paper *High-Performance Hardware Implementation of CRYSTALS-Dilithium*, presented at FPT 2021. The full citation is in README.md.

