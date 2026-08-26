# RS HardPoints

Six mount points, **three around each wrist**. Reach across with your other
hand, grip, and whatever's on that mount comes off it — or roll that wrist
palm-out and fire the mounts where they sit, without ever drawing them.

**Wrists fire, holsters draw.** A torso holster (RS_Holsters) puts a gun in
your hand. A wrist mount shoots in place. They're different verbs on purpose,
so they stop competing for the same reach.

---

- 3 wrist mounts per arm, both arms, each arm switchable on its own and
  dialable to 1 / 2 / 3
- placement mode — drag each mount onto your actual arm
- each arm keeps its own tuned layout; one save covers both
- marker rings you can see when empty, separate cold and hot colors
- stored items show their real model, auto-scaled to fit the ring
- grip to store/draw across to the other arm, removing the item from weapon
  cycling while it's stowed
- gesture-cast: roll a wrist palm-out and fire that arm's mounts in place
  (off by default — see below)

### Requires

- The UZDXREMA engine build (for the native hand-pose and holster fields this
  is built on)
- Any weapon pack providing standard `Weapon` classes — it borrows whatever is
  loaded and never references a specific mod by name

### Loads alongside RS_Holsters

RS_Holsters owns the 8 torso holsters (hip, head, pectoral); this owns the 6
wrist mounts. They were one mod and share a manager design, but nothing is
shared at runtime — separate classes, cvars, netevents, sprites, models and
profile files, so both can be loaded at once.

The one thing that genuinely had to be coordinated is grip. The engine
synthesises F13 (main hand) / F14 (off hand) for a holster-context grip, and a
key binds to exactly one alias — so if both mods claimed F13 the second one
loaded would silently win and the other would never see a grip press again.
Both bind their own alias name, and both aliases fire the same
`rs-vrhp-grab-*` netevent that every handler receives, and each mod's swap
returns immediately unless your hand is inside one of its own anchors.

That is **not** full arbitration, and this README used to claim it was. The
old wording — "your hand is only ever in one place" — is false the moment two
anchors from the two mods overlap, which is easy: a wrist mount rides your own
forearm and a hip holster sits on your body, and reaching one can put your hand
inside both. Both handlers then receive the same netevent, both find a claim,
and **both act on one grip press.** A real arbiter is being built for this; the
current state is one-in-one-place *by luck of placement*, not by design.

**No default binds.** Bind the store/draw keys yourself under Customize
Controls — F13 and F14 are what the engine sends if you want grip to do it.

### No posture profiles, on purpose

RS_Holsters keeps a seated and a standing profile because a hip or pectoral
anchor is placed relative to a *body*, and a table tuned standing does not fit
a seated one. These mounts are bolted to a tracked controller — your arm is in
the same place relative to itself no matter what the rest of you is doing — so
posture is not a variable here. There's one saved layout, not two.

### Which hand reaches which mounts

A hand can never claim gear on its *own* arm, and that isn't a tuning choice.
A mount is a fixed offset from its own arm's controller, so the distance from
that hand to it is the same number every frame no matter how you move — it
could never be a "did I reach it" test. So each bank is worked by the *other*
hand: your main hand stores and draws from your off wrist, your off hand does
the same to your main wrist.

Firing is the mirror of that. An arm arms *its own* three mounts, because the
arming pose is that wrist rolling palm-out.

### Status

Newer and less tuned than the torso holsters this split off from. Positions
are placeholders meant to be dragged in edit mode, not measured — and the
left/right starting offsets are mirrored between the two arms on the
assumption that "outer side" mirrors, which is a guess edit mode is meant to
settle. Known open item: the anchor position follows the arm's yaw and pitch
but not roll, so rolling a hand rotates a stored item without swinging its
mount around the arm.

There's no elbow tracking — every position is a fixed offset from that hand's
own pose. Real IK is waiting on a pending engine update.

### Forearm mounts are deferred

An earlier layout put three more mounts along the off forearm, meant for
*utility* items — flares, a shield, usable inventory — rather than weapons.
That tier is on hold until the content side of it exists (a stackable
`CustomInventory` is not a `Weapon`, and it has no Ready state for a mount to
draw a model from), and its menu rows are gone rather than sitting there doing
nothing. Everything learned about placing it is kept in the source so it does
not have to be re-derived.

### Gesture-cast — built, off by default

Roll a wrist palm-out and that arm's three mounts arm. Each has its own fire
key — six in all, three per arm — and pressing one runs *that item's own Fire
state* in place: it stays on the mount, hidden, rather than being drawn into
your hand. Firing a second or third before you roll back just re-seats a
different one; rolling back out returns whatever you were really holding.

**Switch it on with `rs_hardpoint_gesture_enable`** (Arm Hardpoints → Gesture-
Cast Arming). It defaults off for a reason: arming is a wrist-roll test, and a
hand hanging at rest already passes it, so leaving it on would hold that hand
permanently armed. It also needs that arm's wrist slots switched on, and edit
mode off.

The fire keys are still **placeholder binds** — bind them to throwaway test
keys, not to real grip/trigger/pad. Putting them on the real controller
buttons needs engine-side context gating that does not exist yet; without it a
trigger pull would fire the gesture *and* whatever that hand's own weapon
does.

The roll target and tolerance are shared by both hands. That assumes the two
controllers report roll in the same frame — if the debug dump shows them
reading opposite signs for the same physical pose, it becomes two settings.

### Planned

- **Forearm tier**: the deferred utility row, once inventory items (not just
  weapons) can live on a mount.
- **Paged mounts**: treat a mount as a row you cycle through rather than one
  fixed item.
