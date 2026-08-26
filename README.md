# RS HardPoints

Six mount points on your own off arm. Three along the forearm, three around
the wrist. Reach over with your other hand, grip, and whatever's on that
mount is in your hand.

Utility slots, not a second gun rack — grenades, a flashlight, a tool.

---

- 3 forearm hardpoints, or 6 with the wrist trio (toggle)
- placement mode — drag each mount onto your actual arm
- save your tuned layout to disk, so it survives a restart
- marker rings you can see when empty, separate cold and hot colors
- stored items show their real model, auto-scaled to fit the ring
- grip to store/draw, removing the item from weapon cycling while it's stowed
- gesture-cast: roll palm-out and fire the wrist mounts in place (off by
  default — see below)

### Requires

- The UZDXREMA engine build (for the native hand-pose and holster fields this
  is built on)
- Any weapon pack providing standard `Weapon` classes — it borrows whatever is
  loaded and never references a specific mod by name

### Loads alongside RS_Holsters

RS_Holsters owns the 8 torso holsters (hip, head, pectoral); this owns the 6
arm mounts. They were one mod and share a manager design, but nothing is
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

### Status

Newer and less tuned than the torso holsters this split off from. Positions
are placeholders meant to be dragged in edit mode, not measured. Known open
item: the anchor position follows the arm's yaw and pitch but not roll, so
rolling the off hand rotates a stored item without swinging its mount around
the arm.

There's no elbow tracking — every position is a fixed offset from the off
hand's own pose, so "where the forearm actually is" is approximated rather
than measured. Real IK is waiting on a pending engine update.

### Gesture-cast — built, off by default

Roll your off wrist palm-out and the three wrist mounts arm. Each has its own
fire key, and pressing one runs *that item's own Fire state* in place — it
stays on the mount, hidden, rather than being drawn into your hand. Firing a
second or third before you roll back just re-seats a different one; rolling
back out returns whatever you were really holding.

**Switch it on with `rs_hardpoint_gesture_enable`** (Arm Hardpoints → Gesture-
Cast Arming). It defaults off for a reason: arming is a wrist-roll test, and a
hand hanging at rest already passes it, so leaving it on would hold your off
hand permanently armed. It also needs the wrist tier live ("Which hardpoints"
set to *Wrist only* or *All six*) and edit mode off.

The three fire keys are still **placeholder binds** — bind them to throwaway
test keys, not to real grip/trigger/pad. Putting them on the real controller
buttons needs engine-side context gating that does not exist yet; without it a
trigger pull would fire the gesture *and* whatever your off hand's own weapon
does.

### Planned

- **Paged forearm inventory**: treat a forearm mount as a row you cycle
  through rather than one fixed item.
- **Both forearms**: mounts on the main arm too, reached by the off hand, with
  its own saved layout.
