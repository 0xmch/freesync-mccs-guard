# Valve patch for `dm_freesync_mccs_ddc_worker`

The kprobe guard in this repo is a workaround. This directory is for the real
fix: a patch to Valve's kernel that Valve can apply, compile-tested, plus a way
to test it on the affected Legion Go S before it is sent.

Read [BUG-REPORT.md](../BUG-REPORT.md) first. The root cause is already proven
there; this work does not need to re-derive it.

## Facts

- The bug is in Valve's downstream amdgpu code, not mainline. The fix goes to
  Valve (ValveSoftware/SteamOS#2786), not to amd-gfx.
- Valve's tree is mirrored on GitHub at
  https://github.com/evlaV/linux-integration. Use the mirror, not
  gitlab.steamos.cloud.
- The device runs `7.2.0-valve1-1-neptune-72-gd39b4282853d` (tag
  `7.2.0-valve1`, commit `d39b4282853d`). Its `/proc/config.gz` is
  [config-7.2.0-valve1-1-neptune-72.gz](config-7.2.0-valve1-1-neptune-72.gz).
  It was built with `gcc (GCC) 16.1.1 20260625`. `CONFIG_MODVERSIONS` is off and
  `CONFIG_MODULE_SIG_FORCE` is off, so an unsigned out-of-tree module loads.
- The newest Valve kernel is `7.2.7-valve2`, which is also the head of
  `7.2/integration` (commit `31504582a073`, 2026-10-08). The worker is unchanged
  there: it still takes `&aconn->dm_dp_aux.aux.ddc` when `aux_mode` is set,
  without checking that the adapter was registered.
- On the device, `card0-DP-11` has no `ddc` link in sysfs, so that adapter was
  never registered. A correct check must be true in this case.

## Work

Fetch only what you need (`git clone --depth 1 --branch <tag>`, or
`git fetch --depth` for history). The full history is several GB.

1. **Find where the worker came from.** Identify the commit that added
   `dm_freesync_mccs_ddc_worker` and the one that schedules it on every VRR
   info change, with author and date. This gives the `Fixes:` tag and the
   person to contact. Write it to `AUTHOR.md`.
2. **Write the fix as separate, small patches** against `7.2.7-valve2`, and
   check that each applies to `7.2.0-valve1` too (note any rebase needed):
   1. Skip the DDC message when the selected I2C adapter was never registered.
      Prefer the same adapter selection and capability gating that the existing
      MCCS code in `amdgpu_dm_helpers.c` uses, if that fits; explain the choice
      in the commit message.
   2. Hold a reference on the connector while the work item is queued, and drop
      it in the worker. BUG-REPORT.md describes the use-after-free this closes.
      Check what else touches the queued work on connector teardown.
   3. Separately, a plain revert of the deferred worker, as an alternative
      Valve can choose instead of 1 and 2.

   Use `git format-patch`, kernel commit message style, a `Fixes:` tag, and
   `Signed-off-by: TODO` (the owner decides the identity before sending).
   Put them in `patches/`.
3. **Compile-test** amdgpu with the device config on both trees, with the
   patches applied: `make olddefconfig`, `make modules_prepare`, then build
   `drivers/gpu/drm/amd/amdgpu`. Use the newest gcc you can install and note
   the version. Touched files must build without new warnings at `W=1`. Kernel
   builds are long: run them in the background and poll, so commands don't
   time out. Record exact commands and results in `BUILD.md`.
4. **Make something the owner can test on the device.** It has the stock
   SteamOS headers package (`/usr/lib/modules/$KVER/build`, Arch-style
   `linux-neptune-72-headers`) and DKMS, and nothing else from the kernel
   source. Two options, pick the one you can make work and say why:
   - A DKMS package in `dkms/` that builds the patched amdgpu from the
     `7.2.0-valve1` sources against those headers and installs it in
     `updates/` so it overrides the stock module. Check it against a tree that
     is as close to the headers package as you can make it, and list what you
     could not verify.
   - If that is impractical: a variant of the kprobe guard that evaluates the
     patch's condition at the worker entry, logs whether the patched code would
     skip, and still skips. This proves the condition on real hardware without
     replacing amdgpu.
5. **Write `TEST-PLAN.md`** for the device: steps, pass criteria (no oops in
   `journalctl -k -b`, 4K 120 Hz still active, `TAINT_DIE` clear), and
   rollback. The bug only fires while docked, so an undocked boot is always the
   safe way back. During the test boot, `panic_on_oops=0` is set and the guard
   is disabled.
6. **Draft `ISSUE-COMMENT.md`** for ValveSoftware/SteamOS#2786: short, links to
   the patches on this branch, what was tested, and the author from step 1.
   Do not post it.

## Rules

- Push only to the `valve-patch` branch. Don't touch `main`, don't open issues
  or comments anywhere, don't email anyone.
- This repo is public. No real names, no IP addresses, hostnames, or details of
  the owner's other machines in anything you commit.
- Commit as `0xmch <2436477+0xmch@users.noreply.github.com>`.
- When something can't be verified in the sandbox, say so in the relevant file
  instead of guessing.
