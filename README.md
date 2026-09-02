# to DIE for -- Godot prototype

A 2D roguelite where you draw a hand of dice from your bag and roll
for sixes against a per-round target.

The project layout follows the same conventions as your Space Shooter
project: matching `.tscn` + `.gd` pairs in `Scenes/`, a single `Global`
autoload in `global/`, `@onready var` node refs at the top of each
script, `_on_<node>_<signal>()` handler names wired through the scene's
`[connection]` entries, and `change_scene_to_packed()` for screen
transitions.

## Opening it
1. Godot **4.x** -> Project Manager -> **Import** -> pick this folder.
2. Press **F5**. It boots to the start screen.

I still have no Godot binary to test against, so this is verified by
static checks (every `$NodePath`, signal connection, `ExtResource`,
`SubResource` and `Global` member is confirmed to resolve, and all
`:=` declarations are typed to avoid the inference-warning error you
hit before) rather than by actually running it. Send me any error text
and I'll patch it.

## Scene map
| Scene | Role |
|---|---|
| `Scenes/start_screen.tscn` | Title screen. The project's main scene, so it boots first. Shows carried-over unlocks and swaps to `main`. |
| `Scenes/main.tscn` | The game screen. Deliberately thin -- starts the run, wires children together, handles transitions. |
| `Scenes/hud.tscn` | Top bar: round, bank, round-progress bar, rolls left. |
| `Scenes/dice_tray.tscn` | Your drawn tray. Owns the dice row, Roll button and result log. |
| `Scenes/die.tscn` | One die. Instanced per die in hand by `dice_tray.gd`, the way `mine.tscn` is instanced by `level.gd` in Space Shooter. |
| `Scenes/dice_bag.tscn` | The clickable bag icon + count badge + contents panel. |
| `Scenes/shop.tscn` | Between-round shop overlay. |
| `Scenes/game_over.tscn` | Bust screen, its own scene like your `game_over.tscn`. |
| `global/global.gd` | All run state and rules. The only place gameplay math happens. |
| `global/die_data.gd` | `DieData` -- one die's stats and roll behaviour. Pure logic, no visuals. |
| `tools/make_dice_bag_icon.py` | Regenerates the bag icon if you want to tweak it. |

## The dice bag
The pouch icon sits at the bottom of the screen with a live count
badge. Click it to see everything you own, grouped by kind with a
description of what each die does. `assets/dice_bag_icon.png` is
generated pixel art -- 32x32 drawn then scaled 8x with nearest-neighbour
so the pixels stay hard -- in dark purple and gold with a black 1px
outline. Re-run `python3 tools/make_dice_bag_icon.py` after editing
the palette or shape constants at the top of that script.

## Shop pricing
Prices are a fraction of the threshold you just cleared, so they track
the run's scale on their own instead of needing a separate curve:

| Tier | Price | Dice |
|---|---|---|
| Basic | 1/6 of last threshold | Basic Die |
| Mid | 1/3 | Cracked, Lucky Charm, Loaded |
| Top | 1/2 | Glass, Bone, Mirror |

## One design change I had to make, and why
Simulating 6,000 runs showed the shop was a **trap** as specced: with
the round target measured against your total bank, every purchase
directly undid progress toward the next round, so buying nothing beat
buying greedily (mean round 3.0 vs 2.3). That defeats the whole point
of a dice-bag loop.

So the two numbers are now separate:
- **Bank** -- persistent, and what you spend in the shop.
- **Round progress** -- resets each round, and is what actually clears
  the round. Spending can never undo it.

Both are shown separately in the HUD, with a progress bar for the
round. With that fixed, shopping clearly beats hoarding.

## Balance
Also retuned, since the old curve walled out every run by round 3.
The real ceiling is that you only ever draw 6 dice no matter how big
the bag gets, so earnings are capped by hand *quality*, not quantity --
which a 45%-per-round threshold growth outruns almost immediately.

Now: base target **$8**, growth **1.10x**/round, **22** rolls/round.
Over 6,000 simulated runs:

