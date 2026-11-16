# leopard-boot

LeopardOS UEFI boot component. It builds `out/BOOTX64.efi`, loads
`/system/kernel.krl`, and passes the boot configuration to `leopard-xsk`.

```bash
python3 tools/gen_ninja.py
ninja            # out/BOOTX64.efi
ninja pack       # out/leopard-boot-2.1.1_2.x86_64.xbps
```

The boot protocol and configuration are consumed by `leopard-xsk`. See
`FORK.md` for provenance and licensing.
