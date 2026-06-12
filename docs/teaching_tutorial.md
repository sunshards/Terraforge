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

**Files to edit:** `README.md`, `docs/copyright_cleanroom.md`, and later `metadata/manifest.json`.

**Code or text to add:** In the first laboratory the "code" is the project's public contract. Add a short clean-room statement before any implementation work, so the ethical constraint is visible from the start.

```markdown
<!-- README.md -->
TerraForge 32 is an original PRG32 teaching cartridge. It may use broad
ideas such as block worlds, caves, digging, and first-person exploration, but
it must not copy names, textures, recipes, maps, sounds, story, UI layout, or
other protected expression from commercial games.
```

```json
{
  "id": "org.uniparthenope.prg32.terraforge32",
  "title": "TerraForge 32 3D",
  "summary": "Original PRG32 first-person block cavern cartridge.",
  "tags": ["game", "education", "raycaster", "riscv", "prg32"]
}
```

The metadata fragment teaches two habits: choose a stable reverse-DNS id, and describe the work in original words rather than by comparison with a commercial title.

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

**Files to edit:** `c/game.c`.

**Code to add:** Start with the three required PRG32 entry points. The functions are intentionally small at first; students should see the frame structure before adding world data or rendering detail.

```c
/* Called once when PRG32 loads or restarts the cartridge.
   Keep permanent setup here: assets, initial state, and counters. */
void terraforge32_c_init(void) {
    frame_no = 0;
    last_input = 0;
    game_state = 0;
}

/* Called once per frame before drawing.
   Read input and change game state here, not in the draw function. */
void terraforge32_c_update(void) {
    uint32_t input = prg32_input_read();
    uint32_t press = input & ~last_input; /* Buttons newly pressed this frame. */
    last_input = input;

    if (press & PRG32_BTN_START) {
        terraforge32_c_init(); /* START is a simple, testable restart path. */
    }

    frame_no++;
}

/* Called once per frame after update.
   Drawing should report state, not secretly change it. */
void terraforge32_c_draw(void) {
    prg32_gfx_clear(PRG32_COLOR_BLACK);
    prg32_gfx_text8(4, 5, "TERRAFORGE32", PRG32_COLOR_WHITE, PRG32_COLOR_BLACK);
}
```

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

**Files to edit:** `c/game.c`.

**Code to add:** Add the map arrays and generator before rendering. The important design decision is that vertical walls and floor effects are separate arrays.

```c
#define WORLD_W PRG32_PLAYFIELD_COLS
#define WORLD_H PRG32_PLAYFIELD_ROWS

#define TILE_AIR   0
#define TILE_DIRT  4
#define TILE_STONE 5
#define TILE_WATER 10
#define TILE_LAVA  11

/* world stores solid blocks.  floor_tile stores effects under the player.
   Separating them lets a cell be empty space with water or lava on the floor. */
static uint8_t world[WORLD_H][WORLD_W];
static uint8_t floor_tile[WORLD_H][WORLD_W];

static int hashed_wall(int x, int y) {
    /* Deterministic "randomness": the same x,y always gives the same answer.
       This makes classroom bugs reproducible on every machine. */
    uint32_t h = (uint32_t)(x * 7349u) ^ (uint32_t)(y * 9151u);
    h ^= h >> 7;
    h *= 1103515245u;
    return (int)(h & 15u) < 5;
}

static void carve_rect(int x0, int y0, int x1, int y1) {
    /* Carving by rectangle is deliberately simple: students can predict it. */
    for (int y = y0; y <= y1; y++) {
        for (int x = x0; x <= x1; x++) {
            if (x > 0 && y > 0 && x < WORLD_W - 1 && y < WORLD_H - 1) {
                world[y][x] = TILE_AIR;
            }
        }
    }
}

static void generate_world(void) {
    for (int y = 0; y < WORLD_H; y++) {
        for (int x = 0; x < WORLD_W; x++) {
            floor_tile[y][x] = TILE_DIRT;
            world[y][x] = hashed_wall(x, y) ? TILE_STONE : TILE_AIR;
        }
    }

    carve_rect(2, 2, 12, 8);      /* Starting hub. */
    carve_rect(10, 5, 27, 7);     /* Corridor that students can inspect. */
    for (int x = 15; x < 23; x++) floor_tile[15][x] = TILE_WATER;
    for (int x = 42; x < 50; x++) floor_tile[20][x] = TILE_LAVA;
}
```

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

**Files to edit:** `c/game.c`.

**Code to add:** Add Q8 player state, lookup-table motion, and collision. Q8 means that one tile is `256` integer units.