| | ignore the shop | use the shop |
|---|---|---|
| median round | 5 | **6** |
| mean round | 4.9 | **6.5** |
| cleared round 1 | 90% | 90% |
| reached round 5 | 52% | **66%** |
| reached round 10 | 5% | **22%** |
| reached round 15 | 0% | **2%** |

All of these live as constants at the top of `global/global.gd`.

## Dice (27 kinds)
20 new dice were added on top of the original 7, all available in the
shop pool from the start -- with this many kinds, gating them behind
achievements would mean most never appeared.

**Basic (1/6 of last threshold):** Basic Die.

**Mid (1/3):** Cracked, Lucky Charm, Loaded, Anchor, Echo, Ivory,
Rusted, Serpent, Tithe, Chisel, Mimic, Clockwork, Ashen, Sable, Grafted.

**Top (1/2):** Glass, Bone, Mirror, Twin, Gilded, Hollow, Prism,
Feather, Coin, Warden, Snake Eyes.

Several dice need to see the whole hand, so `roll_hand()` in
`global/global.gd` now runs in six ordered phases: each die rolls ->
copying dice (Mirror, Echo) overwrite from the *original* faces ->
Serpent converts a neighbour -> Warden cancels the first 1 -> tally
payouts and jackpot -> clean up whatever broke or got culled.

### Balance work on the new dice
Isolating each kind (shop offers only that die, 1,200 runs each)
found three that stacked into runaway loops, and they were fixed:
- **Mirror** copied other Mirrors, so a bag of them fed off each
  other. It now ignores other Mirrors.
- **Feather** grew the hand without limit -- the one thing that
  scales forever. Capped at 3 extra slots.
- **Snake Eyes** paid its jackpot once per die in hand. Now it pays
  once per hand, at 6x instead of 10x.

Two were buffed for being strictly worse than a Basic Die: **Rusted**
(now starts at 5% and gains 3%/round) and **Tithe** (fee cut to 0.35).
That leaves a spread of roughly 3.2 to 17.6 mean rounds in isolation,
with no runaways. Hollow sits lowest on purpose -- it pays nothing
itself and only works alongside other dice.

With the full pool, over 4,000 runs:

| | ignore the shop | use the shop |
|---|---|---|
| median round | 5 | **10** |
| mean round | 4.8 | **10.3** |
| reached round 10 | 5% | **53%** |
| reached round 15 | 0% | **29%** |
| best seen | 15 | **36** |

## Settings, profiles and shop items
- **Settings** (`Scenes/settings_menu.tscn`, opened from the title
  screen): master and SFX volume, fullscreen, reduced motion, and a
  colourblind option that draws numerals on dice instead of pips.
  Saved to `user://settings.cfg`; every change applies immediately.
- **Save slots**: Start goes to a 3-slot profile picker showing best
  round, runs played, achievements and dice unlocked, with an Erase
  button per slot. Achievements write through the moment they unlock.
- **Loaded Deal** (shop item): pulls 3 Basic Dice out of your bag for
  good. Priced top-tier because thinning is disproportionately strong
  here -- you only ever draw 6 dice, so removing basics is what makes
  your specials actually show up. Only offered when you still have 3
  basics to remove.

## Unlock ladder (the replay loop)
You now start with **3 dice** (Basic, Cracked, Loaded) and earn the
other 24 by completing objectives -- "roll two 1s in one hand",
"reach round 15", and so on. Open the dice bag in-game to see every
locked die and what it wants.

Two design rules make this actually produce progression, both of
which the simulation forced:

**Unlocks apply to the NEXT run.** The shop pool is snapshotted when
a run starts. Without this, one deep run unlocked ~20 of 27 dice
mid-run, so run 1 ended up as strong as run 20 -- the first run was
literally the best one, which is the opposite of the goal.

**Objective difficulty tracks die power.** The first pass gave the
strongest dice (Mirror, Coin, Snake Eyes, Twin) trivial objectives
like "roll two 6s in one hand", so run 2 already had the full power
curve and everything after it was flat. The strong dice now sit
behind the hardest objectives, and the weak/combo pieces unlock early.

Simulated over 900 careers of 16 runs each:

