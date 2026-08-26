# RS HardPoints — project context

Claude Code auto-loads this file for any session opened in this repo. If you're a
fresh agent reading this cold: this is the handoff doc. Not user-facing — see
README.md for that.

Recreated 2026-08-26. HEAD (`af63649`) deleted the previous one without saying so
in the commit message, while *adding* three code comments that point readers at
it. The old version is recoverable at `git show ed14594:CLAUDE.md`.

## Working with the owner

- **There is no test-compile here.** Don't re-verify ZScript syntax by re-reading
  files — build and find out from a real load / headset error. Owner's words:
  *"if there are syntax errors we will find out on compile, i can't spare 500k
  tokens for it."*
- **Don't bounce confirmations back as questions.** What the owner reports seeing
  in headset is ground truth.
- Expect rapid, informal, mid-turn corrections.

## THE DESIGN INTENT — owner's own framing, 2026-08-26

Read this before changing behaviour. The three systems are **not** three copies
of one idea; they have distinct purposes:

**Holsters (RS_Holsters, separate repo)** — storage for quick access to weapons.
*"Kinda like a favorites list for weapons."*

**Wrist hardpoints** — move guns from the **opposing** hand into the wrist
mounts. Then go **open palm**: the held weapon goes **invisible**, the hand
drops to the **reach pose**, and the three stored weapons can be fired in
place. Net effect the owner is aiming for: *"four weapons in each hand"* — one
held plus three mounted around it.

**HOLSTERS DRAW, WRISTS FIRE** (owner, 2026-08-26). A torso holster puts a gun
in your hand. A wrist mount fires its weapon **in place**, palm-out, without
ever drawing it. They are different verbs so they stop competing. Do not add
drawing behaviour to a wrist mount.

**Forearm hardpoints** — **DEFERRED as a feature, 2026-08-26.** Was going to be
**utility, not weapons**: flares, shields, usable inventory. *"assign inventory
items for quick use."* Bombs are a grey area because a bomb is arguably a
weapon. The menu rows are **removed** so they stop cluttering a menu they
cannot serve; everything learned about placing the row (the 90-degree yaw
correction, why pitch had to be forced flat, the rejected main-hand-aim
experiment, the tuned default offsets) is preserved verbatim in the
`DEFERRED: THE FOREARM TIER` block on `RS_HardPoints.zs`' constants.

### What the code does not do yet

- **`contents` is `Array<Weapon>`.** A mount **cannot hold a flare or a
  shield** — which is a large part of why the forearm tier is deferred.
  `Weapon` derives from `Inventory`, so widening to `Array<Inventory>` covers
  weapons *and* utility items; store/draw then branches on which it got
  (weapons route through `MoveWeaponToHand` / `PendingWeapon`, other inventory
  through a use/activate path).
- **HAND-BINDING COLLISION between the two verbs.** A mount is FILLED by the
  opposite hand (a hand cannot reach its own arm — `updateClaims`) but FIRED by
  the hand that WEARS it (the arming pose is that wrist rolling palm-out). With
  every weapon carrying `+WEAPON.NOHANDSWITCH`, an instance is permanently
  bound to one hand, so the weapon a reach naturally puts on a mount is exactly
  the one `fireGesture`'s wrong-hand guard refuses. **Pre-existing** — it was
  already true when the main hand filled all six off-arm mounts — and never hit
  because gesture-cast defaults off with placeholder binds. Written up in full
  at `fireGesture`'s guard. **Do not delete the guard**; it is what stops a VM
  abort. The fix is an owner decision between "arm the bank you can reach" and
  "give mounts a hand-agnostic copy".
