# Chinese Wall: Dirty Room

This repository is the **dirty room** in a two-team clean-room process. People
(and agents) working here read GPL-licensed source code, chiefly the Linux
kernel at `../linux/`, and write **functional specifications**. A separate
**clean room** team, who never sees the source, implements drivers for a
BSD-licensed operating system using only those specifications.

Everything under `specs/` is meant to go over the wall. It must hold no
copyrightable expression taken from the source.

## Layout

| Path | Purpose | Crosses the wall? |
|------|---------|-------------------|
| `specs/<driver>/` | Functional specifications | **Yes** |
| `provenance/<driver>/` | Which source files and lines informed each spec section, for audit | **No** |
| `SOURCE-BASELINE.md` | Exact upstream revision the specs were extracted from | No |
| `CLAUDE.md`, `README.md` | Process documentation | README yes, this file optional |

## Rules for writing specs (mandatory)

1. **Describe behavior, not implementation.** Write what the hardware or
   firmware needs and how it responds, such as register semantics, sequences,
   timing constraints, memory layouts and error conditions. Do not describe how
   Linux arranges its code.
2. **No source code.** Do not include C, Rust or macro syntax, and do not copy
   or lightly reword code fragments. Pseudo-code is allowed only for
   hardware-mandated sequences. It must be short, written fresh in neutral
   step-numbered prose style, and must not mirror the Linux function structure,
   control flow or variable names.
3. **No copied comments or documentation prose.** Restate every fact in your
   own words. kernel-doc comments are copyrighted expression too.
4. **Facts are fine.** Register offsets, bit positions, field widths, magic
   values, enum encodings, structure layouts at fixed offsets, firmware ABI
   constants, ioctl numbers and struct layouts needed for binary compatibility
   are functional facts needed for interoperability. Present them in tables.
5. **Naming.** Hardware register and field names (for example `GPU_ID`,
   `AS_TRANSTAB`, `MCU_CONTROL`) may be used as mnemonics because they name
   hardware facts. Do **not** use Linux-internal identifiers such as function
   names, struct names, local variables, or kernel framework names presented as
   requirements (for example `drm_sched` or `drm_gpuvm`). Describe the needed
   capability generically, such as "a job scheduler with per-queue
   dependency tracking".
6. **Separate requirement from policy.** Label each statement as one of:
   - **[HW]**: required by hardware or firmware. A different driver must do this.
   - **[POLICY]**: a Linux design choice, such as timeout values, tick periods
     or heuristics. Informative only. The clean room may choose differently.
   - **[ABI]**: userspace-visible interface needed for compatibility with
     existing userspace (Mesa).
7. **Mark uncertainty.** Use `> OPEN QUESTION:` for anything the source
   implies but does not prove, such as undocumented bits or magic constants
   with no explanation.
8. **No source citations inside `specs/`.** File and line references go in the
   matching `provenance/` file. Specs may name the public origin of a fact when
   it is public (an ARM Architecture Reference Manual, a devicetree binding).
9. **Devicetree bindings** (`Documentation/devicetree/bindings`) are
   dual-licensed GPL-2.0/BSD-2-Clause and are interface definitions. They may be
   summarized freely.

## Provenance files

Each `specs/<driver>/NN-topic.md` has a matching `provenance/<driver>/NN-topic.md`
that lists:
- the source files and line ranges read
- the baseline commit (see `SOURCE-BASELINE.md`)
- who or what extracted it, and the date

## Source baseline

The Linux tree at `../linux/` is read-only. Never modify it.
