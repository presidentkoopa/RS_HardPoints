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
`rs-vrhp-grab-*` netevent that every handler receives. Which mod acts needs no
coordination at all: each one's swap returns immediately unless your hand is
inside one of its own anchors, and your hand is only ever in one place.

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

### Planned

- **Gesture-cast**: arm extended, palm rolled up, and the three mounts fire
  what's on them *in place* rather than drawing into your hand.
- **Paged forearm inventory**: treat a forearm mount as a row you cycle
  through rather than one fixed item.
