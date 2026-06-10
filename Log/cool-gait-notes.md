# Cool Gait Command (`rc`) — Notes

Notes tracking the work to add a "cool gait" demo command to the Bittle firmware.

## How the pieces fit together

**Trigonometric core** — gaits are synthesized from trig oscillators in `src/motion.h`:
- The `CPG` class (Central Pattern Generator, `motion.h:384`) precomputes a cosine table —
  `precalcCos[i] = -cos(i * 2 * 3.14159 / _nSample)` (`motion.h:416`) — and phase-shifts it per leg
  in `sendSignal()` to drive the four shoulder joints (8–11). This is the rhythm engine behind
  walking/trotting/bounding.
- `signalGenerator()` (`motion.h:645`) is the simpler sibling: it drives any joint with a pure
  `sin()` wave from `jointIdx, midpoint, amp, freq, phase` tuples.

**Nested task queue** — `src/taskQueue.h`. `tQueue->addTask(token, params, delayMs)` chains commands
that fire one-after-another in `loop()` via `popTask()`. The existing `rg` command (`updateCPG()`,
`motion.h:531`) loads 8 trig-driven CPG presets into the queue back-to-back — the template for a
"gait demo."

## Skill data format (`InstinctBittleESP_arm.h`)

- **Postures** (e.g. `balance`, `str`, `sit`, `up`): header `{1, 0, angle, 1}` = a single stable frame.
- **Behaviors** (e.g. `mw` moonwalk at `-12`): negative frame count = one-shot routine with per-frame
  timing, hand-tuned by the designers to stay balanced.
- Skill names in `skillNameWithType[]` carry a trailing type letter (`mwI`, `ndI`, `strI`); the lookup
  drops it (`skill.h:10`), so the bare names (`mw`, `nd`, `str`, `sit`, `up`, `balance`) match.

## Attempt 1 — raw CPG dance (FAILED: robot toppled)

The first `rc` drove the CPG directly with aggressive amplitudes (trot / sashay / bound). This is
open-loop trig oscillation that fights the balance controller, so Bittle fell after the first
sequence. Lesson: don't hand-crank CPG amplitudes; they overpower balance.

## Attempt 2 — smooth skill choreography (CURRENT)

Replaced the CPG oscillation with a choreography of Bittle's own pre-tuned, balance-stable skills,
chained through the nested task queue. The per-task delay gives each skill time to finish before the
next loads.

```cpp
} else if (subToken == 'c') {  // "rc": smooth, balance-safe choreography from Bittle's own skills
  tQueue->addTask(T_SKILL, "balance", 1000);  // stand up and settle
  tQueue->addTask(T_BEEP, "14 4 16 4 19 4");  // little fanfare
  tQueue->addTask(T_SKILL, "str", 2000);      // a graceful stretch
  tQueue->addTask(T_SKILL, "up", 1000);       // back to a stable stand
  tQueue->addTask(T_SKILL, "mw", 4000);       // moonwalk — the elegant showcase
  tQueue->addTask(T_SKILL, "nd", 1500);       // a gentle nod
  tQueue->addTask(T_SKILL, "sit", 1500);      // sit down
  tQueue->addTask(T_REST, "", 0);             // relax
}
```

The centerpiece is `mw` (moonwalk) — a smooth `-12`-frame behavior that glides rather than lurches.

## Command to issue

Flash the firmware, open the serial monitor at **115200 baud**, and send:

```
rc
```

(`r` = `T_CPG` token; the `c` sub-token selects the choreography.)

## Bonus — pure `sin()` wave gait (no code change)

The signal generator is on the `o` token: `o resolution speed [jointIdx midpoint amp freq phase]...`.
This sends a traveling sine wave across the four shoulders (joints 8/9/10/11) at quarter-period offsets:

```
o 10 4 8 0 25 1 0 9 0 25 1 30 10 0 25 1 60 11 0 25 1 90
```

Start with a modest amplitude (25°) since it drives the joints directly without the CPG's gait shaping.

## Files changed

- `src/motion.h` — the `rc` choreography in `updateCPG()`.
- `src/OpenCat.h` — doc note on the `T_CPG` `#define` (`OpenCat.h:348`).

## Commit

- Branch: `abanerjee/changes`
- Commit: `9494a94` — `Add "rc" command: a smooth balance-safe choreography`
- Not yet pushed.

## Further options

- Tamer version: drop `mw`, build purely from postures (`balance → str → sit → up → rest`).
- Other gentle behaviors to swap in: `wh` (wave hand), `gdb` (good boy).
- For a trig-based gait that stays balanced, use the low-amplitude stock CPG presets via `rg`.

---

# New skill `grv` ("groove") — a custom balance-safe behavior

