# TerraForge 32 3D

TerraForge 32 3D is an original PRG32 cartridge game inspired by broad voxel/sandbox ideas: explore a generated block cavern in first person, dig blocks, place collected material, dodge crystalline drones, and recover four relics. It does **not** use names, sprites, sounds, maps, music, block textures, recipes, UI, story, or protected expression from Minecraft or any other commercial game.

The project follows the PRG32 cartridge convention: the C source exports `terraforge32_c_init`, `terraforge32_c_update`, and `terraforge32_c_draw`, matching the `--entry-prefix terraforge32_c` build option used by PRG32 cartridges.

## Gameplay

- D-pad left/right: rotate the first-person camera.
- D-pad up/down: move forward/backward.
- Hold B while moving: run.
- Tap B while not moving forward: cycle the selected inventory block.
- A: dig the block under the crosshair, if it is close enough.
- Down + A: place the selected inventory block in front of you.
- START: restart/reseed the cavern.

Goal: collect four crystal relics before health reaches zero. Water floors slowly heal; lava floors and crystalline drones damage the player. The minimap in the lower-left corner is included as both gameplay help and a didactic visualization of the ray-cast world.

## 3D implementation

The game uses a compact 2.5D ray-caster designed for PRG32-scale hardware:

- 80 rays per frame, one four-pixel column each, fill the 320-pixel display width.
- A 32-direction fixed-point sine/cosine table avoids floating point.
- Each ray advances through the block grid in Q8 fixed-point steps.
- Wall height is projected with integer division and distance correction to reduce fisheye distortion.
- Material colors are depth-shaded using the separated asset palette.
- Drones and particle bursts are projected as simple first-person billboards.
- The minimap is drawn from the same block grid, making the algorithm easy to teach.

## Repository layout

```text
terraforge32/
├── c/game.c                         # game logic and renderer only
├── assets/
│   ├── png/                         # editable PNG masters
│   ├── wav/                         # editable synthetic WAV masters
│   ├── include/                     # generated C headers consumed by game.c
│   ├── generate_assets.py           # asset-regeneration entry point
│   └── manifest.json                # licensing and provenance
├── metadata/                        # CartridgeStore metadata blocks
├── scripts/
│   ├── build.sh                     # portable cartridge build
│   └── pack-store-bundle.sh         # Store bundle packer
├── docs/
│   ├── didactic_notes.md
│   ├── copyright_cleanroom.md
│   └── build_and_release.md
└── tools/build_cartridge.sh
```

## Build

From this repository, with a PRG32 checkout that supports portable ABI-table
cartridges cloned next to it:

```bash
export PRG32_ARCHITECTURE=esp32c6
scripts/build.sh
```

The script writes `dist/terraforge32-esp32c6.prg32`. The default build is
portable and is not tied to one firmware ELF. Build the QEMU variant with:

```bash
export PRG32_ARCHITECTURE=qemu
scripts/build.sh
```

`tools/build_cartridge.sh` remains as a compatibility wrapper around
`scripts/build.sh`. For older firmware/tooling without portable cartridge
support, set `PRG32_PORTABLE=0` and pass a firmware ELF path.

Upload and publish use the same flow as DeviceDemo's portable branch; see
`docs/build_and_release.md`.

## Capabilities exercised

TerraForge 32 3D deliberately stresses PRG32 subsystems: full-screen rectangle rasterization, fixed-point math, procedural block maps, ray casting, billboard projection, sprite/tile asset use, input edge detection, inventory state, particles, audio beeps/note sequences, stereo soundtrack scheduling, and compact didactic code organization.

## Teaching tutorial

For a step-by-step first-year laboratory sequence covering design, implementation,
debugging, deployment, and Cartridge Store publication, see
`docs/teaching_tutorial.md`.

## License

Code is MIT. Original art/audio assets are CC0-1.0. See `LICENSE` and `assets/manifest.json`.


## Stereo soundtrack

`assets/wav/terraforge32_stereo_loop.wav` is an original 36-second stereo loop master, generated synthetically for this project and released as CC0-1.0 with the rest of the assets. The C cartridge also includes `assets/include/terraforge_music.h`, a compact left/right score used by the minimal PRG32 note/beep API so the soundtrack remains audible on mono setups.

For full stereo on hardware, use PRG32 Audio Plus and convert/package the WAV or a derived tracker representation with the PRG32 audio tools (`prg32_audio_convert.py`, `wav2prg32sample.py`, and `prg32audio_pack.py`) into an AUDIO block. PRG32 documents mandatory mono I2S and optional stereo Audio Plus using two MAX98357A amplifier breakouts.
