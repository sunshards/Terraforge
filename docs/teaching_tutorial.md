# Teaching tutorial: building TerraForge 32

This tutorial guides first-year computer science and computer engineering students through the construction of TerraForge 32, an original PRG32 educational cartridge. It is written as a sequence of one-hour laboratories. Each laboratory has a concrete product, a short technical focus, and a checkpoint that can be evaluated in class.

The course intentionally treats game development as a systems exercise: students move from design to fixed-point C, from assets to generated headers, from local checks to hardware deployment, and finally from a working cartridge to a publication-ready Cartridge Store bundle.

## Prerequisites

Students should know basic C syntax, arrays, functions, integer arithmetic, and command-line navigation. They do not need prior graphics or embedded experience.

Each workstation should have:

- this repository;
- a PRG32 checkout, preferably next to this repository;
- a C compiler usable by `tools/static_check.sh`;
- Python 3 for the PRG32 tooling and asset scripts;
- either QEMU support from PRG32 or access to an ESP32-C6 PRG32 board.

Use the repository scripts before custom commands:

```bash
./tools/static_check.sh
./tools/build_cartridge.sh
```

When building through the portable script, set the architecture explicitly:

```bash
export PRG32_ARCHITECTURE=qemu
scripts/build.sh
```

or:

```bash
export PRG32_ARCHITECTURE=esp32c6
scripts/build.sh
```

## The code-deploy-debug cycle

Every laboratory uses the same pragmatic cycle. Students should write it at the top of their lab notebook and repeat it until it becomes automatic.

1. **State the intended behavior.** Write one sentence before editing code: for example, "A ray stops at the first solid block and reports the previous empty cell."
2. **Make the smallest coherent change.** Edit `c/game.c`, the editable assets in `assets/png` or `assets/wav`, or the store metadata in `metadata/`, depending on the laboratory.
3. **Run the static check.**

   ```bash
   ./tools/static_check.sh
   ```

   A syntax error is a fast result, not a failure. Fix it before thinking about gameplay.
4. **Build the cartridge.**

   ```bash
   export PRG32_ARCHITECTURE=qemu
   scripts/build.sh
   ```

   Use `esp32c6` when preparing a hardware test.
5. **Deploy to the target.** For QEMU staging, upload the QEMU cartridge to the PRG32 flash image. For hardware, upload the ESP32-C6 cartridge to the board.
6. **Observe one thing.** Test only the behavior named in step 1. Do not debug five changes at once.
7. **Record evidence.** Keep a short note: command run, result observed, bug found, next hypothesis.
8. **Commit at laboratory boundaries.** A commit should represent a working teaching milestone, not every keystroke.

This cycle is deliberately conservative. It protects students from the most common early embedded mistake: changing rendering, input, assets, and packaging simultaneously, then not knowing which subsystem broke.

## Laboratory 1: concept, originality, and constraints

**Estimated duration:** 1 hour

**Goal:** Define the game as an original educational cartridge and identify the hardware constraints that shape the implementation.

**Activities:**

1. Read `README.md`, `docs/copyright_cleanroom.md`, and `docs/didactic_notes.md`.
2. Identify the cartridge's original premise: first-person exploration of a generated cavern, digging and placing material, drones, floor hazards, and four crystal relics.
3. List what may be reused from broad game-design tradition: block worlds, digging, inventory-like selection, first-person navigation, and procedural caves.
4. List what must not be copied from commercial games: names, textures, sounds, recipes, maps, mobs, story, user-interface layouts, and other protected expression.
5. Inspect the repository layout and explain why runtime logic lives in `c/game.c`, editable assets live in `assets/png` and `assets/wav`, generated C data lives in `assets/include`, and store metadata lives in `metadata/`.

**Checkpoint:** Submit a one-page design brief containing the game goal, controls, originality statement, and three technical constraints imposed by PRG32.

**Instructor notes:** This lab sets the ethical and engineering tone. Treat copyright cleanliness as a professional design requirement, not as an afterthought.

## Laboratory 2: the minimal PRG32 cartridge loop

**Estimated duration:** 1 hour

**Goal:** Understand the cartridge entry points and the separation between initialization, update, and draw.