Added a brand-new skill directly into `src/InstinctBittleESP_arm.h`. The robot stands in place and
gently bobs while swaying/nodding its head — a slow meditative groove, unlike anything in the stock
skill list.

## Skill-array format (learned from studying the file + `skill.h`)

- **Header bytes** (`dataLen()` in `skill.h:118`): `4` for `period > 0`, `7` for behaviors (`period < 0`).
  - `{period, expectedPitch, expectedRoll, angleDataRatio}` then (behaviors only)
    `{loopStart, loopEnd, repeat}`.
- **`period`**: `>1` = looping gait, `1` = static posture, `<0` = one-shot behavior (`abs` = frame count).
- **`frameSize`**: gait = `WALKING_DOF` (8); posture = `DOF` (16); behavior = `DOF + 4` (20).
- **Behavior frame** = 16 joint angles + 4 trailing bytes `{transitionSpeed, delay, trigAxis, trigAngle}`.
  - transition speed: `transform()` gets `speedByte/8.0`; bigger = faster/fewer interpolation steps.
  - delay: holds `delayByte * 50 ms` after the frame (`skill.h:355`).
  - trigAxis/trigAngle: optional IMU gating (leave `0`).
- **loopCycle** `{start, end, repeat}`: at frame `== end` it jumps back to `start` until `repeat`
  passes complete (`skill.h:357`, `perform()` `skill.h:282`).
- **Joint columns**: `0` head pan, `1` head tilt (nod), `2` tail/arm clip, `4–7` unused (skipped when
  `WALKING_DOF==8`), `8–11` shoulders (FL,FR,BL,BR), `12–15` knees (FL,FR,BL,BR).

## Why `grv` cannot topple

- It's a behavior, so **gyro balancing is active** (its name isn't in the fast-motion exclusion list,
  `skill.h:284`) — the IMU actively corrects drift.
- **Shoulders are frozen** at the proven standing stance (`25,25,20,20` from `up`/`balance`), so the
  feet keep their stable footprint.
- **All four knees move identically** (`50 → 58 → 42`, only ±8°) → symmetric, so the center of mass
  never shifts in roll or pitch.
- Expression comes only from **head pan/tilt** (no balance impact); the arm (joint 2) stays neutral,
  matching how the codebase avoids perturbing it (`skill.h:496`, `mirror()` skips index 2).
- It **starts and ends in the standing stance** — no abrupt entry/exit.

## The data

```c
const int8_t grv[] PROGMEM = {
-6, 0, 0, 1,        // behavior, 6 frames, expected pitch/roll 0, angleRatio 1
 1, 4, 4,           // loop frames 1..4, repeat 4x
    0,   0, 0,0,0,0,0,0,  25,25,20,20,  50,50,50,50,   8, 2, 0, 0,  // neutral stand (entry)
   25,  15, 0,0,0,0,0,0,  25,25,20,20,  58,58,58,58,   6, 0, 0, 0,  // look left + dip
  -25,   0, 0,0,0,0,0,0,  25,25,20,20,  42,42,42,42,   6, 0, 0, 0,  // look right + rise
   25,  15, 0,0,0,0,0,0,  25,25,20,20,  58,58,58,58,   6, 0, 0, 0,  // look left + dip
  -25,   0, 0,0,0,0,0,0,  25,25,20,20,  42,42,42,42,   6, 0, 0, 0,  // look right + rise
    0,   0, 0,0,0,0,0,0,  25,25,20,20,  50,50,50,50,   8, 2, 0, 0,  // settle back (exit)
};
```

## Registration (must keep both arrays in the SAME order)

In `src/InstinctBittleESP_arm.h`:
- Added `"grvI"` to `skillNameWithType[]` (the trailing type letter is dropped at lookup, `skill.h:10`).
- Added `grv` to the full `progmemPointer[]`.
- Bumped the `//number of skills:` comment to 86.
- Both arrays validated at 86 entries, aligned, `grvI` → `grv`.

Note: skills load straight from PROGMEM here because `I2C_EEPROM_ADDRESS` is commented out
(`OpenCat.h:86`), so editing the two arrays + data is enough — no EEPROM rewrite needed.

## Command to invoke

Over serial (115200 baud):

```
kgrv
```

(`k` = `T_SKILL`; `grv` = skill name. `kgrvL` / `kgrvR` mirror the starting head direction.)

## Tuning

If the bob is too subtle or too strong, adjust the knee values (`58`/`42`) — keep all four equal and
within ~±12 of `50`. Only added to `InstinctBittleESP_arm.h` (the `ROBOT_ARM` build); a non-arm build
would also need it in `InstinctBittleESP.h`.
