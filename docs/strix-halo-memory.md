# Strix Halo 128 GiB shared GPU memory profile

This automatically detected host profile sets `ttm.pages_limit=20971520`: an 80 GiB ceiling
for GPU-accessible system memory with 4096-byte pages. Memory is allocated
on demand; it is not reserved at boot. CPU workloads, VMs and GPU workloads
still compete for the same physical RAM.

The intended BIOS UMA/framebuffer reservation is 16 GiB for gaming. Together
with the dynamic ceiling, this leaves roughly 32 GiB of theoretical headroom
out of 128 GiB, before other hardware reservations. This is not guaranteed
free RAM or a guarantee that an application can allocate 96 GiB on the GPU.
The service does not change or enforce the BIOS reservation.

The source of truth is
`files/system/usr/lib/systemd/system/niri-strix-halo-128g-memory.service`.
The image ships the unit and its runtime helper through the existing files
module. The systemd module enables it at boot. Before changing any kernel
arguments, the service requires all of the following:

- A bare-metal x86-64 host booted under OSTree.
- An AMD Ryzen AI Max+ 395 CPU, identified through `/proc/cpuinfo`.
- An AMD Strix Halo GPU with PCI ID `1002:1586`.
- Exactly 128 GiB of installed RAM, summed from SMBIOS type 17 memory devices
  using `dmidecode`, independently of the BIOS GPU reservation.
- 4096-byte memory pages.

Other hardware is skipped. Missing, unreadable or unknown memory information
also skips the profile without modifying kernel arguments. This targets a
hardware configuration, not a hostname, serial number or specific computer.

## Apply

Use this profile on a Strix Halo machine with 128 GiB RAM and a recent kernel
(AMD documents Linux 6.18.4 or later for non-Ubuntu distributions).

1. Keep the BIOS UMA/framebuffer reservation at 16 GiB for gaming. This fixed
   reservation is unavailable to CPU workloads; the 80 GiB dynamic ceiling
   balances GPU workloads with host memory headroom.
2. Boot an image containing this profile. It runs automatically; inspect it with:

   ```bash
   journalctl -u niri-strix-halo-128g-memory.service -b --no-pager
   rpm-ostree kargs
   ```

3. If a change was staged, reboot again when ready to activate the memory limit.
   Any already staged OS update will also take effect. The first boot of the new
   image configures the next boot; it cannot change the running kernel's limit.

The helper reads the default deployment's arguments, replaces existing
`ttm.pages_limit` values in one transaction and preserves other arguments.
If the desired value is already present exactly once, it makes no changes.
Failures from rpm-ostree are reported by the service; retry after resolving
them (for example, an in-progress update or insufficient disk space) with
`sudo systemctl start niri-strix-halo-128g-memory.service`, or on the next boot.

The setting persists across normal rpm-ostree upgrades. If the profile value
changes in a future image, the service stages the new value at boot.
Rollback to an older deployment can restore that deployment's older arguments.
The desired value is reconciled at each boot, with no automatic reboot.

## Verify after reboot

```bash
cat /proc/cmdline
cat /sys/module/ttm/parameters/pages_limit
grep . /sys/class/drm/card*/device/mem_info_{vram,gtt}_total
free -h
```

Expect `pages_limit` to be `20971520`, GTT to be approximately 80 GiB
(`85899345920` bytes), and the VRAM reservation to match the 16 GiB BIOS setting.
Linux should see less than 112 GiB of system RAM after other hardware reservations.

Podman containers share this host limit. Existing llama.cpp Vulkan Quadlets
using `/dev/dri` require no TTM setting inside the container. Check llama.cpp's
startup logs for the actual Vulkan memory budget and GPU allocations; a GTT
ceiling does not guarantee that an application can allocate the entire amount.
Model weights, KV caches, context sizes and parallel requests all contribute
to memory usage. Leave space for the desktop and other host workloads.
Avoid running a large GPU-resident model alongside a memory-heavy game.

## Remove

First mask the service to prevent automatic reapplication (also across image
updates), then remove the profile's argument from the default deployment and reboot:

```bash
sudo systemctl mask --now niri-strix-halo-128g-memory.service
sudo rpm-ostree kargs --delete-if-present=ttm.pages_limit=20971520
```

If you changed the profile value, substitute that value in the command.
The driver returns to its default limit if no other override is present.
The BIOS reservation is independent and can remain at 16 GiB.

To restore automatic management:

```bash
sudo systemctl unmask niri-strix-halo-128g-memory.service
sudo systemctl enable --now niri-strix-halo-128g-memory.service
```

Reference: [AMD RDNA3.5 memory optimization](https://rocm.docs.amd.com/en/latest/how-to/system-optimization/strixhalo.html).