- **Open-palm does not hide the held weapon or change the hand pose.**
  `POSE_REACH` exists (`hand_frames.txt`, `HOLD_BASE + 7`, "splayed, about to
  take hold") and `RS_Hands` already maps `GRIPSUBJ_Holster → POSE_REACH`
  (`rs_hands.zs:272`), so the pose half is a claim write. Hiding the held weapon
  is new work.

## The engine dependency

**UZDXREMA / DoomXR** (`E:\UZDXREMA`), **UZDoom 5.0.0-rc.2** (`src/version.h:29`),
built to `build-dxr\Debug\doomxr.exe` — Debug is the owner's real launch config.

`zscript.txt` declares `version "4.14"`. That is legacy: the floor is forced by
**RS_Hands** (`rs_grab.zs:123` calls a `version("4.15.1")`-gated native), and this
repo's const methods were checked against the 4.15.1 SafeConst rule and lift to
5.0.0 with no code changes.

Fork-only fields: `OffhandPos/Angle/Pitch/Roll`, the `Attack*` pair, `HmdPos*`,
`HardpointClaimMain/Off`, `GripContext*`, `level.VRHaptic`,
`level.GetActorModelClass`, `level.GetModel*Hint`, `level.JSONProfile*`.

**`AActor.StabilizeReach`** is live and per-weapon in **inches**, but the geometry
test it feeds is switched off — `vk_openxrdevice.cpp:3610` hardcodes
`stabilizeGeometryOk = false`. Proximity stabilize is **retired**; it now requires
a claim (off hand on the grip or forend).

## ZScript dialect — VERIFIED 2026-08-26

Three inherited cautions turned out **false**:

- **`clamp()` / `min()` exist** (lowercase). `base.zs:857`; seven sites across the
  mods including `RS_HolsterFlashlight.zs:143`. Only capitalised `Min()`/`Clamp()`
  are absent — the comment at `RS_HardPoints.zs:1132` claiming otherwise is wrong.
- **`double(x)` cast-as-call works** (`RS_Hands/zscript/rs_hands.zs:500`).
- **`Actor.Spawn` accepts a runtime `String`** (`RS_Hands/zscript/handworld.zs`).

Still avoid (no counter-example found): method-body `const`, field initializers.

**Two traps that WILL kill the build — both were hit on this branch:**

- **Private native fields.** `Actor.RenderStyle` is `native private`
  (`actor.zs:628`); read it from outside Actor and it is `MSG_ERROR`, not a
  warning. Use `GetRenderStyle()` (`actor.zs:971`). *If a symbol has no precedent
  anywhere in these five mods, that is usually because it does not work.*
- **`EventHandler.Find("Literal")` and `Service.Find(class<...>)` are
  COMPILE-TIME links.** A missing class is fatal AND **global** —
  `thingdef.cpp:420-424` refuses to compile every pk3 later in the load order.
  For soft lookups use `ServiceIterator.Find(String)` only.

## The fork — this repo's biggest structural problem

Split from RS_Holsters on 2026-08-23. Normalised diff: **852 differing lines of
4,338** in the managers, **61 of 1,299** in the props.

`isHandAnchored(idx)` is `idx >= HAND_HOLSTER_START` in **both** files. Here
`HAND_HOLSTER_START = 0` with `HOLSTER_COUNT = 6`, so it is **always true**. In
RS_Holsters it is `9` with `HOLSTER_COUNT = 9`, so it is **always false**. Each
fork therefore compiles a complete dead implementation of the other's live
direction. That is the merge's job, and it is what caps this mod's quality score
until then.

**WORKING RULE: any fix touching this file or RS_Holsters' equivalent must be
applied to BOTH in the same pass.** Every divergence bug found so far came from a
fix landing in one sibling and not the other — including the
`GetActorModelClass` one repaired on this branch.

**ONE DELIBERATE EXCEPTION, 2026-08-26: the dual-arm re-layout below.** It is a
change to what an *index means* in this fork's anchor table (main-arm bank vs
off-arm bank), not a defect repair, and the owner scoped it to this repo alone
— RS_Holsters owns eight torso anchors that have no arm to be laid out on. The
divergence therefore GREW: `handBasisPose`, `handAnchorPos`, `worldToHand`,
`updateGrabs`, `updateClaims`' self-claim guard, `holsterActive` and the profile
key schema now differ structurally between the siblings, on top of the
`isHandAnchored` inversion. Anyone doing the merge should read this section
first; the two files are further apart than the 852/4,338 figure above.

## Open work

34 findings from the 2026-08-25 audit.

### Fixed 2026-08-26 (do not redo — re-locate by name, line numbers moved)

All of these landed in **both** siblings where they existed in both.

**Gesture-cast, all six defects:**
- `fireGesture` now carries `doSwap`'s wrong-hand guard
  (`w.bNoHandSwitch && !w.bOffhandWeapon`) ahead of the `SetPsprite`.
