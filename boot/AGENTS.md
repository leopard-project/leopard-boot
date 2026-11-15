# BOOT KNOWLEDGE BASE

## OVERVIEW
UEFI bootloader subtree. Builds `out/BOOTX64.efi`, gathers firmware state, loads `\system\kernel.krl`, and passes `BOOT_CONFIG` into `KernelMain`.

## WHERE TO LOOK
| Task | Location | Notes |
|------|----------|-------|
| UEFI entry | `bootx64.c` | `efi_main(EFI_HANDLE, EFI_SYSTEM_TABLE *)`. |
| EFI helpers | `bootlib.c`, `bootlib.h` | File, memory, ELF helper routines. |
| Boot headers | `include/` | Local EFI/ELF/ACPI/framebuffer definitions. |
| Boot LSP flags | `.clangd_template` | Separate include path: `${workspaceFolder}/boot/include`. |

## CONVENTIONS
- Built by the generated Ninja graph with the boot-specific C flags emitted by `tools/gen_ninja.py`.
- Entry symbol is forced with `-e efi_main`; PE/COFF subsystem is set with `-Wl,--subsystem,10`.
- Boot code uses `-fshort-wchar`, `-nostdinc`, `-nostdlib`, and its own `boot/include` headers.
- Global EFI handles (`ST`, `BS`, `GOP`, `SFSP`, `LIP`, `SPP`, `BAT`, `IM`) are established in `bootx64.c`; preserve setup order.
- Boot menu options set `BOOT_FLAG_*` bits consumed later in `kernel/main.cpp`.
- Status/progress UI uses fixed small buffers; keep boot messages short.
- Keep boot-only helpers here. Kernel logic belongs in `kernel/`, not in the EFI loader.

## BOOT.CONF / LIVE ROOT (install-closure additions)
- `\system\boot.conf` is parsed by `boot_conf_load()` before `BOOT_CONFIG` is filled. The raw text is copied
  into the variable-length `BOOT_CONFIG.cmdline` tail; the kernel parses `init=<path>` (empty/missing ⇒
  `/system/leopard/leopard-rc.elf`) and recognizes `flags=<dec|0xhex>` (also OR-ed into `BOOT_CONFIG.boot_flags`).
  Unknown keys are ignored and logged on serial; `#` starts a comment.
- `\system\live-root.img` (legacy filename) is probed by `boot_load_initrd()`; when present it sets
  `BOOT_FLAG_INITRD` and hands the buffer to the kernel as `initrd_phys`/`initrd_size` (generic initrd).
- `BOOT_CONFIG` layout must stay in sync with `OpenXSKernel/include/efi/boot.h`; both sides carry the same
  `sizeof`/`offsetof` static assertions. The kernel also validates `BOOT_CONFIG.layout` and halts on mismatch.
- The kernel keeps a pointer into the bootloader's `BOOT_CONFIG` allocation for the variable `cmdline` tail;
  do not free or reuse that UEFI pool allocation before/after handoff.

## PACKAGING
```bash
python3 tools/gen_ninja.py
ninja -f build.ninja pack     # → out/OpenXSBoot-2.1.1_2.x86_64.xbps (installs EFI/BOOT/BOOTX64.EFI)
```
The `OpenXSBoot` package is what puts the bootloader on an installed disk; without it the target partition
lacks the `EFI/BOOT/BOOTX64.EFI` marker and `mount_root` will not select it.


## ANTI-PATTERNS
- Do not assume libc, heap, threads, or kernel facilities are available in boot code.
- Do not replace 16-bit EFI strings with normal `char *` paths where firmware APIs expect `CHAR16`/wide strings.
- Do not bypass `BOOT_CONFIG`; kernel-visible boot decisions should be passed through that struct/flags.
- Do not apply root `.clangd_template` to boot; it intentionally excludes `boot/.*`.

## BUILD / VERIFY
```bash
python3 tools/gen_ninja.py --out build.ninja
ninja -f build.ninja all
ninja -f build.ninja vdisk
```
Inspect `out/BOOTX64.efi`; runtime validation requires booting the image via `ninja -f build.ninja run` or
`ninja -f build.ninja justrun`.