| run # | median round | dice in pool | reached round 15 |
|---|---|---|---|
| 1 | 4 | 3 | 0% |
| 2 | 8 | 11 | 13% |
| 3 | 9 | 15 | 21% |
| 5 | 10 | 19 | 28% |
| 8 | 10 | 22 | 31% |
| 16 | 10 | 24 | 29% |

Each of the first ~8 runs is measurably better than the last, then it
levels off once nearly everything is unlocked and it comes down to
skill and RNG. That's the curve you asked for.

## Boss rounds
Every 10th round is a boss: 1.5x the normal target, and one of six
random rule changes, each a real drawback with a compensating twist.

| Boss | Effect |
|---|---|
| The Short Hand | Draw only 4 dice -- but 6s and 1s hit twice as hard |
| The Cold Deck | Sixes pay half -- but the jackpot multiplier doubles |
| The Viper | Ones cost triple -- but sixes pay double |
| The Hourglass | Only 12 rolls -- but the target is a quarter lower |
| The Tax Man | Every roll is taxed -- but the jackpot starts a step higher |
| The Mirror Match | Every special rolls as a Basic -- but payouts are tripled |

Beating one pays a bonus worth half the target and widens the next
shop to 4 offers. Round 10 is a visible difficulty spike in the
death distribution (6.6% of runs end there vs ~3% on normal rounds),
which is the point.

## Other changes this pass
- **Roll cooldown** -- 1 second after each roll (`ROLL_COOLDOWN` in
  `global/global.gd`), so the button can't be spammed and each result
  gets a beat to land.
- **6s and 1s render 1.25x bigger**, tweened in place so growing a die
  never reflows the row. Skips the tween when reduced motion is on.
- **Lucky Charm counts only its own rolls.** The countdown used to sit
  inside `roll_face()`, which is also called by Bone's bonus reroll --
  and a Mimic copying a Lucky Charm started at -1 charges, so it burnt
  out on its first roll. It now ticks once per die per hand from
  `DieData.on_rolled()`.
- **Dice-bag rows show a picture of the die** next to the name, in both
  the owned and locked lists (locked ones washed out). A name like
  "Sable Die" tells a new player nothing about what they'd see on the
  tray.
- **Clearing a round holds for a beat** before the shop appears, so the
  hand that actually won is still readable, and the progress bar counts
  up or down over roughly one roll cooldown instead of snapping.
- **Project targets Godot 4.7**, matching your Space Shooter project.
- **Dice bag is now a pause menu** -- opening it pauses the tree and
  blurs the game behind it (`assets/blur.gdshader`, a cheap mip-level
  blur). Escape closes it. It lists what you own on the left and every
  locked die with its objective on the right.

## Characters
Choosing a save slot now leads to a scrollable player picker. Each
character is a different opening bag rather than a stat line, so the
choice changes how a run plays from the first roll. Locked ones stay
visible with their objective, same as the locked dice.

| Character | Starts with | Modifier | Unlocked by |
|---|---|---|---|
| The Regular | 6 Basic | -- | always available |
| Four Eyes | 2 Glass | -3 rolls | unlock the Glass Die |
| The Gambler | 4 Coin | -7 rolls | unlock the Coin Die |
| The Watchmaker | 6 Clockwork | -- | unlock the Clockwork Die |
| The Understudy | 3 Mimic + 3 Basic | -- | unlock the Mimic Die |
| The Snake Charmer | 2 Snake Eyes + 2 Basic | -4 rolls | roll snake eyes with a Snake Eyes Die |
| The Collector | 10 Basic | +3 rolls | carry 12 dice at once |
| The Ascetic | 3 Basic | +25% per six | clear a round in 6 rolls or fewer |

Most unlock off the die they're built around, which gives each die
unlock a second payoff.

**The roll penalties exist for a reason.** Simulation showed a small
*pure* bag is strictly better than six Basics, because your hand is
min(6, bag) -- so starting with 4 Coin Dice is a free Loaded Deal, with
every draw guaranteed to be your good dice. Untuned, The Gambler and
The Snake Charmer both averaged round 18 against The Regular's 11.
Roll-budget penalties bring the whole roster into a 10-13 band while
keeping each one's identity.