```c
#define ANGLES 32
#define PLAYER_R 42

static int player_x, player_y, player_angle;

/* Fixed-point trigonometry: values are scaled by 256.
   Angle 0 points east; increasing angles rotate around the compass. */
static const int16_t cos32[ANGLES] = {
    256,251,237,213,181,142,98,50,0,-50,-98,-142,-181,-213,-237,-251,
    -256,-251,-237,-213,-181,-142,-98,-50,0,50,98,142,181,213,237,251
};
static const int16_t sin32[ANGLES] = {
    0,50,98,142,181,213,237,251,256,251,237,213,181,142,98,50,
    0,-50,-98,-142,-181,-213,-237,-251,-256,-251,-237,-213,-181,-142,-98,-50
};

static int wrap_angle(int a) {
    while (a < 0) a += ANGLES;
    while (a >= ANGLES) a -= ANGLES;
    return a;
}

static int passable_q8(int x, int y) {
    /* Four samples approximate the player's body as a small square.
       This is inexpensive and prevents corner clipping in a tile map. */
    return !cell_solid((x - PLAYER_R) >> 8, (y - PLAYER_R) >> 8) &&
           !cell_solid((x + PLAYER_R) >> 8, (y - PLAYER_R) >> 8) &&
           !cell_solid((x - PLAYER_R) >> 8, (y + PLAYER_R) >> 8) &&
           !cell_solid((x + PLAYER_R) >> 8, (y + PLAYER_R) >> 8);
}

static void reset_player(void) {
    player_x = 5 << 8; /* Tile 5.0 in Q8. */
    player_y = 5 << 8;
    player_angle = 0;
}

static void try_move(int dx, int dy) {
    /* Move one axis at a time so sliding along walls remains possible. */
    int nx = player_x + dx;
    if (passable_q8(nx, player_y)) player_x = nx;

    int ny = player_y + dy;
    if (passable_q8(player_x, ny)) player_y = ny;
}
```

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

**Files to edit:** `c/game.c`.

**Code to add:** Add the hit record, the ray marcher, and the column renderer. The comments emphasize why the algorithm is inspectable on small hardware.

```c
#define RAYS 80
#define COL_W 4

typedef struct {
    int dist;              /* Approximate Q8-ish distance along the ray. */
    int tx, ty;            /* Solid tile hit by the ray. */
    int last_tx, last_ty;  /* Last empty tile before the hit; useful later. */
    uint8_t tile;
    uint8_t side;          /* Which grid boundary was crossed. */
} rayhit_t;

static rayhit_t cast_ray(int angle) {
    rayhit_t h = {4096, -1, -1, player_x >> 8, player_y >> 8, TILE_AIR, 0};
    angle = wrap_angle(angle);

    int x = player_x;
    int y = player_y;
    int last_tx = x >> 8;
    int last_ty = y >> 8;
    int dx = cos32[angle] >> 4;
    int dy = sin32[angle] >> 4;

    for (int dist = 16; dist < 4096; dist += 16) {
        x += dx;
        y += dy;
        int tx = x >> 8;
        int ty = y >> 8;

        if (tx != last_tx || ty != last_ty) {
            if (cell_solid(tx, ty)) {
                h.dist = dist;
                h.tx = tx;
                h.ty = ty;
                h.last_tx = last_tx;
                h.last_ty = last_ty;
                h.tile = world[ty][tx];
                h.side = (tx != last_tx);
                return h;
            }
            last_tx = tx;
            last_ty = ty;
        }
    }
    return h;
}

static void draw_ray_column(int col, rayhit_t h, int angle_offset) {
    /* Projection is intentionally integer-only: closer walls become taller. */
    int corr = h.dist;
    int wall_h = 46080 / (corr + 16);
    wall_h = clamp_i(wall_h, 6, 190);

    int y0 = 100 - wall_h / 2;
    int y1 = 100 + wall_h / 2;
    uint16_t color = shade_for(h.tile, h.side, corr);

    prg32_gfx_rect(col * COL_W, y0, COL_W, y1 - y0, color);
}

static void draw_3d(void) {
    draw_background();
    for (int r = 0; r < RAYS; r++) {
        int off = ((r - RAYS / 2) * 8) / RAYS;
        rayhit_t h = cast_ray(wrap_angle(player_angle + off));
        draw_ray_column(r, h, off);
    }
}
```

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

**Files to edit:** `c/game.c`.

**Code to add:** Reuse the ray caster for gameplay. This is an important teaching moment: a rendering query becomes an interaction query.

