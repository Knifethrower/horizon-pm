# Horizon on PortMaster handhelds

Patches that let [Horizon](https://github.com/horizon-rust/horizon), Minecraft: Java Edition 26.3 rewritten in Rust, run on
1 GB RK3326 handhelds such as the R36S through [PortMaster](https://portmaster.games), on OpenGL ES. They're the memory and
frame-rate work behind the *Minecraft Rust Edition* port.

**No Minecraft files are here.** Horizon needs your own copy of Minecraft: Java Edition 26.3, and the port prepares its
files from it on the device.

Vulkan work for the same devices is in progress and isn't in this repository yet.

## What's here

| Path | What it is |
|---|---|
| `patches/` | The series, applied in order to Horizon `26747867d` (see [Building](#building)): 0001, then 0029–0037 and 0041–0055. 0038–0040 are the Vulkan work and aren't here |
| `FINDINGS.md` | The measurements behind 0029–0037 |
| `spikes/` | Experiments that didn't make the series |
| `LICENSE`, `LICENSE.horizon.txt` | This repository's license, and Horizon's |

## Building

The port ships two programs built from this series for `aarch64-unknown-linux-gnu`: `horizon-client` as `mcrust.aarch64`
and `horizon-prepare` as `horizon-prepare.aarch64`. These steps cross-compile both on an x86-64 Debian or Ubuntu PC
against a Debian 11 arm64 sysroot, so the results need only glibc 2.30 and run on older firmware too.

### 1. The source

```bash
git clone https://github.com/horizon-rust/horizon.git
git clone https://github.com/Knifethrower/horizon-pm.git
cd horizon
git checkout 26747867d
git am ../horizon-pm/patches/*.patch
```

`git am` needs a git name and email. If you have none set, add `-c user.name=you -c user.email=you@example.com` after `git`.

### 2. The toolchain

- Rust 1.85 or newer (Horizon's `rust-version`) and its Arm target:
  ```bash
  rustup target add aarch64-unknown-linux-gnu
  ```
- The Arm cross compiler and pkg-config:
  ```bash
  sudo apt install gcc-aarch64-linux-gnu g++-aarch64-linux-gnu pkg-config
  ```
- A Debian 11 arm64 root to build against (the sysroot), with ALSA's and udev's development files. `debootstrap` makes one:
  ```bash
  sudo apt install debootstrap qemu-user-static
  sudo debootstrap --arch=arm64 --include=libasound2-dev,libudev-dev,libgcc-10-dev bullseye ~/sysroot-bullseye http://deb.debian.org/debian
  ```

### 3. A linker wrapper

Ubuntu's cross compiler takes libc and its startup files from its own `/usr/aarch64-linux-gnu`, even when given
`--sysroot`. A program that doesn't happen to pull in another sysroot library, such as `horizon-prepare`, then needs the
PC's glibc (2.39 on Ubuntu 24.04) instead of Debian 11's. This wrapper points everything at the sysroot:

```bash
SYSROOT=~/sysroot-bullseye
mkdir -p ~/xlink
cat > ~/xlink/aarch64-sysroot-gcc <<EOF
#!/bin/sh
exec aarch64-linux-gnu-gcc --sysroot=$SYSROOT -B$SYSROOT/usr/lib/aarch64-linux-gnu -L$SYSROOT/usr/lib/aarch64-linux-gnu \
  -L$SYSROOT/lib/aarch64-linux-gnu -L$SYSROOT/usr/lib/gcc/aarch64-linux-gnu/10 "\$@"
EOF
chmod +x ~/xlink/aarch64-sysroot-gcc
```

### 4. Build

From the `horizon` folder, in the same shell:

```bash
export PKG_CONFIG_ALLOW_CROSS=1 PKG_CONFIG_SYSROOT_DIR=$SYSROOT
export PKG_CONFIG_LIBDIR=$SYSROOT/usr/lib/aarch64-linux-gnu/pkgconfig:$SYSROOT/usr/share/pkgconfig
export CARGO_TARGET_AARCH64_UNKNOWN_LINUX_GNU_LINKER=$HOME/xlink/aarch64-sysroot-gcc
export CC_aarch64_unknown_linux_gnu=aarch64-linux-gnu-gcc CXX_aarch64_unknown_linux_gnu=aarch64-linux-gnu-g++
export CFLAGS_aarch64_unknown_linux_gnu=--sysroot=$SYSROOT CXXFLAGS_aarch64_unknown_linux_gnu=--sysroot=$SYSROOT
cargo build --release --target aarch64-unknown-linux-gnu -p horizon-client --bin horizon-client
cargo build --release --target aarch64-unknown-linux-gnu -p horizon-prepare
```

`release` is thin LTO with one codegen unit, as the port ships. For quicker test builds, `--profile fast` skips LTO, and
its output goes to `target/aarch64-unknown-linux-gnu/fast/` instead.

### 5. Into the port

```bash
T=target/aarch64-unknown-linux-gnu/release
aarch64-linux-gnu-strip -o mcrust.aarch64 $T/horizon-client
aarch64-linux-gnu-strip -o horizon-prepare.aarch64 $T/horizon-prepare
readelf -V mcrust.aarch64 horizon-prepare.aarch64 | grep -oE 'GLIBC_[0-9.]+' | sort -Vu | tail -1
```

The last command prints the newest glibc either program needs, GLIBC_2.30 with these steps. `min_glibc` in the port's
`port.json` must be at least that. Copy both files into the port's `mcrust/` folder. The next launch bakes the textures
again, because the patcher keys its bake on the game binary.

On an arm64 Linux machine you can skip steps 2 to 4: install `libasound2-dev`, `libudev-dev` and Rust, then run the two
`cargo build --release -p ...` commands without `--target`. The programs then need that machine's glibc.

## What they do

**0001** is the port's own changes by SLAYKIN6_TTV, their patches 0001–0028 combined into one. Applied to `26747867d`,
it gives exactly the tree the port was built from.

| Patches | What they do | Measured on an R36S |
|---|---|---|
| 0029–0030 | Music and records stream instead of decoding whole tracks, which was the out-of-memory kill; the sound cache is sized from the RAM | Audio from up to 117 MiB to about 28 MiB |
| 0031–0034 | Block sprite pixels spill to the card; the HUD atlas splits into a baked static part and a banded dynamic part | Atlas GPU memory 80 → 10–13 MiB |
| 0035 | The first launch of a build no longer holds a 540 MiB JSON tree | Was a kill on a 1 GB unit |
| 0036–0037 | No discard over opaque texels, and a plain terrain path | 13.8 → 21.9 fps, frozen scene, release build |
| 0041 | The tutorial ends once a gamepad is used: its popups couldn't be closed with one. It writes `tutorialStep:none`, as finishing the tutorial does | |
| 0042 | The animated sprite array stays an array on GL (water, lava, fire and kelp were black) | Black → correct |
| 0043 | Mesh pool sized by RAM | 17.2 → 4.1 MiB live |
| 0044 | Sound cache a 64th of RAM, not a 32nd | 23 → 15 MiB |
| 0045 | Meshing workers share the face tables | 14.6 MiB → under 2 MiB |
| 0046 | Animated frames kept at their own size | 9.4 → about 0.6 MiB |
| 0047 | Biome cache sized by RAM | Its bound halves, from 14.5 MiB |
| 0048 | Chunk encoder cache swept by RAM | Was nearly unbounded |
| 0049 | Structure template block NBT boxed | 22.6 → 13.0 MiB |
| 0050 | Parsed command registries not kept on small machines | 4.5 MiB |
| 0051 | Meshes waiting for upload count against the mesh jobs | Meshes in the peak 12–24 → 2–7 MiB, with 0052 |
| 0052 | Automatic chunk workers follow the RAM (2 on 1 GB) | |
| 0053 | Abandoned proto chunks saved in small batches | Up to 16 MiB of NBT at once before |
| 0054 | Block states share equal property maps | Heap at rest 292 → 271 MB |
| 0055 | A, B, X and Y follow PortMaster's mapping file, so they act as labeled | |

Each memory patch reads the machine's RAM rather than a device name, so a machine with plenty behaves as before, except
0054, which applies everywhere. Each patch's own message has its hypothesis, numbers and how to falsify it.

## Results on an R36S

R36S with 2 GB held to a 1 GB unit's conditions (ballast, no swap, MemTotal of 1 GB), dArkOS, Mali-G31, GL, `fast` builds,
render distance 2, simulation distance 5:

| At 590 MB available | 0029–0042 | 0029–0054 |
|---|---|---|
| First far teleport | **killed** | survives |
| Twelve far teleports, 30 s apart | not reached | no kill, lowest available 84 MB |
| Nether through a portal | not run | no kill, lowest 104 MB * |
| End, main and an outer island | not run | no kill, lowest 122 MB * |
| Jump into a small pool | not run | no kill, lowest 125 MB * |

\* Run without 0054. At 560 MB available, 0029–0054 is killed on the second far teleport, so the floor is between 560 and
590 MB. Check yours with `grep MemAvailable /proc/meminfo` with the front end at its menu.

What it costs: a far teleport's chunks arrive in 24–34 s instead of 20–30 s. The frame rate doesn't change.

If memory is still tight, two player settings help: fewer `mipmapLevels` in `options.txt` (about 5 MB of GPU memory per
level, calculated rather than measured), and `workers:1` in `horizon-graphics.txt` (slower loading).

## Known problems

- The RGBA fallback, used when the texture bake is missing, has hung in the Mali driver on GL.
- The game panics on a clean stop under Westonpack (`khronos-egl`, `Option::unwrap()` on `None`). It only writes a crash
  report at exit.
- Once, after a visit to the Controls menu on a test build, analog stick movement became very slow, and `options.txt`
  came back with `maxFps:30`, `cutoutLeaves:true` and `vignette:true`. Not reproduced, explained or checked on GL yet.

## Not tested

A release build of the whole series, long play by hand, survival over time, a real ocean, whether re-decoding sounds
(0044) causes hitches, and the buttons on the real pad (0055 was tested with a virtual copy of it).

## License

[MIT](LICENSE). The patches change Horizon, which is MIT too: see [LICENSE.horizon.txt](LICENSE.horizon.txt).
Minecraft is Mojang's; nothing of it is here.