**Code focus:** `terraforge32_c_init`, `terraforge32_c_update`, and `terraforge32_c_draw` in `c/game.c`.

**Activities:**

1. Find the three exported cartridge functions.
2. Trace which state is initialized once, which state changes every frame, and which state is only rendered.
3. Temporarily modify a harmless HUD color or text position, then run:

   ```bash
   ./tools/static_check.sh
   ```

4. Build a QEMU cartridge:

   ```bash
   export PRG32_ARCHITECTURE=qemu
   scripts/build.sh
   ```

5. Revert the temporary visual experiment or keep it only if the class agrees it improves readability and remains original.

**Checkpoint:** Draw a three-box diagram showing initialization, update, and draw, with two examples of state handled by each box.

**Debug habit:** If the cartridge starts but shows an unexpected screen, ask first whether the state was initialized, updated, or drawn incorrectly. This simple classification usually halves the search space.

## Laboratory 3: world representation and deterministic generation

**Estimated duration:** 1 hour

**Goal:** Build a mental model of the block grid and floor-effect grid.

**Code focus:** `world`, `floor_tile`, `hashed_wall`, `carve_rect`, and `generate_world`.

**Activities:**

1. Inspect the `WORLD_W x WORLD_H` arrays.
2. Explain why solid blocks and floor effects are stored separately.
3. Change one carved room or corridor by a small amount.
4. Run the static check and build.
5. Observe the change on the minimap and in first-person view.
6. Restore the original shape if the change damages reachability.

**Checkpoint:** Students must answer: "Why is deterministic generation useful in a classroom?" Expected themes include reproducibility, debugging, assessment, and fair comparison of algorithmic changes.

**Extension:** Add one new floor-effect zone using an existing tile identifier. Keep it small and reachable.

## Laboratory 4: fixed-point coordinates and movement

**Estimated duration:** 1 hour

**Goal:** Use integer arithmetic to represent smooth movement without floating point.

**Code focus:** Q8 positions, `cos32`, `sin32`, `passable_q8`, `try_move`, and player input in `terraforge32_c_update`.

**Activities:**

1. Convert three example positions between tile coordinates and Q8 coordinates: `5`, `5 << 8`, and `(5 << 8) + 128`.
2. Inspect the 32-direction trigonometric lookup tables.
3. Change walking or running speed by a small integer amount.
4. Run the code-deploy-debug cycle.
5. Observe collision near corners and explain why `passable_q8` samples four points around the player.

**Checkpoint:** In a short note, compare fixed-point arithmetic with floating-point arithmetic for a small RISC-V teaching target.

**Common bug:** If movement appears to skip through walls, the step size may be too large relative to the collision radius and map cell size.

## Laboratory 5: ray casting and first-person projection

**Estimated duration:** 1 hour

**Goal:** Implement the central visual idea: one ray per screen column group.

**Code focus:** `cast_ray`, `draw_3d`, and `draw_ray_column`.

**Activities:**

1. Trace a ray from the player's Q8 position until it enters a solid cell.
2. Identify the fields returned by `rayhit_t`: distance, hit cell, previous cell, tile, and side.
3. Change `RAYS` or `COL_W` only as an experiment, keeping `RAYS * COL_W == 320` when possible.
4. Build and compare the result.
5. Restore the default unless the class intentionally discusses the performance and quality trade-off.

**Checkpoint:** Students must explain why `80` rays of width `4` cover the `320` pixel display, and why integer projection is suitable for this cartridge.

**Debug habit:** When a wall is drawn incorrectly, print or inspect the ray result conceptually before changing the renderer. The renderer can only draw the data that `cast_ray` reports.

## Laboratory 6: interaction, inventory, and game rules

**Estimated duration:** 1 hour

**Goal:** Connect the ray caster to gameplay.

**Code focus:** `break_target`, `place_target`, `cycle_selected`, inventory counts, relics, health, and input edge detection.

**Activities:**

1. Explain how the center ray becomes a targeting tool.
2. Trace digging: ray hit, distance limit, inventory increment, relic check, cell removal, particles, and sound.
3. Trace placing: selected block, inventory count, previous empty cell, and map update.
4. Modify one distance limit or inventory rule as a controlled experiment.
5. Test digging, placing, selection, restart, and relic collection.