## Pause menu and suspended runs
Escape opens a pause menu (Continue / Settings / Player Select / Exit
to Main Menu). It freezes the tree and blurs the game behind it, the
same treatment the dice bag gets. Escape closes the dice bag first if
that's open -- `main.gd` owns the key so the two can't race for it.

**Leaving mid-run keeps the run.** Both exits park the whole thing on
disk (`user://run_slot_N.json`): bag, hand, round, boss, threshold,
bank, rolls left, and every die's accumulated state (Ivory's earned
payout, Clockwork's position in its cycle, Grafted's absorbed power).
The player screen then offers **Continue Run** alongside Start Run,
labelled with who's playing and what round they're on.

The hand is stored as *indices into the bag* rather than as copies,
because the two hold the same objects -- copying would split one die
into two that then drift apart.

Only two things discard a suspended run: starting a new one (the Start
button says so outright) and going bust.

## Character unlock chain
Character objectives are now scoped to *who did it*, so the roster
gets played rather than everyone settling on one favourite. Every
stat is filed twice -- once overall, once under the character who set
it -- which is what lets an objective say "as The Gambler, ...".

| Character | Unlocked by | Odds per run |
|---|---|---|
| The Regular | always available | -- |
| Four Eyes | unlock the Glass Die | -- |
| The Watchmaker | As The Regular, reach round 12 | 48% |
| The Gambler | As Four Eyes, reach round 5 | 82% |
| The Understudy | As The Watchmaker, reach round 8 | 87% |
| The Snake Charmer | As The Gambler, roll two 1s in a single hand | 100% |
| The Collector | As The Understudy, own 8 different kinds at once | 78% |
| The Ascetic | As The Collector, carry 16 dice at once | 83% |

Simulated per-run odds are in the last column. The Regular -> round 12
is the deliberate gate at ~4 runs; after that each character opens the
next in a run or two, so the chain keeps moving.

## The dice throw
Pressing Roll throws the dice into the tray: they fly in from off the
top edge at staggered intervals, spinning 2-4 full turns, land scattered
at random spots with a slight tilt, sit there for a beat, then slide
into an even horizontal row and straighten up.

Timings live at the top of `Scenes/dice_tray.gd` and total 0.90s for a
full 9-die hand -- deliberately just under `ROLL_COOLDOWN`, so the dice
have always settled before another roll is possible.

Three details that took some care:

- **The dice moved out of an `HBoxContainer`.** A container overwrites
  any position you set, so the row is now a plain `Control` laid out by
  hand in `_row_positions()`.
- **Nothing can spill off the felt.** The row's step is capped at
  `(width - 64) / (count - 1)`, which makes the row mathematically
  guaranteed to fit however many dice a Feather-heavy bag draws -- they
  tighten up rather than overflow. The scatter area is separately inset
  by 10px, because a tilted square reaches about 9px further than its
  own width and would otherwise hang a corner over the edge.
- **Faces are revealed on impact**, not when the roll resolves, so the
  result reads as the outcome of the throw. A generation counter lets a
  throw abandon itself if the hand is rebuilt mid-flight (a Glass Die
  shattering, say), instead of touching freed nodes.

Reduced motion skips the flight entirely and just shows the result.

## Fixes in this pass
- **Dice settle upright.** The spin count was `randf_range(2.0, 4.0)`
  turns -- a fraction, so 2.7 turns left a die resting at 252 degrees.
  It's now a whole number of turns, which lands exactly straight.
- **The row is inset from the wooden frame.** The felt's right edge sits
  almost exactly where the dice area ends, so `EDGE_INSET_RIGHT` pulls
  the usable width in; the row and the scatter both respect it, so
  nothing sits on the wood.
- **Erasing a profile erases its parked run.** `delete_slot()` only
  removed `save_slot_N.json`, so erasing mid-run left a clean-looking
  profile that could still resume the run you thought you'd deleted. It
  now clears `run_slot_N.json` too, and wipes the in-memory state if the
  erased slot is the one currently loaded.