```c
static uint16_t inv[13];
static uint8_t selected;
static uint16_t health, relics;
static rayhit_t center_hit;

static void break_target(void) {
    rayhit_t h = cast_ray(player_angle);
    if (h.tile == TILE_AIR || h.dist > 1536) return;

    /* The block removed from the world becomes inventory.
       Crystals also advance the victory condition. */
    uint8_t t = h.tile;
    inv[t]++;
    if (t == TILE_CRYSTAL) relics++;

    world[h.ty][h.tx] = TILE_AIR;
    burst_cell(h.tx, h.ty, t);
    prg32_audio_beep(t == TILE_CRYSTAL ? 1100 : 220, 30);
}

static void place_target(void) {
    rayhit_t h = cast_ray(player_angle);
    int tx = h.last_tx;
    int ty = h.last_ty;

    if (selected == TILE_AIR || inv[selected] == 0) return;
    if (cell_solid(tx, ty)) return;
    if ((player_x >> 8) == tx && (player_y >> 8) == ty) return;

    world[ty][tx] = selected;
    inv[selected]--;
    burst_cell(tx, ty, selected);
}

/* In update: newly pressed A performs one action, not one action per frame. */
if (press & PRG32_BTN_A) {
    if (input & PRG32_BTN_DOWN) place_target();
    else break_target();
}
```

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

**Files to edit:** `c/game.c`.

**Code to add:** Add fixed-size arrays for transient effects and autonomous enemies. The fixed maximums make memory use visible and predictable.

```c
#define MAX_BOTS 6
#define MAX_PARTICLES 32

typedef struct { int x, y, vx, vy, life; uint16_t color; } particle_t;
typedef struct { int x, y, dir, alive, phase; } bot_t;

static particle_t particles[MAX_PARTICLES];
static bot_t bots[MAX_BOTS];

static void reset_bots(void) {
    int bx[MAX_BOTS] = {23, 37, 52, 17, 49, 31};
    int by[MAX_BOTS] = {14, 10, 20, 25, 6, 24};
    for (int i = 0; i < MAX_BOTS; i++) {
        bots[i].x = bx[i] << 8;
        bots[i].y = by[i] << 8;
        bots[i].dir = (i * 5) & 31;
        bots[i].alive = 1;
        bots[i].phase = i * 13; /* Phase prevents all drones turning together. */
    }
}

static void update_bots(void) {
    for (int i = 0; i < MAX_BOTS; i++) if (bots[i].alive) {
        int dx = cos32[bots[i].dir] >> 3;
        int dy = sin32[bots[i].dir] >> 3;
        int nx = bots[i].x + dx;
        int ny = bots[i].y + dy;

        if (!cell_solid(nx >> 8, ny >> 8)) {
            bots[i].x = nx;
            bots[i].y = ny;
        } else {
            bots[i].dir = wrap_angle(bots[i].dir + 8);
        }
    }
}

static void floor_effects(void) {
    uint8_t f = floor_tile[player_y >> 8][player_x >> 8];
    if (f == TILE_WATER && health < 9) health++;
    if (f == TILE_LAVA && health > 0) health--;
}
```

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

**Files to edit:** `assets/generate_assets.py`, `assets/manifest.json`, and generated files under `assets/include`.

**Code to add:** Students should add assets through the generator and manifest, not by copying bytes into C. The example below shows the pattern for a small generated sound or tile entry.

```python
# assets/generate_assets.py
def write_u16_array(out, name, values):
    # A tiny helper keeps generated headers deterministic and reviewable.
    # Deterministic output matters because students can compare git diffs.
    out.write(f"static const uint16_t {name}[] = {{\n")
    for i, value in enumerate(values):
        out.write(f"  0x{value & 0xffff:04x},")
        if (i + 1) % 8 == 0:
            out.write("\n")
    out.write("\n};\n")
```

```json
{
  "path": "assets/png/tiles_8x8.png",
  "license": "CC0-1.0",
  "source": "Original procedural pixel art generated for TerraForge 32",
  "notes": "Do not replace with textures copied from another game."
}
```

After editing assets or the generator, rebuild the generated headers:

```bash
python3 assets/generate_assets.py
```

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

**Files to edit:** `c/game.c`.

**Code to add:** Add the minimap and HUD after the world and player state exist. These functions are not decorative; they expose state that students need for debugging.

