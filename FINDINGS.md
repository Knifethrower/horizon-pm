# Horizon on the R36S: where the OOM while playing comes from

For the port owner. Measured on 2026-11-04 against `hz-base` as received (upstream `26747867d` +
patches 0001–0028). Patches, to apply in order after 0028:

- [patches/0029-audio-stream-music-instead-of-decoding-whole.patch](patches/0029-audio-stream-music-instead-of-decoding-whole.patch): the OOM.
- [patches/0030-audio-size-the-sound-cache-by-the-machine-s-RAM.patch](patches/0030-audio-size-the-sound-cache-by-the-machine-s-RAM.patch): up to 36 MiB more back on a 1 GB device ([below](#0030-the-sound-cache-sized-by-the-ram)).
- [patches/0031-render-spill-the-block-sprite-pixels-once-a-renderer.patch](patches/0031-render-spill-the-block-sprite-pixels-once-a-renderer.patch): 20 MiB of block-texture pixels no longer held in RAM all session ([below](#0031-the-block-sprite-pixels-spilled-to-the-card)).
- [patches/0032-render-keep-the-shared-atlas-s-run-time-rows-in-a-te.patch](patches/0032-render-keep-the-shared-atlas-s-run-time-rows-in-a-te.patch): the HUD atlas split into a static and a dynamic texture (your §6.8). No memory change by itself: it's what makes 0033 possible ([below](#0032--0033-the-hud-atlas-split)).
- [patches/0033-render-load-the-shared-atlas-s-static-rows-from-thei.patch](patches/0033-render-load-the-shared-atlas-s-static-rows-from-thei.patch): the static part from its ASTC bake: the atlas takes 55 MiB of GPU memory instead of 80. **Needs a rebake** (your `patch.bash` does it when the binary changes).

- [patches/0034-render-hold-the-shared-atlas-s-dynamic-rows-in-bands.patch](patches/0034-render-hold-the-shared-atlas-s-dynamic-rows-in-bands.patch): the atlas's dynamic part held in 64-row bands, only the ones written to: 46 MiB → 1–4 MiB of GPU memory ([below](#0034-the-atlas-s-dynamic-part-held-in-bands-as-it-is-written)).
- [patches/0035-world-read-gameplay-facts.json-state-by-state-for-th.patch](patches/0035-world-read-gameplay-facts.json-state-by-state-for-th.patch): the first launch of every new build no longer holds a 540 MiB JSON tree, which got the game killed on a 1 GB R36S ([below](#0035-the-first-launch-of-a-build)).
- [patches/0036-render-draw-faces-over-opaque-texels-without-discard.patch](patches/0036-render-draw-faces-over-opaque-texels-without-discard.patch): the first frame rate patch. About 14 → 16–17 fps at low settings, same pixels ([below](#0036-faces-over-opaque-texels-drawn-without-discard)).
- [patches/0037-render-a-plain-terrain-path-when-only-sprite-light-a.patch](patches/0037-render-a-plain-terrain-path-when-only-sprite-light-a.patch): 16–17 → 21–23 fps, one pixel in 307,200 differs, about 20 MiB more memory ([below](#0037-a-plain-terrain-path)).
- 0041 onward: in the [README](README.md) and each patch's own message. 0038–0040 aren't in this repository.

**Frame rate with the whole series, release build, measured in one sitting on the R36S: 13.8 → 21.9 fps**
at low settings and render distance 2 ([table](#the-release-build-in-one-sitting)). That is short of the
60 asked for, and [what's left](#whats-left-for-the-frame-rate) says why I don't expect 60 on GL.
The [README](README.md) has the latest status.

**0032 and 0033 were re-exported** (they fix a test of yours my first 0032 broke, see the correction
below): use these files, not earlier copies.

Together, at the R36S's 64 px sprite cells: audio drops from up to 117 MiB (typical track playing,
full cache) to about 28 MiB, the sprite pixels take 20 MiB less, and the HUD atlas 67–70 MiB less
GPU memory (80 → 10–13 MiB), which on the R36S is the same 895 MiB. That's about 180 MiB in normal
play. In creative mode the audio saving is up to 123 MiB, in the End up to 188 MiB. The audio
figures are ceilings worked out from the files and the cache limits. The parts were measured one
patch at a time on PC (below); 0034 and the whole series also ran on an R36S held to 1 GB
([below](#on-an-r36s-at-1-gb-with-00290034)).

The [in-world memory breakdown](#where-the-rest-of-the-memory-is-in-a-world) ranks what's left.

## Summary

**Music is the OOM.** Horizon "streams" music and records (`stream: true` in `sounds.json`) by
decoding the whole track into one mono `Vec<f32>` on a thread, then holding that `Vec` for as long
as the track plays (`MixerBackend::start` → `decode` in
`crates/horizon-client-render/src/audio.rs`). On an R36S already at ~430 MiB that is enough to run
out of memory, at whatever moment the music manager starts a track.

| Track (26.3 assets) | Decoded size | When it plays |
|---|---:|---|
| median of the 70 music tracks | 52.7 MiB | game, menu |
| `music/menu/beginning_2` (shortest menu track) | 30.7 MiB | title |
| `music/game/nightly`, `home`, `endless` | 55–74 MiB | title + game |
| `music/game/creative/*` | 61–87 MiB | **creative mode** |
| `music/game/end/alpha` | 101 MiB | credits |
| `music/game/end/the_end` | 152 MiB | **the End** |

The `Vec` grows by doubling while it decodes, so the peak while a track starts is up to about
twice its size. The sound cache doesn't count streams, so nothing bounds this.

Vanilla doesn't do this. 26.3's `com.mojang.blaze3d.audio.Channel` has `QUEUED_BUFFER_COUNT = 4`
and `BUFFER_DURATION_SECONDS = 1`, so it keeps 4 seconds of a streamed track decoded, not the
whole thing (checked with `javap` on the pinned 26.3 client jar).

## Measured (PC, R36S-like run)

Same harness for both: WSL, 4 CPUs (`taskset -c 0-3`), 640×480, wgpu GL backend on Mesa llvmpipe
(GPU memory in-process, like the Mali), sound into a FIFO drained at exactly 44.1 kHz × stereo × f32
so tracks last as long as on a device, returning player (`onboardAccessibility:false`), title screen.
Absolute RSS is higher than on the R36S (llvmpipe, x86 code); the music deltas are CPU-side heap
and carry over as they are.

| Run | Track | RSS just before → after it starts |
|---|---|---|
| baseline | `beginning_2` (30.7 MiB) | 542 → 575 MiB (**+33**), held for the whole 3 min track |
| baseline, unpaced | `nightly` (55.3 MiB) | 579 → 638 MiB (**+59**) |
| baseline, unpaced | `home` (69.6 MiB) | 585 → 656 MiB (**+71**) |
| **0029** | `nightly` (55.3 MiB) | 558 → 558 MiB (**+0**), for the whole 5 min track |

With 0029 the music plays continuously. 60 s of the game's actual output were captured: audible
from the track's start to the end of the capture, with no silent gap mid-track (the only zero run
is the track's own 18.7 ms lead-in). The baseline, captured the same way, behaves the same.

## The fix (0029)

`crates/horizon-client-render/src/audio.rs`, one file, +322 / −48:

- A streamed sound gets a decoder thread (`sound stream`) that keeps a queue
  `STREAM_SECONDS_AHEAD = 4` seconds ahead of the mixer, vanilla's 4 × 1 s. It starts the channel
  once the queue is full, as `attachBufferStream` pumps before `play`. It loops by reopening the
  file. It returns at the end of the track, or when the channel lets go of the stream (released, or
  stopped before it started), so no thread is left behind.
- The mixer reads a streamed voice from the queue with the same interpolation as a whole sound,
  then drops what it has played and wakes the decoder. If the decoder hasn't caught up, the voice
  plays nothing and holds its place, like an OpenAL source out of buffers.
- Whole sounds (the cache) are now trimmed with `shrink_to_fit`. A `Vec` grown by doubling could
  hold up to twice the samples `SoundCache` counts against its 64 MiB.
- No setting, device or size is assumed. The queue length comes from the file's own sample rate
  times vanilla's constant.
- Tests: `a_streamed_sound_plays_as_the_whole_sound_with_a_bounded_queue` (a decoder thread feeds
  the queue; every mixed buffer equals the whole sound's, and the queue never exceeds 4 s plus one
  packet) and `a_stream_nobody_plays_stops_decoding`. All 20 `audio::` tests pass; the two new
  ones passed 50 runs out of 50.

### Checked

- `cargo build --profile fast -p horizon-client` (x86_64): clean.
- `cargo test --profile fast -p horizon-client-render --lib -- audio::`: 20 passed.
- `git apply --check` and `patch -p1 --dry-run` against a fresh unzip of the received tree: clean.
- `cargo clippy -p horizon-client-render --lib -- -D warnings`: nothing in `audio.rs` (the crate
  still fails on 9 lints from earlier patches, see below).
- Not done: the aarch64 cross `cargo check` from spec §4.1 (this machine has no aarch64 ALSA/udev
  pkg-config), and of course the device.

### On the device

After installing a build with 0029 (no rebake needed; it doesn't touch textures):

```sh
# in a second SSH session while playing; RSS once a second
while sleep 1; do grep VmRSS /proc/$(pgrep -x mcrust.aarch64)/status; done
```

Expected: RSS no longer jumps by 30–90 MiB when a track starts (creative mode and the End are the
worst cases), and the "random" kills while playing stop. Music should sound unchanged.

## 0030: the sound cache sized by the RAM

The cache of decoded static sounds (`SoundCache` in `audio.rs`) kept up to a fixed 64 MiB of
`f32` whatever the machine. 26.3 has 4843 static sound files that decode to 1375 MiB, so over a
session the cache fills to whatever limit it has and stays there: the Nether's ambience alone is
166 MiB of variants, caves 34 MiB (`tools/static_sizes.py`).

0030 gives it a thirty-second of `MemTotal`, never more than the old 64 MiB:

| RAM | Budget |
|---|---:|
| 2 GiB and up, or unknown (no `/proc/meminfo`) | 64 MiB, as before |
| R36S, 895 MiB | 27 MiB |
| 512 MiB | 16 MiB |

It reads `MemTotal` once and logs `Horizon: sound cache budget N MiB`, so the device log shows
what it chose. Past the budget the least recently used sound goes and is decoded again the next
time it plays. Same LRU as before, only the limit changed. Sounds already playing keep their
samples. No option, device or RAM size is assumed. It reads `MemTotal` itself, as
`sprites::array_budget` does, so it doesn't touch `sprites.rs`.

Measured with both builds seeing an 895 MiB `MemTotal` (a private mount namespace with a
`/proc/meminfo` copy, so the game sizes everything as on the R36S). The same scripted 5-minute
creative scene ran twice: night, thunder, 30 mobs (villagers, golem, farm animals, hostiles), the
player walking and jumping, then the Nether (`tools/scene_compare.sh`):

| Build | Cache after 5 min | Static sounds decoded | Decoded a second time |
|---|---:|---:|---:|
| 0029 (64 MiB) | 34 MiB, 83 sounds, still growing | 89 | 0 |
| 0030 (27 MiB) | 27 MiB, 51 sounds, holding | 76 | **0** |

So in a busy scene the 27 MiB budget dropped old sounds without a single re-decode. The saving
in that run is only 7 MiB, because 5 minutes didn't fill 64 MiB. A longer session with more
places and mobs grows the old cache to 64 MiB, which is 36 MiB more than 0030 keeps on the
R36S. Whole-process RSS on llvmpipe varies by more than that between runs (world generation),
so the cache's own counter (`status` → `sounds.decoded_mib`) is the measure here, not RSS.

Checked: the 21 `audio::` tests pass (new: `the_sound_cache_budget_follows_the_machines_ram`; the
LRU test now uses the cache's own budget, so it passes on any machine). Nothing from clippy in
`audio.rs`. `rustfmt --check` on `audio.rs` is clean. 0029 then 0030 apply with `git apply` and
`patch -p1` to a fresh unzip of the received tree, and give the tested `audio.rs` byte for byte.
Not done, as for 0029: the aarch64 cross check and the device. On the R36S the log should say
`sound cache budget 27 MiB`.

## 0031: the block sprite pixels spilled to the card

`RenderAssets.pixels` is the block sprite arrays' top level, every layer's RGBA in turn: 1291
layers × 64×64 × 4 = **20.2 MiB** at the R36S's cells. It stayed in RAM all session, though
nothing reads it once a renderer has uploaded the arrays. The ASTC bake doesn't change that: the
renderer still packs its key and the animated layers' first frames from these bytes. They are
needed again only when another renderer is built: another graphics API, a splitscreen player
joining, or the window coming back on resume. A resource-pack reload brings its own fresh pixels.

0031 does with them what the game already does with the GUI atlas's 80 MiB (`hud-atlas-*.rgba`):

- Once a renderer is built, a short-lived thread (`sprite spill`) writes them to
  `cache/sprite-pixels-<size>-<layers>l-<key>.rgba`, next to the atlas spill, and drops the CPU
  copy. The thread ends when it's done, so it isn't a daemon. Hashing and writing 20 MiB to the
  card stays off the render thread.
- The name's key is the bytes' own `astc::content_key`, so a later launch with the same textures
  finds its file and writes nothing. A file of that name always holds these bytes, even one left
  by another copy of the game. The file is written under a `.part` name and renamed, so no reader
  sees half of it. Spills of other textures in the folder are deleted, so changing resource packs
  doesn't pile up 20 MiB files.
- `build_renderer` reads the bytes back through `SpritePixels::get` (from memory, or the file
  checked by its length). If the file can't be written, the bytes stay in memory, as before.
- No option, size or path is assumed: the size and layer count in the name come from the pack,
  and the folder is the one the atlas spill uses.

Files: `assets.rs` (`SpritePixels`), `game.rs` (`build_renderer`, `spill_sprite_pixels`;
`prepare_renderer_textures` now returns a `Result`), `game/packs.rs`, and
`examples/sprite_cache_parity.rs`, which hashed the field.

Measured (PC, the R36S's MemTotal, default options = render distance 12, 4 minutes in a creative
world, heaptrack on a system-allocator build, what was still allocated when the run stopped):

| | 0030 | 0031 |
|---|---:|---:|
| Live heap | 501 MiB | 474 MiB |
| `RenderAssets::load_with` (the sprite pixels) | 20.2 MiB | gone |
| GPU memory inside llvmpipe | 209 MiB | 210 MiB |

The other ~7 MiB of the difference is structure templates: this run generated fewer structures,
so that part is noise. In the game:

1. **First launch:** `block sprite pixels spilled to …/cache/sprite-pixels-64-1291l-94038058d7387ea1.rgba
   and their CPU copy dropped`. The file is 21,151,744 bytes.
2. **Second launch, same folder:** the file was found and not rewritten (same modification time).
3. **Splitscreen join (`HORIZON_SPLIT=2`):** both players' renderers were built from the file, and
   the screenshot shows correct block textures in both views. This needed a software renderer (lavapipe): on
   this PC's GL setup the join fails with "surface does not support copying offscreen frames",
   with 0030 too, and the game then keeps its old renderer.

Checked: `assets::tests::spilled_sprite_pixels_come_back_from_their_file`, a new test (round trip;
the same bytes find their file; other textures replace an older spill; an unwritable folder keeps
the bytes in memory), plus the 21 audio tests. Both examples compile, and clippy finds nothing in
`assets.rs` or in the changed lines of the game crate. `rustfmt` is clean on the changed lines
(`game.rs` already has three unformatted spots from earlier patches). 0029–0031 apply in order to
a fresh unzip and give the tested files byte for byte (`tools/apply_check.sh`). Not done: the
aarch64 check and the device. On the R36S the log should show the `spilled` line once a world
opens, and `conf/cache/` gains one 20 MiB file.

## 0032 + 0033: the HUD atlas split

Your §6.8, done as two patches. The shared atlas (GUI, items, entities, particles, paintings,
glyphs; 4096×5120 RGBA = 80 MiB of GPU memory) couldn't be ASTC because the game writes into it
while it runs. 0032 moves everything written at run time into a texture of its own; 0033 loads the
rest from its ASTC bake.

| | Static part | Dynamic part | GPU memory |
|---|---|---|---:|
| 0031 | one texture, 4096×5120 RGBA | | 80 MiB |
| 0032 | 4096×2152 RGBA (33.6 MiB) | 4096×2968 RGBA (46.4 MiB) | 80 MiB |
| 0033 with the bake | 4096×2152 ASTC 4x4 (8.4 MiB) | 4096×2968 RGBA (46.4 MiB) | **55 MiB** |

**0032, the split** (`hud.rs`, new `gfx/shared_atlas.rs`, the four WGSL files that read the atlas):

- `HudAtlas::pack` packs the images the game writes at run time (map, skin, trim, world-icon and
  pack-icon slots, the minimap, the on-screen keyboard's text, the animated sprites) below all the
  others, from an ASTC block row (`static_rows`), with the glyph rows last. Packed apart with the
  old shelves, the atlas grew to 6144 rows, so the layout is now a bottom-left skyline. That fits
  both parts in the same 5120 rows the mixed shelves needed, so a GPU without ASTC uses exactly
  what it did before.
- `SharedAtlas` holds the two parts as two textures and sends every run-time write to the part it
  falls in. Atlas coordinates keep their meaning: `atlas_sample`, `atlas_load` and `atlas_size` in
  `hud.wgsl`, `entity.wgsl`, `screen.wgsl` and `oit_clouds.wgsl` read the two as one atlas as tall
  as both. That needs no new uniform, only the two textures' sizes (`textureDimensions`). The HUD
  bind group gains binding 5 and the entity one binding 8. Samples are explicit level 0, which is
  what the implicit ones read from a one-level atlas, and they're valid in any control flow.
- Three things sample the atlas without those shaders: a resource pack's text shader, a shader
  pack, and a native mod. They get `SharedAtlas::combined`, the whole atlas in one texture, made
  the first time it's asked for and kept up to date after that. It costs the old 80 MiB again, so
  only they make it. **Native mod API change:** a mod that wants `WorldDrawn::atlas` now sets
  `RenderNeeds::atlas`; otherwise it's `None`.

**0033, the bake** (`astc.rs`, `gfx.rs`, `game.rs`'s `bake_atlas`):

- `--bake-atlas` (run by your patcher) now encodes only the static rows, to
  `cache/hud-atlas-<width>x<rows>-<content key>.astc` (8.4 MiB), and deletes earlier atlas bakes.
  The old whole-atlas `hud-atlas-4096x5120.astc` was never loaded by anything. Its name was only
  the atlas's size, so it could also have matched another pack's atlas. The key is
  `HudAtlas::static_key`, hashed while the packed pixels are still in memory (FNV over ~34 MiB,
  tens of ms).
- The renderer loads the bake when the GPU samples ASTC (`astc::supported`, a capability) and the
  RGBA spill is there, then streams only the dynamic rows from the spill: a launch reads 46 MiB
  from the card instead of 80. Without the bake or the capability, nothing changes. The whole-atlas
  copy reads its static rows from the spill, since ASTC can't be copied into an RGBA texture.

**On the device:** after installing, `patch.bash` must rebake (its stamp sees the new binary).
Expected: `cache/` holds one `hud-atlas-4096x2152-….astc` (8.4 MiB) and no `hud-atlas-4096x5120.astc`,
and the log says:

```
Horizon: shared atlas static rows 4096x2152 from their ASTC bake (8 MiB), dynamic rows 4096x2968 RGBA (46 MiB)
```

Visual pass worth doing: HUD text, item icons in the hotbar, mobs, paintings, particles, and a map
in an item frame or in hand.

**Checked on PC:**

- **Tests:** `hud::` 29 (new: written images go below the fixed ones from a block row; skyline
  placements keep a texel apart), `gfx::shared_atlas` 2 (rows land in the part they fall in, also
  across the boundary), `astc::` 1 (the bake's name), plus the shader-translation test over every
  WGSL file, and `audio::`/`assets::`.
- **Linters:** clippy finds nothing in the new code (the render crate's older lints are down from
  9 to 8: one was in the dead ASTC code 0032 removed). `rustfmt` is clean on the changed lines.
- **0032 in the game:** compared frame for frame against 0031 in a still scene (`tools/atlas_scene.py`:
  frozen ticks, NoAI mobs, a held filled map, animated hotbar items, a chat line of characters the
  bitmap fonts lack, so both parts are read). The two builds' frames differ by no more than two
  runs of the same build do, except 36 of 307,200 pixels. Those take a neighbouring texel's colour
  on exact texel edges (float rounding of the row arithmetic), the same thing any change of sprite
  positions causes.
- **0033 in the game:** with the bake on Mesa's GL (llvmpipe samples ASTC), the log above, and the
  frame draws normally. Bitmap-font text is texel for texel what RGBA gives; GUI sprites differ in
  a few edge shades. llvmpipe stores ASTC decompressed, so the PC can't show the memory saving; the
  55 MiB is the format's size (ASTC 4x4 = 1 byte a texel).
- **Splitscreen** (the rebuild path) on lavapipe, and the whole series applied to a fresh unzip.
- **On a device:** both load their bake and draw on an R36S (see 0034's section).

**Found while testing (not patched):**

- **Scripted runs never use any ASTC bake.** With `--control-socket` (and other automated
  launches) `synchronous_startup` skips the loading screen, so the atlas is never spilled, and the
  sprite-array and atlas bakes are only looked for next to the spill. Players aren't affected, but
  a scripted measurement sees RGBA everywhere.
- **Possibly a stale atlas after a pack reload:** `Renderer::new` reuses the previous renderer's
  atlas whenever the size matches, ignoring new pixels (now: the size and static rows). A resource
  pack whose atlas packs to the same size would keep the old textures. Not reproduced, only read.

**What was left in the atlas:** the dynamic part, 46 MiB of RGBA, mostly reservations at desktop
counts. 0034 holds only the parts of it that get written ([below](#0034-the-atlas-s-dynamic-part-held-in-bands-as-it-is-written)).

**Correction (re-exported 0032 and 0033):** the 0032 I first sent broke your GPU test
`gfx::end_sky::tests::separate_draws_match_combined_celestial_pixels`. The test makes its bind group
from the entity shader's own layout, which gained the `atlas_dynamic` binding, and my checks didn't
run it. The test now binds a `SharedAtlas` (0032; 0033 adds the `None` bake argument). With that, the
whole render crate's lib tests fail only where the tree as received does
(`gfx::panorama::tests::original_resolution_texels_and_vanilla_cube_orientation_are_preserved`).

## 0034: the atlas's dynamic part held in bands as it is written

After 0033 the atlas's dynamic part was still a full 4096×2968 RGBA texture: 46 MiB of GPU memory,
nearly all of it empty slots. 0034 holds it in 64-row bands, only the ones something was written to.

| | Dynamic part on the GPU |
|---|---:|
| 0033 | 46.4 MiB, always |
| 0034, title screen | 1 band, 1 MiB |
| 0034, in a world with a map in hand, animated items and characters the bitmap fonts lack | 4 bands, 4 MiB |

What the dynamic part of the default resources' atlas holds (rows 2152–5120 of 4096×5120):

| Rows | What | RGBA | Used when |
|---|---|---:|---|
| 1024 | on-screen keyboard text and corner masks (`horizon/ui-text`, 2048×1024) | 8 MiB | the theme's text, filled from its top |
| ~1290 | 256 map slots, 128×128 | 16 MiB | a map in hand or in a frame, lowest free slot first |
| | 64 pack-icon slots, 128×128 | 4 MiB | the pack screen |
| | 128 skin slots, 64×64 | 2 MiB | other players' skins |
| | 32 world-icon slots, 64×64 | 0.5 MiB | the world list |
| 32 | 54 animated GUI sprites, 32 trim slots, the minimap | 0.4 MiB | always (the sprites), armour trims, the minimap extra |
| ~910 | glyph rows (512 reserved + the height rounded up to 1024) | 14 MiB | characters the bitmap fonts lack, row by row |

Everything there fills from its own start, so most of the 46 MiB is never written in a session.

**How** (`gfx/shared_atlas.rs`, the three WGSL files with the atlas helpers, `hud.rs`, `gfx.rs`):

- The atlas's layout and coordinates don't change. The dynamic part is cut into bands of
  `BAND_ROWS` (64) rows, a MiB each at 4096 across. A band is held from the first write into it:
  it becomes the next band of a texture that grows by it, with the bands held so far copied over
  on the GPU. Writes go to where their band is held, one copy for bands held one after the other.
- A page table (a 1 KiB uniform: the atlas's rows, the band size, then each band's place)
  tells the shaders where each band is. `atlas_sample`, `atlas_load`, `atlas_nearest` and
  `atlas_linear` read the static part as before. For a row of the dynamic part they look its band
  up, and a band never written reads as transparent black, as the empty slot did. `atlas_size`
  now reads the atlas's height from the table, since the texture is shorter than the atlas.
  The HUD bind group gains binding 6, the entity one binding 9.
- The initial upload (the packed pixels, or the spill streamed back) holds only the bands with
  something in them: the animated sprites' first frames. At load that's one band.
- When the texture grows, the bind groups that hold its view are remade before the frame's first
  pass: the HUD's in `HudRenderer::prepare`, the entity and hand groups in `draw_frame`, after
  every write of the frame. `SharedAtlas::generation` says when.
- `SharedAtlas::combined` (resource-pack text shaders, shader packs, native mods) copies each held
  band to its place, so those users see the whole atlas as before.
- Growing holds the old and the new texture until the frame is done: at most twice what's held,
  briefly. A session that fills every slot ends where 0033 always is.
- Linear filtering across a band edge (blurred sprites only, `screen.wgsl`) reads the next band
  in the texture. At load the bands with content are held in order, so an animated sprite's
  neighbours across an edge are its own. The other run-time images are drawn nearest.

**Checked on PC:**

- **Tests:** `gfx::shared_atlas` 7, among them a GPU one (bands held only for what was written,
  the texture grows, the atlas reads back whole and right through `combined`) and one that the
  page table matches the shaders' array size. The render crate's lib tests: 835 pass; the 4 that
  fail are panorama (failing as received) plus two `crash::` tests and one `graphics_ui::` test,
  in files no patch of mine touches.
- **Frames:** the still atlas scene (map in hand, animated hotbar items, a chat line of unifont
  characters) against 0033: text, static items and the map are pixel for pixel the same. What
  differs moves with time between runs: animation frames, clouds, a mob's pose, the map's fill.
- **Memory, title screen** (R36S MemTotal, two runs each, last 10 s): RSS 553/540 MiB with 0033,
  508/512 with 0034, so about 36 MiB less. On llvmpipe the textures live in the process; on the
  R36S it's Mali memory from the same 895 MiB.

**Checked on an R36S** (our 2 GB R36S held to a 1 GB R36S's conditions: front end stopped, no
swap, a locked ballast leaving 746 MB available, the game shown a 1 GB MemTotal; aarch64 builds of
0033 and 0034, the ASTC bakes made with the same code; title screen, last 30 s):

| | Mali GPU memory | Game RSS | MemAvailable |
|---|---:|---:|---:|
| 0033 | 124 MB | 468 MB | 358 MB |
| 0034 | 78 MB | 439 MB | 407 MB |

46 MB less GPU memory, the dynamic part exactly, and 49 MB more available. The log line on the
device: `shared atlas static rows 4096x2152 from their ASTC bake (8 MiB), dynamic rows 4096x2968
RGBA in 64-row bands, 1 of 47 held (1 MiB)`.

## 0035: the first launch of a build

The first launch of each new binary has no snapshot of the behaviour tables yet
(`.horizon-cache`), so the world generator's `Behaviours::load` builds them itself, starting
before your rest load has stored one. It parsed `gameplay-facts.json` (55.7 MB) whole into a
`serde_json::Value`: **542 MiB** of the 962 MiB live at the heap's peak (heaptrack, on PC), held
through the build next to the renderer and your rest tables.

0035 keeps each state only as the `GameplayProjection` your typed path already uses: one state's
tree at a time, each dropped once its row is made. The projection moves from `horizon-items` down
to `horizon-world` and takes any `GameplayStateSource`, so its fields go through the same getters.

| R36S, 746 MB available, no snapshot, release build | Before | With 0035 |
|---|---|---|
| Result | killed in 4 of 4 runs, ~33 s in, on `world-rest-load` | loads and plays |
| Peak RSS (memory log, about once a second) | 704 MB and climbing at the kill (the kernel reports 759 MB anonymous) | 560 MB |
| Lowest MemAvailable | under 100 MB | 256 MB |

This may explain exit-137 runs of yours, or some of them: it's a launch-of-a-new-build peak, so a
later launch of the same binary (with the snapshot in place) passes. Worth checking
`dmesg | grep -i -E "out of memory|killed process"` after the next one. The later launches never
had this peak, which is also why it's easy to miss: a binary run once on a roomier path has its
snapshot, and stays fine.

Tests: `level::data::gameplay_tests` (a row reads as the state's own JSON does; a file without a
`states` array fails, as before), `horizon-world`'s `level::` (94) and `horizon-items` (42). Your
`mod_content` tests with `--include-ignored` (49, against the prepared resources): those that
touch this path are ignored by default, among them
`shared_behaviour_document_matches_independent_loaders` and
`independent_reloads_read_changed_behaviour_json_and_report_invalid_files`.

## On an R36S at 1 GB, with 0029–0034

Our 2 GB R36S (dArkOS on Debian 13, the Mali-G31 blob, your westonpack `crusty_x11egl` path) held
to a stock 1 GB R36S: front end stopped, zram off, a locked ballast leaving 746 MB available, the
game shown a 1 GB MemTotal. aarch64 builds of the patch series, the ASTC bakes in `cache/`.
Scripts: `tools/device/` (README there).

| Settings | World | 4-minute result |
|---|---|---|
| your render distance 2, simulation distance 5 | saved | steady: RSS 563–582 MB (GPU ~151 MB of it), 228–245 MB still available |
| render distance 6, simulation distance 12 | new | killed for memory after ~90 s (RSS 705 MB, GPU ~150 MB of it, growing) |
| render distance 6, simulation distance 5 | saved | killed after ~85 s |

So at your settings the series leaves a 1 GB R36S about 235 MB in hand while standing in a
world. Render distance 6 doesn't fit at all: the game's heap reaches ~600 MB and the GPU ~150–240
MB while chunks generate and mesh. The terrain mesh arena grows in steps (24 → 36 → 54 → 81 MiB on
the way there), each step briefly holding the old buffer and the new one.

Also seen on the device:

- The game panics on a clean stop (SIGTERM) under westonpack, in `khronos-egl`
  (`lib.rs:1184`, `Option::unwrap()` on `None`: an EGL function crusty doesn't give). It only
  writes a crash report at exit.
- Load times on this unit (exFAT card): resources ~27–32 s, world renderer ready ~32–40 s.

## Frame rate: first look on the R36S (nothing patched yet)

Same setup at render distance 2, simulation distance 5, **every other option at its default**
(not your `options.txt`, which I don't have), standing still in a saved world, `fast` profile
build (no LTO, so CPU figures are pessimistic):

```
Horizon: profile 9.9 fps, frame p50 99.7 p95 110.9 max 210.3 ms, cpu 25.2 ms, phases update 0.1 tick 5.8 prepare 1.2 render 18.1
```

- **The GPU is the larger cost here.** The Mali's load reads 73–82% at 10 fps, so about 77 ms of
  GPU work a frame against 25 ms of CPU. The default fancy leaves and clouds are the first suspects.
  Your 22–53 fps says your settings cost far less; I'll measure with your options when I have them.
- **The unit was fully thermally throttled.** `soc-thermal` has passive trips at 70 and 85 °C. In
  play it read 85–87 °C with both cooling devices at their maximum state: CPU limited to 1008 MHz
  of 1512, GPU to 400 MHz of 600. It was on its charger after hours of tests, so this may be
  worse than yours, but any long session heads there. Worth logging with your profile lines:
  `cat /sys/class/thermal/thermal_zone0/temp /sys/class/thermal/cooling_device*/cur_state`.
  Frame numbers from different thermal states can't be compared.
- **A background thread burns a core after every world open.** `Ring search`
  (`vanillagen.rs::search_rings_behind`) computes the stronghold rings at idle priority. It
  doesn't take CPU from the game's threads directly, but on this chip heat is the budget the clocks
  depend on. It was still running two minutes in. How long it takes is the next thing to measure.

### What the Mali counters say (Streamline, recorded on the card)

Release build, low settings (`cutoutLeaves:false renderClouds:"false" ao:false particles:2
entityShadows:false biomeBlendRadius:0 vignette:false chunkSectionFadeInTime:0.0`), render
distance 2, simulation distance 5, standing in a saved world, thermally limited to about
420 MHz GPU and 1050 MHz CPU. Low settings gain little by themselves: 12.8 fps against 11.1.

| Per frame at ~13.7 fps | |
|---|---:|
| GPU active | 21.0 M cycles, about 50 ms |
| Fragment queue | 15.9 M cycles, about 38 ms |
| Non-fragment queue (vertex) | 5.2 M cycles, about 12 ms |
| CPU | about 20 ms; the two add up to the frame, they don't overlap |
| Tiles | 5,268: 4.4 screens' worth (a screen is 1,200) |
| Quads shaded | 508 K, 6.6× the screen |
| Early ZS updated quads | **0** |

**`discard` in `solid_color` (terrain.wgsl) is the biggest single GPU cost found so far.** Every
solid face, stone included, runs a shader that can discard, which on Mali stops early depth updates
and weakens Forward Pixel Kill. A throwaway build with both discards removed (plants look wrong;
**not a patch**, fast profile) in the same scene:

| | With `discard` | Removed |
|---|---:|---:|
| Frame rate | 12–13 fps | **22–25 fps** |
| Median frame | 75–83 ms | **40–46 ms** |
| Fragment cycles per frame | 15.9 M | about 5.7 M |
| FPK occluding quads per second | 1.8 M | 5.5 M |

The frame is then about 24 ms GPU (fragment 13.5, vertex 11) plus 17 ms CPU. 60 fps would still
need about 2.5× more, on a thermally limited chip. Next levers, unmeasured: the vertex queue, the
passes that reload the frame (hand, HUD), the per-pixel terrain sampling, the CPU's `render` phase.

The patch this calls for: opaque faces (every texel of the layer's sprite and animation frames at
full alpha, or the opaque-leaves flag) drawn first with a no-discard pipeline, the rest with today's.

## 0036: faces over opaque texels drawn without discard

From here on the runs use a **frozen world**, because the plain save drifted: over a few runs mobs
piled up, the tick went from 4 to 20 ms and the frame rate from 11 to 7 fps with nothing changed.
`tools/device/freeze_world.py` copies the save with the clock at noon, clear weather, no random
ticks, no spawning and no entities, and `r36_session.sh` puts it back before each run. Same spot,
same view, same picture every time.

The mesher already marks quads over opaque texels (`OPAQUE_LAYER`). `Sides` now sorts those first
in each section, with side buckets of their own, and the world pass draws them with `fs_solid_opaque`,
which is `solid_color` without its discards. Occlusion culling, flat draws, a shader pack's terrain
and distant tiles keep one list as before.

| R36S, frozen world, low settings, render distance 2, GPU about 400 MHz at 76–80 °C | Before | 0036 |
|---|---:|---:|
| Frame rate | 13.5–14.6 fps | **15.8–17.3 fps** |
| `Early ZS updated quads` per frame | 0 | about 53,000 |
| Fragment cycles per frame | 17.3 M | about 11.8 M |
| All GPU cycles per frame | 22.9 M | about 17 M |
| Screenshot on the device | | **0 of 307,200 pixels differ** |

The "before" is a release build and the "after" a `fast` one, so the CPU half of this is if
anything pessimistic ([release against release](#the-release-build-in-one-sitting) comes after 0037).
Less than the throwaway build above gained, because plants and everything
else outside the opaque part still go through the discarding shader.

Tests: `mesher::` 31 pass, `gfx::` 161 pass with the one `panorama` failure the tree as received
also has, `shaders_translate` 2 pass.

## 0037: a plain terrain path

`malioc` for Mali-G31 showed where the terrain shaders' cost is. A vertex is bound by writing its
outputs (32 load/store cycles against 4 for its position), and the pixel shader uses 64 registers,
which halves the threads the core can run at once.

When a frame has no fade, no HDR alpha, no emissive layers, no dynamic lights, no RGSS and no distant
tiles, `vs_terrain_plain` and `fs_solid_plain_opaque` / `fs_solid_plain` draw the same picture from
much less. A vertex writes 9 values instead of 17 and looks the animated layer up itself. Fog goes
through as unclamped fractions, which interpolate as the distances would, so a pixel neither divides
nor branches for it. Any of those features in play falls back to the existing pipelines.

| R36S, frozen world, as above | 0036 | 0037 |
|---|---:|---:|
| Frame rate | 15.8–17.3 fps | **21.5–23.3 fps** |
| Fragment cycles per frame | about 11.8 M | 7.5 M |
| Vertex cycles per frame | about 5.2 M | 3.5 M |
| All GPU cycles per frame | about 17 M | 11 M |
| GPU memory in the world | 105 MB | **126 MB** |
| Screenshot against the tree before 0036 | 0 pixels differ | 1 of 307,200 differs |

| `malioc`, Mali-G31 | Before | Plain |
|---|---:|---:|
| Vertex, varying variant | 32 load/store cycles | 14 |
| Fragment, longest path | 51 arithmetic cycles | 5.25 |
| Registers | 64, 50% occupancy | 25, 100% occupancy |

**The 21 MB of GPU memory isn't explained.** It isn't the shaders' size: with the side buckets off (47
draws instead of 78) the same shaders stay at 105 MB, so it looks like a driver pool stepping up. RSS
goes from about 540 to 560 MB. It does no harm at render distance 2: held to 1 GB by the ballast (746 MB
available) for 4 minutes in the frozen world, peak RSS 566 MB, lowest available 252 MB, no OOM.

Tests on the final tree: `shaders_translate` 2 pass, `gfx::` 161 pass with the one `panorama` failure the
tree as received also has (with `tools/test_render.sh`, which sets aside the two `animate.rs` tests that
don't compile as received).

### The release build, in one sitting

The tables above compare `fast` builds run at different times. This is what you would ship: release
builds of 0035 and 0037 from the exported patch files, run one after the other on the same frozen world,
so both had the same heat. The temperature was 82 °C, the CPU about 1,020 MHz and the GPU about 420 MHz.

| R36S, frozen world, low settings, render distance 2, release | 0029–0035 | 0029–0037 |
|---|---:|---:|
| Frame rate, last three windows | 13.2–14.4, mean **13.8 fps** | 20.0–23.0, mean **21.9 fps** |
| Frame time, median | 68–74 ms | 44–51 ms |
| Frame time, 95th percentile | 77–91 ms | 49–63 ms |
| CPU per frame | 17–22 ms | 17–25 ms |
| All GPU cycles per frame | 22.9 M | **10.8 M** |
| Fragment cycles per frame | 17.3 M | 7.3 M |
| Vertex cycles per frame | 5.6 M | 3.4 M |
| `Early ZS updated quads` per frame | 0 | about 60,000 |
| GPU memory in the world | 105 MB | 126 MB |
| RSS | 534 MB | 559 MB |
| Screenshot against the 0035 picture | | 1 of 307,200 pixels differs |

The GPU's work halved, and the frame rate rose by 59% because the CPU's share didn't change.
10.8 M cycles at 420 MHz is 26 ms, and with 17.5 ms of CPU that gives the 44 ms measured, so the two
really do add up. The first 0037 windows ran slower (19.5 and 20.0 fps, CPU 22–25 ms) than the later
ones (23.0 and 22.7, CPU 17 ms). I don't know why; the baseline shows a smaller swing of the same kind.

## Where a frame goes now, and what didn't help

A frame is now about 45 ms: about 17 ms CPU plus about 27 ms GPU (11 M cycles at about 400 MHz). **They
barely overlap.** gdb and strace on the main thread show it blocked once a frame in the swap
(`wgpu_hal Surface::present` → libmali → SDL2's KMSDRM swap), then `DRM_IOCTL_MODE_PAGE_FLIP`: it waits
for the GPU to finish before it flips. So 60 fps on GL means CPU + GPU under 16.7 ms.

Everything below was measured on the same frozen world and **dropped**. The diffs are in
[spikes/](spikes/).

| Tried | Result | Why, as far as I can tell |
|---|---|---|
| Swap at the next frame's present, so CPU and GPU overlap | **9 fps**, 237 MB GPU memory | The wait moved into the driver with the GPU mostly idle. Probably stalls and copies over resources still in use by the frame in flight; not proven. |
| Consecutive passes on one target kept as one GL frame | 21.3 against 22.0 fps | Fired (tiles 4,427 → 2,205) and saved 2 ms of CPU, but fragment cycles didn't fall: the depth clear between world and hand costs a full-screen quad. And the GPU now starts later, which loses the little overlap there was. My guess that reloads between passes were a large cost was wrong. |
| `Ring search` off | 21.6 against 21.4 fps | Corrects my guess above that it mattered through heat. |
| `HORIZON_FLAT_DRAWS=1` | 21.8 against 21.4 fps | Counters unchanged, so it may not even be active on this driver. A frame has only 78 draws. |
| Side buckets off | 20.8 against 22.5 fps | Draws 78 → 47, but more vertices. |
| Depth pre-pass of the opaque faces | 19.9 against 21.5 fps | Fragment cycles didn't fall (7.9 M against 7.4 M), texture requests barely moved (502 K against 516 K), vertex cycles rose 3.5 → 4.6 M. So terrain hidden behind terrain isn't where the fragment time goes, despite what the raw quad counts suggest. |

Weights, by switching each off: hand about 2 ms of CPU and little GPU, HUD about 1.5 ms of CPU and
little GPU. Each extra pass is expensive on the CPU. 52% of the main thread's CPU time is inside the
Mali driver and 15% in libc; the game's own code is flat, with nothing above 2%.

### What's left for the frame rate

**I don't expect 60 fps on GL.** It needs CPU plus GPU under 16.7 ms, and the CPU alone is 17 ms, most of
it inside the driver. About 30 fps looks like the ceiling, and that is a judgment, not a measurement. It
would take 11 ms off a 44 ms frame.

Still open, neither tried:

- **Where the 7.3 M fragment cycles go.** Not terrain hidden behind terrain (the pre-pass above) and not
  reloads between passes (the pass merge above). It comes to about 1,660 cycles per tile.
- **Tick, prepare and encode on another thread**, so that they overlap the GPU. GL calls have to
  stay on one thread and the driver is most of the CPU, so only the rest can overlap. My estimate is about +15%,
  unmeasured, and my estimates in this area have been wrong before (see above). It's also a much larger
  change than anything in this series.

## Where the rest of the memory is, in a world

What was still allocated after 4 minutes in a creative world at render distance 12, with 0031
(heaptrack, system-allocator build, the R36S's MemTotal). On PC, llvmpipe keeps GPU textures and
buffers in the process. On the R36S those live in the Mali's share of the same 895 MiB, but
outside the process's RSS.

**GPU side (210 MiB on llvmpipe):**

| Item | PC | On the R36S |
|---|---:|---|
| GUI/HUD atlas, 4096×5120 RGBA | 80 MiB | the same 80 MiB, the biggest single item in the game; 55 MiB with 0032 + 0033 and the bake |
| Block sprite arrays | 27 MiB (RGBA: llvmpipe has no ASTC) | ~6.7 MiB from the ASTC bake |
| llvmpipe's own device state | 29 MiB | the Mali driver's own, not comparable |
| Everything else (HUD buffers, panorama, depth, entity outline, sections) | ~74 MiB | depends on the driver |

**The game's own heap (264 MiB), biggest first:**

| Item | Live | Notes |
|---|---:|---|
| Block-state table (`BlockRegistry`) | 21 MiB | each of ~29,700 states holds its own name `String` and a `BTreeMap<String, String>` of properties. Interning them would save most of it, but `BlockState` is a core type used everywhere. |
| Baked block models (`StateModels`) | 16.5 MiB | needed data |
| Animated sprites' frames, upscaled 16 → 64 px | 9.3 MiB | 54 animated layers, every frame stored at the array size, 16× its native pixels. Native frames scaled at upload would be ~0.6 MiB. This is in `Sprites::lay_out` / `gfx/animate.rs`, the code under suspicion for the water/kelp bug (spec §9.2), so not touched. |
| Structure templates' NBT | 8–16 MiB | the block entities of every structure piece generated so far, kept like vanilla's `StructureTemplateManager`. Grows with exploration. |
| Worldgen scratch buffers | ~10 MiB | per worker thread |
| Block shapes, loot tables, advancements, item models, recipes… | 2–8 MiB each | needed data |

## Other things found on the way (not patched)

1. **The §9.1 hang is the RGBA fallback, not memory** (corrected: I first guessed OOM). On an
   R36S with 2 GB and no memory pressure, every launch without the ASTC bakes (3 of 3, with 0031,
   0033 and 0034 alike) stops while the shared atlas uploads: the log's last line is `no ASTC bake
   … uploading RGBA` (0033+) or `shared atlas 4096x5120, 80 MiB` (0031), every thread sleeps, and
   gdb shows the main thread inside the Mali driver, `osup_sync_object_wait` from libGLESv2,
   waiting for a sync object that never signals. No GPU fault in `dmesg`. With the bakes in place
   the same builds load every time. Your hung run also had the RGBA fallback. Not fixed yet.
2. **The GUI atlas (4096×5120, 80 MiB RGBA) is half empty by design.** Only 31.5 MiB is real
   textures. 30.8 MiB is runtime slots reserved at fixed desktop counts (`hud.rs`: 256 map slots =
   16 MiB, the on-screen-keyboard text area 2048×1024 = 8 MiB, 64 pack icons = 4 MiB, 128 skins =
   2 MiB, 32 world icons, 32 trims). Another 8 MiB is the 512 glyph rows and ~9 MiB is packing
   gaps (`tools/atlas_usage.py` reproduces the layout from the jar). 0032 + 0033 compress the
   static part; the dynamic part still holds those slots at those counts (see above).
3. **The decoded-sound cache stores `f32`; vanilla's OpenAL buffers are 16-bit.** 0030 bounds
   the cache by RAM. Storing the cached samples as `i16` and converting in the mixer would halve
   whatever the budget holds again. That is a larger change to the mixer, so it isn't patched.
4. **The received tree doesn't pass its own checks.** `cargo test -p horizon-client-render` doesn't
   compile: `gfx/animate.rs:867` and `:934` call `SpriteAnimator::new` with its old arguments
   (`&textures, split` instead of `&texture, &layer_index`). And `cargo fmt` rewrites five files the
   patches touched (`astc.rs`, `gfx.rs`, `gfx/arena.rs`, `hud.rs`, `sprites.rs`), so
   `tools/pr_ready.py` will fail its fmt step. `cargo clippy -p horizon-client-render --lib -- -D
   warnings` fails on 9 lints, which point at `astc.rs:185, 282, 286, 431, 432`, `gfx.rs:2364, 2376`,
   `gfx/arena.rs:280, 425`, `gfx/panorama.rs:31` (same-type casts, operator precedence, `expect`
   on `None`, `chunks_exact` with a constant size). 0029–0034 leave these alone, except that 0032
   removes the dead ASTC branch in `gfx.rs` where the `gfx.rs` lints were, which leaves 8.
5. **No in-game leak at this scale.** A creative world at render distance 6, standing still,
   settles at ~1040–1070 MiB (PC, llvmpipe) after loading and stays there for the next 4+ minutes.
   The title screen alone does creep (~10–20 MiB a minute, sound on or off), but that isn't a leak.
   A build with the system allocator, under heaptrack, keeps a flat live heap at the title
   (380 MiB from 60 s to 180 s), so the creep is mimalloc holding freed pages.
6. **Transparent huge pages are not a lever on the R36S.** The dedicated server turns mimalloc's
   THP off in low-memory mode ("mostly empty huge pages cost 60 MB idle"). The client doesn't, but
   dArkOS's 4.4.189 kernel on an R36S has no THP at all (no `/sys/kernel/mm/transparent_hugepage`,
   checked read-only on another R36S), so there is nothing to turn off.
7. **On the R36S the logged RSS does include the GPU memory** (corrected: I first said it didn't).
   Measured on dArkOS's Mali driver: VmRSS is the sum of the process's mappings in `smaps` plus
   about the Mali driver's own count (`/sys/kernel/debug/mali0/gpu_memory`, as root): 667 MB =
   523 + ~140 in one world. The GPU pages don't appear in `smaps`, only in the total.