- **Fullscreen fills the monitor properly.** Two separate causes: the
  stretch aspect was `expand`, which grows the viewport without scaling
  the content, so the tray stayed 620px on a 4K screen. It's now `keep`,
  which scales everything up (1.5x at 1080p, 3x at 4K). And the toggle
  now sizes the window to the current monitor's resolution, remembers
  where the window was, and restores plus centres it on the way out.

  `keep` was chosen over `keep_height` deliberately: keep_height would
  have clipped the sides on a 16:10 screen, where the visible width
  drops to 1152 against a 1280 base. `keep` never clips -- 16:9 fills
  exactly, other ratios get small bars.

## Dice sprite sheet
`assets/dice_sheet.png` -- 8x2 cells of 128px, 1024x256 total:

| Row | Cells | Contents |
|---|---|---|
| 0 | 0-5 | three-quarter hero views, faces 1-6 |
| 1 | 0-7 | 8-frame tumble loop |

The dice are an actual cube: eight corners rotated in 3D,
orthographically projected, back-face culled and lit per face. That is
what gives the sides real perspective, and it means the tumble row is a
genuinely rotating cube rather than a flat sprite being squashed --
which is what the previous version did, and it never looked like the
die had sides.

Face numbering comes off the unfolded net, so opposite faces sum to 7
(1-6, 2-5, 3-4). Because the cube is real rather than drawn, the two
side faces visible in any frame are automatically ones genuinely
adjacent to the top face. Hand-picking them is how you end up with an
impossible die showing, say, 2 and 5 at once. Verified: all six hero
frames show exactly three faces, the requested value on top, and never
an opposite pair.

Art is composed at 64px and scaled up 2x with nearest-neighbour, so the
pixels stay chunky and match the tray. Lighting is quantised to four
shades rather than a smooth gradient, for the same reason -- a smooth
ramp reads as a 3D render, not a sprite.

Three tinted variants ship alongside (`_crimson`, `_violet`, `_jade`);
`tools/make_dice_sheet.py` regenerates any of them and the palettes are
a dict at the top:

    python3 tools/make_dice_sheet.py            # ivory
    python3 tools/make_dice_sheet.py crimson

**Not wired into the game yet, on purpose.** `Scenes/die.gd` still draws
each die procedurally, which is what lets all 27 kinds have their own
body colour. Swapping to the sheet means either a palette per kind or
tinting one sheet with `modulate` -- your call, and easy either way,
since `die.gd` is the only file that would change.

## Sprite sheet is wired in
`Scenes/die.gd` now draws from the sheet instead of drawing squares by
hand. Row 0 supplies the six faces; row 1 supplies the tumble, which
plays while a die is airborne and stops the instant it lands.

Because the sheet's spin now does the work, **the throw no longer
rotates the node**. Doing both looked like two spins fighting each
other, and it also removes the need to unwind each die to straight when
the row forms.

Drawn sizes are exact divisors of the 128px source cell -- 64px on the
tray (2:1) and 32px in the lists (4:1) -- so with nearest filtering the
pixels stay square instead of smearing.

### Resolution
Composed at 128px internally and written out at 256px cells
(2048x512 sheets), up from 64/128. This is genuinely more detail, not
an upscale. It shows on high-DPI screens, because the `canvas_items`
stretch mode renders at the device resolution rather than the 1280x720
base -- at 1440p a 64-unit die is drawn with 128 real pixels. Drawn
sizes stay exact divisors of the cell (64 on the tray = 4:1, 32 in
lists = 8:1), so nothing smears.

### The art in assets/ is hand-edited
The four sheets carry edits made outside the generator: the light
catch-light along each cube's upper edges was replaced with solid
black, so every edge now has the same outline weight and the silhouette
reads cleanly at size.

`tools/make_dice_sheet.py` has been updated to match, and now writes to
`tools/generated/` rather than `assets/` -- a regenerate silently
overwriting hand-edited art is a trap worth designing out. Copy across
deliberately if you want generated output to become the live art.

### Palettes
27 die kinds, four sheet palettes. Each kind takes the palette closest
to its character, then a per-kind `modulate` separates it from its
neighbours, so every kind stays distinct without 27 hand-drawn sheets.
The palette carries meaning:

| Palette | Reads as | Kinds |
|---|---|---|
| ivory | plain and mechanical | Basic, Glass, Bone, Ivory, Sable, Rusted, Clockwork, Chisel, Gilded, Tithe |
| crimson | risk and breakage | Cracked, Coin, Ashen, Snake Eyes |
| violet | copying and jackpot tricks | Mirror, Echo, Twin, Prism, Hollow, Grafted, Mimic |
| jade | protection and income | Lucky Charm, Serpent, Anchor, Warden, Loaded, Feather |

The colourblind numeral option still works: pips are baked into the
sheet now, so it lays an outlined numeral over the die rather than
swapping the pips out.

## Economy: reroll and tier rarity
Money was outpacing anything to spend it on. Simulation found the real
cause was not income but the **bag cap**: once you hit 20 dice, buying
stops entirely, and 11.6 purchases per run were being refused for lack
of room. Three changes:

- **Shop reroll.** Costs 1/4 of the threshold you just cleared and
  multiplies by 1.75 each time within a visit, resetting when you
  leave. A burst of spending you choose to make, not a tax.
- **Tier rarity.** Offers are now weighted 6 / 2.5 / 1 across basic /
  mid / top. Top-tier dice went from 40.6% of offers to 20.9%, and
  hunting a specific one went from a median of 6 shops to 12 -- which
  is what gives rerolling something to chase.
- **A reserved thinning slot.** Once the bag is within one of the cap,
  one offer is guaranteed to be a thinning item. Being at the cap
  should be a decision -- pay to cull, then pay to upgrade -- not a
  dead end where the shop only shows dice you have no room for.

Together, money leaving the player's hands per run went from $56 to
$124, and unspent money at bust dropped by roughly a third.

## Consumables
Four bag-thinning items now, up from one. The gap they close: only
Basic Dice could ever be removed, so a special die that stopped fitting
the build was stuck in the bag forever, diluting every draw.

| Item | Tier | Effect |
|---|---|---|
| Loaded Deal | top | Removes 3 Basic Dice |
| Clean Sweep | top | Removes EVERY Basic Die at once; needs 4+ to appear |
| The Broker | mid | Pick any one die -- specials included -- and scrap it |
| Melt Down | top | Pick any two to scrap, refunding half their shop value |

The Broker and Melt Down open `Scenes/dice_picker.tscn`, a grid of your
whole bag with each die shown as itself. They stay available with zero
Basics left, which is exactly the case that had no answer before. Every
item is gated on whether it would actually do something, and the
pick-based ones refuse to leave you with fewer than two dice.

## Unlock popup
Unlocks were a toast that scrolled past. They now stop and show
themselves: `Scenes/unlock_popup.tscn` on layer 5, in front of the HUD,
shop and tutorial, with the die drawn large from the sheet, its name,
the objective that earned it, and what it does. Character unlocks use
the same popup and show that character's opening bag as dice.

Clearing a round can trip a die unlock and a character unlock together,
so they queue and show one at a time rather than overwriting each other.

## Tutorial
Ten steps, shown on a profile's first run. It doesn't fake anything --
it guides the player through their real first round, so nothing has to
be re-learned when it ends. It covers the hand, what 6s and 1s do, the
round target, the bank (and that spending never undoes round progress),
the roll budget, the jackpot, and the dice bag.

Two steps wait on an actual action -- pressing Roll, and opening the
bag -- with Next disabled until it happens. That's what stops it being
a wall of text you click through without reading.

**Skip is always on screen.** Finishing or skipping switches the
checkbox off for next time, and it stays visible so it can be turned
back on deliberately. The checkbox is its own saved preference now --
it used to derive from "has the tutorial run", so unticking it never
stuck and the box came back ticked on the next visit. The flag is per save slot, and the character screen has a
"Play the tutorial" checkbox, ticked by default until the profile has
seen it once and available afterwards to replay. Resuming a suspended
run never triggers it.

The dimmer cuts a hole around whatever is being explained, framed in
gold, and passes clicks straight through so Roll and the bag still
work. It's a `CanvasLayer` above the HUD rather than a plain Control --
the HUD is itself a CanvasLayer, so a Control dimmer would have been
drawn underneath it and the bank and target could never have been
dimmed.

