# big-endian AArch64 and ARM closure argument/return offsets are wrong

## Summary

Sub-64-bit (AArch64) and sub-32-bit (ARM) integer and scalar-float arguments
passed in registers or on the stack are mis-extracted on big-endian targets,
so closures receive wrong/garbage arguments and (AArch64) wrong return values.
Reproduces on `aarch64_be-linux`, `aarch64_be-musl`, and `armeb-linux`.

## Details

On big-endian RISC ABIs a sub-word scalar occupies the **least significant
bits** of its register/stack slot, i.e. the **highest-address bytes**. To read
it you must offset the slot pointer by `slot_width - type_size`. libffi's
AArch64 and 32-bit ARM closure paths advance the slot without that offset:

- AArch64 `src/aarch64/ffi.c`:
  - `allocate_int_to_reg_or_stack` returns the raw slot base (no BE offset).
  - `ffi_call_int` spills float args to the stack without the offset and
    without copying the value.
  - `ffi_closure_SYSV_inner` spills scalar floats without the offset.
  - `compress_hfa_type` reads S1/D1 returns without the register offset.
- ARM `src/arm/ffi.c`:
  - `ffi_prep_incoming_args_SYSV` and `ffi_prep_incoming_args_VFP` read
    sub-4-byte ints from offset 0 (comment says *"Because we're little
    endian, this is what it turns into."*).

Other BE backends already do this (`s390` uses `&slot + sizeof(long) - size`;
`raw_api.c` uses `+ FFI_SIZEOF_ARG - size`).

## Repro

`testsuite/libffi.call/many.c` (13 float args, float return) and the closure
tests fail on `aarch64_be`. `testsuite/libffi.closures/closure_fn2.c` shows a
16-bit `signed short` arg arriving as 0 and an int return as garbage on
`aarch64_be`; `closure_fn0.c` etc. fail on `armeb`. A minimal closure calling a
`sint`-returning function on `aarch64_be` under QEMU reports
`sshort` → 0 and result garbage (expected 255).

## Proposed fix

Add a `BE(x)` helper and apply `BE(slot_width - size)` when locating
sub-word scalar arguments/returns. Included in:
`https://github.com/CoffeeImpliesCode/libffi/tree/big-endian-closures`
(aarch64: fix big-endian argument and return offsets in closures;
arm: fix big-endian sub-word argument offsets in closures).

This also addresses the still-open **issue #675** (`ffi_call_int` float return
on `aarch64_be`).

## Verification

All 226 upstream tests pass on `aarch64_be` and `aarch64` (LE), and the
closure tests pass on `armeb`/`arm` (LE), under QEMU, built with both
clang (zig) and GCC cross-toolchains.
