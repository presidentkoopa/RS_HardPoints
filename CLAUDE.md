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

**Wrist hardpoints (indices 3–5)** — move guns from the **opposing** hand into
the wrist mounts. Then go **open palm**: the held weapon goes **invisible**, the
hand drops to the **reach pose**, and the three stored weapons can be fired in
place. Net effect the owner is aiming for: *"four weapons in each hand"* — one
held plus three mounted around it.

**Forearm hardpoints (indices 0–2)** — **utility, not weapons.** Flares, shields,
usable inventory. *"assign inventory items for quick use."* Bombs are a grey area
because a bomb is arguably a weapon.

### What the code does not do yet

- **`contents` is `Array<Weapon>`** (`RS_HardPoints.zs:232`). A forearm mount
  **cannot hold a flare or a shield.** `Weapon` derives from `Inventory`, so
  widening to `Array<Inventory>` covers weapons *and* utility items; store/draw
  then branches on which it got (weapons route through `MoveWeaponToHand` /
  `PendingWeapon`, other inventory through a use/activate path).
- **The tier split exists but does not gate content.**
  `FOREARM_HOLSTER_END = 3` (`:75`) is already used for basis math (`:624`) and
  the marker feed (`:1145`), but nothing stops a weapon going in a forearm slot
  or a flare in a wrist slot.
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

## Open work

34 findings from the 2026-08-25 audit. The big ones:

**Gesture-cast (the open-palm firing) — six defects, one can crash:**
- `fireGesture` (`:1729`) omits `doSwap`'s wrong-hand guard, then `SetPsprites` a
  foreign weapon's Fire state. The same file documents that guard 700 lines away
  as a **VM abort**.
- Firing a mount **empties it** — `updateProps` reconciliation nulls the slot the
  same tic (`:1174`).
- Arming is **on at rest by default and ungated** (`:1484`), pinning
  `HardpointClaimOff` true forever.
- **No hysteresis** on the roll threshold (`:1487`) — boundary chatter spams
  haptics, console and weapon swaps.
- Never clears `bHolsterHidden`/`bNoAutoSwitchTo` (`:1686`), which this file says
  blocks firing entirely.
- **Not gated on edit mode** (`:1616`) — placement and live fire coexist.

**Lifecycle:**
- A level change **permanently bricks** any stowed weapon (`:56`) —
  `bHolsterHidden` outlives the per-level handler that clears it.
- The **saved layout never auto-loads** (`:1582`); tuning resets to the
  placeholder table every map.
- Edit-mode drag state is a single global, and a disconnecting player **leaks 12
  visible actors** (`:274`).

**Declared nowhere:** `rs_hardpoint_prop_scale` is read by
`RS_HardPointProp.zs:351` but declared in no CVARINFO — it silently runs at 0.18.

**Unreachable:** `rs-hardpoint-recalibrate` is handled (`:1526`) but has no alias
and no menu entry.

**Needs the arbiter:** two invisible independent backup/restore stacks for the
same `OffhandWeapon` slot (`:1727`); both managers acting on the shared
`rs-vrhp-grab-*` netevent (`:1600`).

**Open headset questions — both now live cvars rather than blocking:**
- **Wrist pitch.** `handBasisPose` reads `OffhandPitch` **raw** while every other
  consumer negates it, which would invert the vertical response of wrist anchors
  3–5. Turn on `rs_hardpoint_wristdump`, tilt, read the numbers. **Not fixed
  blind** — this file's rotation math has been hand-derived wrongly twice.
- **Roll is not in the anchor basis.** Built from the off hand's yaw+pitch only,
  so rolling the arm rotates a stored item without swinging its mount around the
  arm. Squarely in gesture-cast's path, since the arming pose *is* a roll.

## Dual-arm — specified 2026-08-26, not yet built

Owner wants mounts on **both** forearms. Menu shape, exactly as specified:

```
MAIN HAND                     OFF HAND
  Forearm slots  on/off         Forearm slots  on/off
  Forearm slots  1 / 2 / 3      Forearm slots  1 / 2 / 3
  Wrist slots    on/off         Wrist slots    on/off
  Wrist slots    1 / 2 / 3      Wrist slots    1 / 2 / 3
```

Eight controls, up to **12 mounts**. This replaces the current single `armMode()`
(`:1473`). Each arm needs its **own saved layout** — the mounts sit relative to
whichever controller wears them.

The old notes scoped this as **cloning the 6-index block into a second bank**
rather than threading a hand argument through every function, on the grounds that
the existing one-bank pattern is proven and a parameterised version touches far
more call sites for equivalent risk. That reasoning still holds.

Note the currently-dead off-hand path (`nearOff` pinned to -1, `:1608`) becomes
**live** once a main-arm rig exists: mounts on the off arm are reached by the main
hand, and vice versa. **Do not delete it.**

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