## Title screen animation
`Scenes/title_animation.tscn` loops behind the menu: a shrouded dealer
in shirt, waistcoat and bow tie, palms turned up, with three dice
turning above them.

The dice are real `Die` instances running the sprite sheet's tumble
row, so the spin is literally the in-game spin rather than a separate
animation that could drift out of step with it.

Colour drifts through all 27 die kinds. They sit on four different
sheet palettes, so a straight switch would be a hard cut -- instead
each slot keeps two dice stacked and crossfades, the outgoing kind
fading out as the incoming one fades in. A full pass through every kind
takes about 28 seconds.

The figure is original pixel art from `tools/make_dealer.py`, built to
the same rules as the rest: composed small, scaled with
nearest-neighbour, hard black outline. Two details that needed solving:
the arms vanished into the torso because both are near-black, so the
outline pass draws the armpit and elbow seams by hand; and a near-black
figure on a near-black title screen is just a hole, so there's a cold
rim light down the upper-left silhouette.

The palm anchors are printed by the generator and read by the scene as
fractions of the dealer's rect, so the dice stay over the hands at any
size rather than being positioned twice and drifting apart.

Reduced motion rests the dice on a face and stops the bob; the colour
crossfade still plays, since a slow fade isn't the uncomfortable part.

## Relics
Permanent for the rest of a run, bought from the **left half of the
shop** -- relics on the left, dice on the right, split by a divider.
The two halves answer different questions: dice change what you draw,
relics change the rules those dice play by.

25 to start, across three tiers (9 common, 10 rare, 6 mythic), priced
at 1/3, 1/2 and 3/4 of the threshold just cleared and weighted 6 / 2.5
/ 1 so mythics are hunted rather than tripped over.

### Built to scale to hundreds
A relic is a bag of named modifiers, and every rule in the game reads
those through `relic_mult()`, `relic_add()` or `relic_flag()`. Adding
the hundredth relic is a dictionary entry and no new code, provided it
only uses keys that already exist:

    "gilded_cup": {"name": ..., "tier": "rare", "icon": 9,
                   "desc": ..., "mods": {"jackpot_add": 0.3}}

16 modifier keys are wired in: six/one value, jackpot factor, steps and
multiplier, rolls, hand size, bag cap, shop offers, die price, reroll
cost, threshold, snake payout, boss bonus, and the face odds. Only
genuinely structural effects need a flag, and there are five, each one
a branch someone has to maintain.

Icons come from `tools/make_relic_icons.py`, which builds emblems from
a small grammar (tier-coloured plate, ring style, glyph by index) --
hand-drawing one per relic would not survive the plan.

### What the simulation found
Two relics were broken, both instructive:

- **Weighted Bones** at +8%/-3% face odds was 2.14x baseline from a
  *rare* slot, beating most mythics. Shifting the odds on every die at
  once is simply the strongest thing a relic can do; it is now +4%/-2%.
- **Snake's Ascension** -- your "all 6s become 1s" idea -- was a
  0.10x instant loss. Three fixes were needed before it worked:
  every Snake Eyes Die cashes in rather than only the first, those
  payouts jackpot together (without that the build earns a flat amount
  per die, and linear income always loses to a compounding threshold),
  and ordinary ones pay a little so taking it blind is a bad gamble
  rather than a dead run.

It now does what you wanted. With a bag the player keeps concentrated,
it takes the snake build from **round 18.5 to round 30.0**. Let a
greedy buyer dilute that bag with whatever the shop offers and it
collapses to 9.9 -- which is the intended shape: the relic rewards
curating the bag, and the thinning consumables are the tool for it.

Overall, relics roughly double a run: mean round 9.5 without them,
18.3 with, and 42% of runs reach round 30 where none did before.

## Shop presentation
The shop is now a full-bleed panel (1200x680 at base resolution, up
from 1020x560) with each half on its own tinted felt -- violet behind
the relics, green behind the dice -- a gold-bordered frame and a drop
shadow.