- Firing a mount no longer empties it: `updateProps`' reconciliation exempts the
  one instance named by the new `gestureSeatedOff[]` array.
- Arming is gated on a new `rs_hardpoint_gesture_enable` (**defaults off**), on
  the wrist tier being live, and on edit mode being off.
- `GESTURE_ROLL_HYSTERESIS = 1.35` — enter at the tolerance, leave past it.
- `fireGesture` strips `bHolsterHidden`/`bNoAutoSwitchTo` so the weapon can
  actually fire; `updateGestureArm`'s falling edge puts them back, and multi-fire
  re-stows the previously seated one.
- `fireGesture` re-checks the master switch and edit mode itself, because a
  netevent can arrive between two `WorldTick`s.

**Lifecycle — the class had NO overrides but `WorldTick`/`NetworkProcess`:**
`PlayerDied`, `PlayerRespawned`, `PlayerDisconnected`, `WorldUnloaded`,
`WorldLoaded` now exist, plus `releasePlayer` / `despawnPlayerActors` /
`unstowInventory` / `autoLoadLayout`. That covers the level-change bricking, the
never-auto-loading layout, and the 12 leaked actors per disconnect. Edit-mode
drag state (`editMode`/`grabbedMain`/`grabbedOff`) is per-player now, and
`updateGrabs` calls `ensureEdit()` first so the int zero-default can never read
as "dragging holster 0".

**Declared:** `rs_hardpoint_prop_scale` is in CVARINFO with a menu row.
**Reachable:** `rs-hardpoint-recalibrate` has an alias and a menu row.
**Audible:** the store/draw cue named `rs_fx_holster`/`rs_allclear_ready`, which
only RS_Main defines — an undefined sound is silence, not an error, so it had
never played. The audio now ships here as `rs_hardpoint_fx_store`/`_ready`
(`SNDINFO.txt`, `sounds/RSHP*.ogg`); RS_Holsters ships the same under `RSHO*`.
**Gated:** `rs_hardpoint_verbose` covers the range edge lines and the automatic
post-store dump. The haptic and the manual dump are deliberately outside it.

### Still open

**Needs the arbiter:** two invisible independent backup/restore stacks for the
same `OffhandWeapon` slot (`gesturePreviousOff` here, `pouchPrevious*` in
RS_Holsters); both managers acting on the shared `rs-vrhp-grab-*` netevent. The
lifecycle pass only nulls those pointers when the pawn dies — it does not touch
the capture/restore protocol, deliberately.