**Checkpoint:** Students produce a small state-transition table for one action: idle to dig, idle to place, or playing to victory.

**Common bug:** If placing creates blocks inside the player, check whether the previous empty cell and the player collision radius are both considered.

## Laboratory 7: enemies, particles, hazards, and feedback

**Estimated duration:** 1 hour

**Goal:** Add life to the world while keeping the simulation simple enough to inspect.

**Code focus:** `bot_t`, `particle_t`, `reset_bots`, `update_bots`, `update_particles`, `floor_effects`, `draw_bot_3d`, and `draw_particles_3d`.

**Activities:**

1. Identify which state belongs to drones and which belongs to particles.
2. Explain why arrays with fixed maximum sizes are preferable to heap allocation here.
3. Adjust one drone starting position or particle lifetime.
4. Run the static check, build, deploy, and test.
5. Observe water healing, lava damage, and drone damage.

**Checkpoint:** Students submit a table of the feedback channels used by the game: color, motion, particles, sound, health, and HUD.

**Instructor notes:** This is a good point to discuss simulation budgets. The design uses a few expressive objects rather than many expensive ones.

## Laboratory 8: original assets and audio pipeline

**Estimated duration:** 1 hour

**Goal:** Understand the difference between editable source assets and generated runtime data.

**Asset focus:** `assets/png`, `assets/wav`, `assets/generate_assets.py`, `assets/manifest.json`, and `assets/include`.

**Activities:**

1. Inspect the editable PNG and WAV masters.
2. Inspect `assets/manifest.json` and identify provenance and licensing information.
3. Make a small original tile, icon, splash, or sound adjustment.
4. Regenerate assets if the edited file requires it:

   ```bash
   python3 assets/generate_assets.py
   ```

5. Run `./tools/static_check.sh` and rebuild.
6. Confirm that generated headers changed only as expected.

**Checkpoint:** Students must write two sentences explaining why generated C-facing headers should not be hand-edited.

**Clean-room rule:** An asset is acceptable only when the student can explain how it was made without reference to a commercial game's protected expression.

## Laboratory 9: heads-up display, minimap, and readability

**Estimated duration:** 1 hour

**Goal:** Treat the interface as an instrument panel for both players and learners.

**Code focus:** `draw_minimap`, `draw_hud`, `draw_crosshair`, `draw_uint`, and end-state rendering.

**Activities:**

1. Identify how the minimap reuses the same world grid as the 3D renderer.
2. Explain why the crosshair is both a gameplay affordance and a debugging aid.
3. Improve one HUD detail: position, color contrast, or numeric clarity.
4. Test at least three states: normal exploration, low health, and relic collection.
5. Verify that text remains readable and does not obscure the first-person task.

**Checkpoint:** Students present one interface decision and justify it with respect to limited screen space.

**Instructor notes:** Encourage restraint. A small embedded display rewards clear hierarchy more than decoration.

## Laboratory 10: integration testing on QEMU and hardware

**Estimated duration:** 1 hour

**Goal:** Practice disciplined release testing before publication.

**Activities:**

1. Run the static check:

   ```bash
   ./tools/static_check.sh
   ```

2. Build the QEMU cartridge:

   ```bash
   export PRG32_ARCHITECTURE=qemu
   scripts/build.sh
   ```

3. Stage it in QEMU, adapting paths to the local PRG32 checkout:

   ```bash
   python3 "$PRG32_REPO/tools/prg32_game.py" upload-qemu \
     dist/terraforge32-qemu.prg32 \
     --flash "$PRG32_REPO/build-qemu/flash_image.bin" \
     --partitions "$PRG32_REPO/partitions_prg32.csv"
   ```

4. Build the ESP32-C6 cartridge:

   ```bash
   export PRG32_ARCHITECTURE=esp32c6
   scripts/build.sh
   ```

5. Upload to a board, adapting the URL to the classroom network:

   ```bash
   python3 "$PRG32_REPO/tools/prg32_game.py" upload \
     dist/terraforge32-esp32c6.prg32 \
     --url http://192.168.4.1
   ```

