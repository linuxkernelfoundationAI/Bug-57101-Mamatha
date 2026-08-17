# Build fixes — bug 57101 (fscache/stats leak) repro

Target: Linux v3.9-rc8 (pre-fix, `fs/fscache/stats.c` untouched), built in the
`kernel-builder` Docker container (ubuntu:14.04, gcc 4.8.4.2).

## Toolchain gaps found and fixed

1. **`libelf-dev` missing.** Fresh ubuntu:14.04 image had `libssl-dev` and
   `ncurses-dev` already present but not `libelf-dev`. Installed via
   `apt-get install -y libelf-dev` inside the container. Not required by
   `bzImage`/`modules` targets on this config, but installed proactively to
   avoid a stall on any host-tool that links against libelf.
2. No other toolchain issues encountered. `make defconfig`, `./scripts/config`
   option toggles, `make olddefconfig`, and `make -j$(nproc) bzImage modules`
   all completed with **exit code 0** and **zero compiler errors** on the
   first full attempt (see `proof/build.log`).

## fs/fscache/stats.c

Not modified. Verified before and after the build that
`fscache_stats_fops.release` is still wired to `seq_release` (not
`single_release`), i.e. the bug is present in the binary that was booted.