```c
static void draw_crosshair(void) {
    /* The crosshair marks the same center ray used by dig/place. */
    prg32_gfx_rect(156, 99, 8, 2, PRG32_COLOR_WHITE);
    prg32_gfx_rect(159, 96, 2, 8, PRG32_COLOR_WHITE);
}

static void draw_minimap(void) {
    int ox = 4;
    int oy = 126;
    prg32_gfx_rect(0, 122, 138, 78, PRG32_COLOR_BLACK);

    for (int y = 0; y < WORLD_H; y++) {
        for (int x = 0; x < WORLD_W; x++) {
            if (world[y][x]) {
                prg32_gfx_rect(ox + x * 2, oy + y * 2, 2, 2,
                               shade_for(world[y][x], 0, 2000));
            }
        }
    }

    /* Q8 to minimap pixels: divide by 128 because each tile is two pixels. */
    int px = ox + (player_x >> 7);
    int py = oy + (player_y >> 7);
    prg32_gfx_rect(px - 2, py - 2, 5, 5, PRG32_COLOR_WHITE);
}

static void draw_hud(void) {
    prg32_gfx_rect(0, 0, 320, 18, PRG32_COLOR_BLACK);
    prg32_gfx_text8(4, 5, "TERRAFORGE32 3D",
                    PRG32_COLOR_WHITE, PRG32_COLOR_BLACK);
    prg32_gfx_text8(126, 5, "HP", PRG32_COLOR_WHITE, PRG32_COLOR_BLACK);
    for (int i = 0; i < health; i++) {
        prg32_gfx_rect(148 + i * 5, 7, 4, 6, PRG32_COLOR_RED);
    }
    draw_crosshair();
}
```

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

**Files to edit:** normally none. Optional edits may go in `docs/build_and_release.md` when the classroom environment has a custom board URL or QEMU path.

**Code or command block to add:** Record the local deployment commands in a classroom handout or environment file. The comments explain which values are site-specific.

```bash
# Choose the architecture before each build so the output filename is explicit.
export PRG32_ARCHITECTURE=qemu
scripts/build.sh

# These paths depend on the local PRG32 checkout.
python3 "$PRG32_REPO/tools/prg32_game.py" upload-qemu \
  dist/terraforge32-qemu.prg32 \
  --flash "$PRG32_REPO/build-qemu/flash_image.bin" \
  --partitions "$PRG32_REPO/partitions_prg32.csv"

export PRG32_ARCHITECTURE=esp32c6
scripts/build.sh

# Replace the URL with the address shown by the classroom board.
python3 "$PRG32_REPO/tools/prg32_game.py" upload \
  dist/terraforge32-esp32c6.prg32 \
  --url http://192.168.4.1
```

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

**Files to edit:** `metadata/manifest.json`, `metadata/metadata.json`, `metadata/colophon.json`, and `scripts/pack-store-bundle.sh`.

**Code to add:** Store publication is data-driven. The manifest must name the files produced by the build so the store can install the correct cartridge for each target.

```json
{
  "abi": "prg32-metadata-1.0",
  "id": "org.uniparthenope.prg32.terraforge32",
  "title": "TerraForge 32 3D",
  "version": "1.0.0",
  "summary": "Original PRG32 first-person block cavern cartridge.",
  "architectures": [
    {
      "id": "esp32c6",
      "file": "terraforge32-esp32c6.prg32"
    },
    {
      "id": "qemu",
      "file": "terraforge32-qemu.prg32"
    }
  ]
}
```

```bash
# scripts/pack-store-bundle.sh
# Copy exactly the artifacts named by metadata/manifest.json.
# A mismatch here is a release bug, even if the game itself runs locally.
cp "$repo_dir/dist/terraforge32-esp32c6.prg32" "$stage_dir/terraforge32-esp32c6.prg32"
cp "$repo_dir/dist/terraforge32-qemu.prg32" "$stage_dir/terraforge32-qemu.prg32"

python3 "$prg32_repo/tools/prg32_game.py" pack-bundle \
  --manifest "$stage_dir/manifest.json" \
  --out "$repo_dir/dist/terraforge32-store-bundle.zip"
```

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

**Files to edit:** `docs/didactic_notes.md` and the student's release notes.

**Code or text to add:** The final laboratory should leave an explanation trail. Require students to add a compact engineering note for their most important change.

```markdown
## Student extension note

Changed subsystem: ray casting

What changed:
The ray count was changed from 80 four-pixel columns to 64 five-pixel columns.

Why it is suitable for PRG32:
The renderer performs fewer ray steps per frame, which may improve
responsiveness. The visual cost is lower horizontal detail.

How it was tested:
Static check passed, QEMU cartridge built, and the manual checklist was run:
rotate, move, dig, place, collect relic, restart.
```

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
