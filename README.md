# OpenBSD 8.0 amdgpu: fix boot-time screen noise on the RX 9070 XT

This is the OpenBSD 8.0 DRM driver (`sys/dev/pci/drm`) plus `efifb`, with one
fix on top. The first commit is the unmodified 8.0 source from
`sys.tar.gz`, so `git diff HEAD~1` shows the change.

## The problem

On an RX 9070 XT (Navi 48, GC 12.0.1, DCN 4.0.1), with UEFI boot, the screen
shows noise for a few seconds while amdgpu starts. Everything works after it.

The monitor keeps scanning out the framebuffer the firmware (GOP) set up until
amdgpu does its first modeset. amdgpu only keeps that buffer out of its VRAM
allocator if it starts at VRAM offset 0: `gmc_v12_0_get_vbios_fb_size()`
returns 0, and OpenBSD's workaround (`efifb_stolen()`) reserves the right size
but at offset 0. When the firmware put the framebuffer somewhere else, ring
buffers and firmware images get allocated over it during init.

## The fix

- `efifb.c`: `efifb_stolen_paddr()` returns the framebuffer's physical
  address (it stays valid after `efifb_detach()`).
- `amdgpu_ttm.c`: `amdgpu_ttm_reserve_efifb()` turns that address into a
  VRAM offset using the VRAM BAR and reserves the range right after the usual
  stolen-memory reservations. It is freed in `amdgpu_ttm_fini()` along with
  them. It skips APUs, framebuffers outside this card's BAR, and ranges that
  are already reserved. If the range can't be reserved, it logs a message and
  boots as before.

amd64 only. The cost is the size of one framebuffer of VRAM (about 33 MB at
3840x2160).

## Building

On OpenBSD 8.0, with the matching `sys.tar.gz` in `/usr/src`:

    cp -R sys/* /usr/src/sys/     # or apply: git diff HEAD~1 | patch -p1 -d /usr/src
    cd /sys/arch/amd64/compile/GENERIC.MP
    make obj && make config && make -j$(sysctl -n hw.ncpuonline)
    cp /bsd /bsd.orig && cp obj/bsd /bsd

The boot-time kernel relinking (`reorder_kernel`) skips a kernel whose
checksum doesn't match `/var/db/kernel.SHA256`, so the custom kernel is kept.
It prints "Failed to verify /bsd's checksum" at boot. Don't run the
`sha256 -h` command it suggests: that would relink the stock objects from
`/usr/share/relink` and drop the fix on the next boot.

## License

The same licenses as the OpenBSD and Linux DRM sources: see the file headers
(mostly MIT/X11; ISC for OpenBSD-specific files).