**The "one hand, one place" claim was false and is now documented as false** in
all five places it appeared (both KEYCONFs, both `NetworkProcess` comments, this
repo's README). Overlapping anchors between the two mods mean one grip press can
be acted on by both handlers.

**Open headset questions — both now live cvars rather than blocking:**
- **Wrist pitch.** `handBasisPose` reads the wearing hand's pitch **raw** while
  every other consumer in the family negates it, which would invert the vertical
  response of every mount. Turn on `rs_hardpoint_wristdump` — it prints **both
  arms** now — tilt, read the numbers. **Not fixed blind** — this file's
  rotation math has been hand-derived wrongly twice.
- **Roll is not in the anchor basis.** Built from the wearing hand's yaw+pitch
  only, so rolling the arm rotates a stored item without swinging its mount
  around the arm. Squarely in gesture-cast's path, since the arming pose *is* a
  roll.

## Dual-arm — BUILT 2026-08-26 (re-spec, then implemented)

The owner re-specified the layout: **forearm slots disabled and their menu rows
removed**, **three wrist mounts per arm on BOTH arms**. Net anchor count
**unchanged at 6** — deliberately, so dual-arm arrives without adding anything
new to learn. Menu shape, as shipped:

```
MAIN HAND                     OFF HAND
  Wrist slots    on/off         Wrist slots    on/off
  Wrist slots    1 / 2 / 3      Wrist slots    1 / 2 / 3
```

Built as a **clone of the index block into a second bank**, not by threading a
hand argument through every function — the one-bank pattern was proven and a
parameterised version touches far more call sites for equivalent risk.

### What that means in the code

- `HOLSTER_COUNT` stays **6**. `OFF_WRIST_START = 3`, `WRIST_PER_ARM = 3`.
  Indices **0-2 = main arm**, **3-5 = off arm**.
- `FOREARM_HOLSTER_END` and `FOREARM_YAW_CORRECTION` are **gone as symbols**
  (a constant whose value stays 3 while its meaning inverts is a trap); their
  content survives in the `DEFERRED: THE FOREARM TIER` comment block.
- **The off bank kept indices 3-5** so the three gesture-fire netevents, their
  KEYCONF aliases and any bind a player already made still mean what they meant.
- `isMainArmAnchor(idx)` is the one predicate everything branches on.
  `handOrigin` picks `AttackPos` vs `OffhandPos`; `handBasisPose` picks the
  matching angle/pitch/roll and now has **no special case in it at all**.
- `armMode()` is replaced by `wristCount(bool mainArm)` / `wristTierLive(bool)`,
  reading four new cvars: `rs_hardpoint_wrist_main{,_count}` /
  `rs_hardpoint_wrist_off{,_count}`. `rs_hardpoint_arm_active_count` is
  **undeclared** now.
- **`nearOff` is live.** The self-claim exclusion in `updateClaims` is symmetric
  now: a hand can only claim the OTHER arm's bank. `updateGrabs` grew back its
  off-hand hand-anchored branch, which had been structurally unreachable.
- **Per-arm saved layout.** Profile keys are arm-tagged: `m0_`..`m2_` and
  `o0_`..`o2_` (`profileKey`), replacing the flat `h0_`..`h5_`. That matters
  because `loadProfile` falls back **per field** rather than erroring, so a
  stale `h0_fwd = -4.0` from the forearm era would have silently loaded into
  MainWristBelow. Old profiles now simply do not match and every field falls
  back to the new default. One save slot still covers both arms.
- **Gesture-cast is per arm.** `gestureArmedMain` / `gesturePreviousMain` /
  `gestureSeatedMain` + `updateGestureArmMain` + `fireGestureMain` mirror the
  off-hand originals (`AttackRoll`, `PSP_WEAPON`, `ReadyWeapon`, hand 0, haptic
  channel 0). Three new netevents
  `rs-hardpoint-gesture-main-{grip,padx,trigger}` map to mounts 0/1/2.
  `updateClaims` now ORs the armed flag into **both** claim fields, and the
  main-hand edge test was switched to the same value it writes (it had been
  reading the un-ORed one).

### Still open on the dual-arm work

- The **hand-binding collision** above. It is the one thing that can make
  gesture-fire refuse every mount, and it needs an owner decision.
- **hsSide is mirrored between the banks as a guess** — with both hands
  pointing the same way, identical `hsSide` would stack both banks on the same
  side of the body rather than putting each on its arm's outer side. Which side
  is "outer" is not measured. Edit mode settles it.
- **The gesture roll target/tolerance are shared by both hands**, which assumes
  the two controllers report roll in the same frame. `rs_hardpoint_wristdump`
  now prints both arms specifically so that can be read rather than guessed.

## Where the audit lives

`github.com/presidentkoopa/RS_VR_Unified` — 190 findings across five mods, 26
adversarially verified. `TESTING.md` there is the current test list.

## Forearm content — worked example the owner supplied

`D:\SteamLibrary\...\notinuse\Gameplay_RastaFlares` — *"example of what I would
use forearm slots for, example only."*

```
Class Flare : CustomInventory
    +Inventory.InvBar
    Inventory.MaxAmount 1000
    Inventory.Icon "FLARD0"
    ... Use state spawns ActiveFlare : Actor
```

Three concrete consequences for the forearm tier:

1. **A utility item is a `CustomInventory`, not a `Weapon`.** Drawing it means
   `UseInventory()`, not `MoveWeaponToHand`.
2. **It is STACKABLE** (`MaxAmount 1000`). A forearm slot therefore holds an item
   *and a count*, not a single instance pointer like `contents[]` does today.
3. **It has no `Ready` state.** `ShowWeapon` does
   `State rs = w.FindState("Ready"); if (rs == null) { SetVisible(false); return; }`
   — so a flare in a mount renders as **nothing at all**. The prop needs to fall
   back to `Spawn` (or use `Inventory.Icon`) for non-Weapon contents.