**Nothing scrolls.** The old columns were ~470px wide and clipped every
description behind a scrollbar. At ~568px the longest entry in the
catalog wraps to three lines and fits, and the column is tall enough
for a full post-boss shop (4 dice + 2 relics) without overflow.

**Two reroll buttons**, one per side. They track separately, so
spinning for a relic no longer inflates the price of rerolling dice,
and each sits under the column it affects.

**Tax Haven** now frees only the *first* reroll of each side per shop.
Free rerolls outright let a player respin both halves until the offers
were perfect, every shop, for the rest of the run -- it trivialised the
economy within a couple of rounds. One free spin per side, then the
usual 1.75x curve.

The dice bag icon is gone from the title screen, which now leads with
the dealer and the starfield.

## Scoring readout and relic shelf
Two panels now flank the tray, so the run's state is legible without
opening anything.

**Left -- the scoring walk.** After each roll it steps through the hand
one die at a time, showing what each contributed and building a running
total, then lands the jackpot line last. The point isn't the final
number -- the HUD already shows that -- it's watching a middling hand
turn into a big one when the multiplier hits, which is how a player
feels a build getting stronger rather than being told.

This needed the roll pipeline to report more than totals. `roll_resolved`
now carries a `steps` array built *as the hand is tallied*, because only
that loop knows which branch each die took: a Loaded Die paying flat, a
Gilded Die that missed its 3-six requirement, a Glass Die shattering,
Snake Eyes cashing in. Reconstructing it afterwards would mean
duplicating every rule. Entries that moved no money are skipped, and
the walk abandons itself if another roll starts mid-animation.

**Right -- the relic shelf.** Everything you own in a 4-wide grid at
50% opacity, the way Binding of Isaac keeps items on screen. Hovering
brings an icon to full opacity and names it, with the full description
in the tooltip -- glanceable at rest, readable on demand. It's the
answer to relic names saying nothing about their effects.

Both panels are anchored to the screen edges and clear the centred tray
at base resolution (tray x330-950, readout x18-268, shelf x1052-1262).

## The climb
A run no longer goes until you bust -- it has a summit. Beat the boss
at the top of your current tier and the run **ends as a win**, and the
summit moves ten rounds further out for next time. Tier 1 ends at
round 10; tier 10 ends at round 100 and there is nothing above it.
`boss_tier` is saved per profile, and the HUD shows "Round 7 / 30" so
the target is always visible.

**Ten bosses for ten boss rounds**, and a climb never meets the same
one twice -- `bosses_faced` tracks the run, so a full ascent to 100
faces each exactly once. Four new ones were added to make that possible:

| Boss | Effect |
|---|---|
| The Pit | Draw only 3 dice -- but the jackpot multiplier is tripled |
| The Ledger | Target is half again as high -- but 15 extra rolls |
| The Famine | Sixes pay a quarter -- but every 1 pays you instead |
| The Gauntlet | Ones cost five times as much -- but the jackpot starts two steps higher |

## Difficulty curve
Threshold growth is steeper: **1.10 to 1.125** per round.

Worth knowing why not steeper. Because the climb now ends at a fixed
target, the curve can be measured against it rather than guessed. Peak
income is bounded -- the bag caps, the hand caps, so a near-perfect
build tops out around $2.07M in a round. Against that:

| growth | round 100 asks | share of a peak build |
|---|---|---|
| 1.10 (old) | $150,334 | 7% |
| **1.125 (now)** | **$1,390,852** | **67%** |
| 1.13 | $2,157,482 | 104% -- impossible |

At 1.125 the summit needs about two thirds of a theoretical perfect
run: demanding, but not luck-locked. Anything past ~1.13 makes round
100 arithmetically unreachable no matter how the dice fall.

## Round-end pacing
The shop, the bust screen and the victory screen all now wait for the
scoring walk to finish counting, then a further **2.5 seconds**, so a
final total is never snatched away mid-count. All three route through
one `_pending_end` state rather than three separate timers.

## Still placeholder
- No audio, no animation, no juice yet (M3 in the roadmap).
- No save file. Achievements and the unlocked shop pool persist across
  runs within a session but reset when you close the editor.
- Only the 6 seed dice from the roadmap, not the full 15-30 pool.