6. Execute the manual test checklist: rotate, move, run, dig, place, select, take damage, heal, collect relics, restart, and reach an end state.

**Checkpoint:** Students submit a test log with commands, target used, observations, and one fixed bug or justified non-bug.

**Debug habit:** Prefer a reproducible input sequence over a vague report. "Turn right twice, move forward six steps, press A" is useful; "digging feels broken" is not.

## Laboratory 11: metadata, store bundle, and publication

**Estimated duration:** 1 hour

**Goal:** Prepare a cartridge for distribution through the Cartridge Store.

**Publication focus:** `metadata/manifest.json`, `metadata/metadata.json`, `metadata/colophon.json`, `assets/icon.png`, `assets/screenshot.png`, and `scripts/pack-store-bundle.sh`.

**Activities:**

1. Inspect the store manifest and verify the cartridge id, title, version, summary, tags, assets, and architecture files.
2. Verify that the icon and screenshot are current and original.
3. Build both architecture variants:

   ```bash
   export PRG32_ARCHITECTURE=esp32c6
   scripts/build.sh
   export PRG32_ARCHITECTURE=qemu
   scripts/build.sh
   ```

4. Pack the store bundle:

   ```bash
   scripts/pack-store-bundle.sh
   ```

5. Publish to the Cartridge Store when authorized:

   ```bash
   python3 "$PRG32_REPO/tools/prg32_game.py" publish-bundle \
     dist/terraforge32-store-bundle.zip \
     --store-url http://192.168.1.42:5080 \
     --token "$PRG32_STORE_TOKEN"
   ```

6. Record the published version, store URL, commit hash, and test evidence.

**Checkpoint:** Students produce a release sheet containing version, architectures, bundle path, clean-room confirmation, and publication command used.

**Professional rule:** Publication is not just "uploading the game." It is the act of making a reproducible, licensed, tested artifact available to other people.

## Laboratory 12: final review and oral defense

**Estimated duration:** 1 hour

**Goal:** Demonstrate understanding of the whole system.

**Activities:**

1. Present the game in two minutes.
2. Explain one algorithmic subsystem: world generation, fixed-point movement, ray casting, collision, audio scheduling, asset generation, or store packaging.
3. Demonstrate the code-deploy-debug cycle with a tiny safe change.
4. Answer a clean-room question: "How do you know this asset or mechanic is original enough for publication?"
5. Identify one improvement that would be technically realistic on PRG32.

**Checkpoint:** The final grade should reward understanding over feature count. A modest, well-explained cartridge is a better first-year outcome than an ambitious but opaque program.

## Suggested assessment rubric

| Area | Weight | Evidence |
| --- | ---: | --- |
| Original design and clean-room discipline | 15% | Design brief, asset provenance, no copied expression |
| C implementation understanding | 25% | Correct explanations of state, arrays, fixed-point math, and functions |
| Rendering and interaction | 20% | Ray casting, targeting, HUD, minimap, and input behavior |
| Debugging discipline | 15% | Repeated use of static check, build, deploy, observed evidence |
| Asset and audio pipeline | 10% | Editable assets, generated headers, manifest updates |
| Release engineering | 10% | Architecture builds, store bundle, metadata, publication readiness |
| Communication | 5% | Lab notes, oral defense, concise diagrams |

## Capstone variations

Students who finish early may attempt one extension, subject to the same clean-room and performance constraints:

- add a new original block type with a generated tile and manifest entry;
- add a new floor effect using `floor_tile`;
- tune ray count and column width, then measure visual quality against responsiveness;
- add a second drone behavior using the existing fixed-size array;
- create an Audio Plus release path while preserving mono fallback;
- improve the store screenshot and release notes.

Each extension must include a short explanation of the trade-off it introduces.

## Instructor closing notes

TerraForge 32 is useful because its systems are visible. A student can hold the whole cartridge in mind: a grid, a player, a ray, a few arrays, a small renderer, original assets, metadata, and a repeatable release path. That is the central lesson. Good engineering is not only making the game work; it is making the work inspectable, explainable, and publishable.
