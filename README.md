# ape9patch

**APE (Actually Portable Executable) binary rewriting and live delta patching.**

`ape9patch` extends [e9patch](https://github.com/GJDuck/e9patch) — Gregory J. Duck's static
binary rewriter — from `x86_64` Linux **ELF** binaries to
[Cosmopolitan](https://github.com/jart/cosmopolitan) **APE polyglot** binaries, across
AMD64 and AArch64.

Upstream e9patch is ELF-only. An APE binary is a single file that is simultaneously a DOS MZ
executable, a shell script, a PE executable and an ELF — and, critically, **it carries no
embedded `x86_64` ELF header at all**. The ELF-header and program-header paths that upstream
relies on have nothing to read. `ape9patch` is the layer that makes APE a rewritable target.

## Why APE needs its own layer

Structure recovered by reverse-engineering real `cosmocc` output
(see [`doc/ape-anatomy-analysis.md`](doc/ape-anatomy-analysis.md)):

```
0x00000   MZ header "MZqFpD" + shell-script bootstrap
0x10A58   PE header (e_lfanew points here)
0x11000   .text     <- file offset == RVA in APE
0x2F000   .rdata
0x35000   .data
0x3C000   ARM64 ELF  (aarch64 only - NOT x86-64)
EOF-256   ZipOS (.cosmo, .symtab.amd64, .symtab.arm64)
```

Consequences that drive the design:

- **No `x86_64` ELF header.** PE sections are used as ground truth for address translation.
- **`file_offset` frequently equals RVA**, but not always — the mapping is computed per
  section rather than assumed.
- **The ARM64 ELF at `0x3C000` is a separate image view.** Patching an x86-64 offset must not
  disturb it, and vice versa.
- **ZipOS occupies the tail of the file**, holding `.cosmo` and the per-architecture symbol
  tables.

## What ships today

| Capability | Entry point |
|---|---|
| APE detection | `e9_ape_detect()` |
| APE parsing into image views | `e9_ape_parse()` -> `E9_APEInfo` |
| Address translation | `e9_ape_rva_to_offset()` / `e9_ape_offset_to_rva()` |
| In-place delta patching | `e9_ape_patch_offset()` / `e9_ape_patch_rva()` / `e9_ape_patch()` |
| ZipOS entry enumeration | `e9_ape_zipos_exists()` / `e9_ape_zipos_free_list()` |
| Live reload / hot patching | `src/e9patch/e9livereload.c` |
| Cross-platform process memory | `src/e9patch/e9procmem.c` |
| Info dump | `e9_ape_dump_info()` |

Live reload watches the target with `stat` polling — portable across the platforms an APE
runs on, rather than depending on a Linux-specific notify API — and applies patches to the
mapped image.

### Status — scope of the patching primitive

Patching is **same-size, in-place replacement** at a computed file offset: new bytes must be
the same length as the bytes they replace. This is what the live-reload path is built on
(`e9livereload.c` treats Binaryen-produced patches as same-size replacements).

Trampoline-based *insertion* — upstream's `e9trampoline` machinery, which is what gives
e9patch its ELF instrumentation power — is **not wired into the APE path**. Same-size
overwrites are safe across the disjoint MZ / PE / ELF / ZipOS regions by construction;
trampoline-grade insertion on APE would additionally have to preserve coherence between the
image views, and that is not implemented here. The ELF and APE feature sets are not
equivalent.

## Layout

First-party APE work (no counterpart upstream):

```
src/e9patch/e9ape.c          e9ape.h          APE detect / parse / address map / patch
src/e9patch/e9livereload.c   e9livereload.h   live reload + hot patching
src/e9patch/e9procmem.c      e9procmem.h      unified cross-platform process memory
doc/ape-anatomy-analysis.md                   RE analysis of APE structure
specs/e9ape.schema  e9ape.sm                  APE model + state machine
specs/e9livereload.schema                     live-reload model
specs/behavior/livereload.sm                  live-reload state machine
specs/features/ape_detection.feature
specs/features/ape_patching.feature
specs/features/e9livereload.feature
test/livereload/                              test harness
```

Everything else is upstream e9patch, retained with its history intact. Upstream's own README
is preserved as [`README.e9patch.md`](README.e9patch.md).

## Build

```sh
make -f Makefile.e9studio          # builds the APE layer (e9ape.c, e9livereload.c)
make -f Makefile.cosmo studio      # builds e9studio.com - the tool itself as an APE
cd test/livereload && make         # live-reload test harness
```

## Related

- [`cosmo-bde`](https://github.com/ludoplex/cosmo-bde) — the spec-driven C generation
  framework these specs are dogfooded against.
- [`e9studio`](https://github.com/ludoplex/e9studio) — IDE front-end: compile and edit C
  source with the resulting binary updated dynamically, DWARF symbol mapping, auto-CFG.

## License

**GPLv3+**, inherited from upstream e9patch.

Upstream e9patch is copyright © Gregory J. Duck and licensed GPLv3. This repository is a
derivative work: upstream commit history and copyright notices are preserved, and the APE
layer added here is released under the same terms. See [`LICENSE`](LICENSE).
