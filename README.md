# gbs-DynamicProjectilePlugin

**Version 4.3.0. Requires GB Studio 4.3.0 or newer.**

Replaces GB Studio projectiles, which travel in straight lines, with ones that can do a great deal more.

A shot can arc like a thrown rock, weave through the air, come back like a boomerang, orbit a character as a shield, act as a grappling hook that pulls the player across a gap, hang off an actor, be steered frame by frame from variables, string a chain of sprites between two actors, or leave a trail behind it. Every projectile also gains collision with walls, bouncing, scripts that run on impact, and a set of parameters you choose per launch.

This is a port of [fredrikofstad's CustomProjectile plugin](https://github.com/fredrikofstad/GBStudioPlugins) (originally written for GB Studio 4.1.x), rebuilt for the reworked 4.3.0 engine and extended well beyond it.

The plugin ships with `DynamicProjectileExample/`, a complete project with one demo scene per behaviour.

---

## Table of Contents

1. [Concepts](#concepts)
2. [Project Setup](#project-setup)
3. [Size Limits and Restrictions](#size-limits-and-restrictions)
4. [Behaviours Reference](#behaviours-reference)
5. [Engine Settings](#engine-settings)
6. [Tile Collision Modes](#tile-collision-modes)
7. [Events Reference](#events-reference)
8. [Engine Fields Reference](#engine-fields-reference)
9. [FAQ](#faq)
10. [Media](#media)
11. [Memory Footprint](#memory-footprint)
12. [Credits](#credits)
13. [Bank 0 (HOME) Usage](#bank-0-home-usage)
14. [Changelog](#changelog)

---

## Concepts

### Slots and Launches

A projectile is described by a **slot**, an entry in a shared table holding its sprite, speed, lifetime, collision groups and behaviour. Slots are what you launch from, and a single slot can be launched any number of times. The two core events follow that split: **Load Dynamic Projectile Into Slot** writes a slot, and **Launch Dynamic Projectile From Slot** fires one.

GB Studio works the same way with its own **Load Projectile Into Slot** and **Launch Projectile In Slot** events, and this plugin's events build on the same foundation, so the two sets work together. A slot written by one can be launched by the other.

> **Order matters if you mix them.** *Load Dynamic Projectile Into Slot* writes the whole definition, so running the stock *Load Projectile Into Slot* afterwards on the same slot wipes the behaviour back to Default.

### What Lives Where

A slot is a fixed 32 bytes, which is not enough room for everything, so configuration is split across the two events:

| Lives on | Holds | Applies to |
|----------|-------|------------|
| **Load Dynamic Projectile Into Slot** | Sprite, speed, lifetime, collision groups, behaviour, and the behaviour's shape (gravity, bounce, wave size, chain slack, trail length…) | Every projectile launched from that slot |
| **Launch Dynamic Projectile From Slot** | Source, aim, and the handful of values that vary shot to shot (arc height, orbit target, chain endpoints, trail head…) | That one shot |

Because a live projectile stores only an *index* into the slot table, redefining a slot changes the behaviour of anything already in flight from it. Only the lifetime and the per-launch parameters are truly per-projectile.

### The Projectile Pool

All live projectiles come from a fixed pool, sized by the **Max concurrent projectiles** setting (default 6). A launch that finds the pool full is silently dropped. Bear in mind how many entries a shot really uses: most behaviours take one, but a **hookshot takes four**, the head plus three chain links.

### Strands

**Chain** and **Trail** are the two behaviours that draw a *run* of sprites from a single projectile. Those sprite positions cannot be worked out from the projectile's own position, so they are kept in a **strand**, a small buffer from a second pool sized by **Max chains + trails** and **Points per chain / trail**.

A chain or trail launched when every strand is busy is silently not launched, exactly as it would be if the projectile pool were full.

---

## Project Setup

### Installation

Copy `src/DynamicProjectilePlugin` into your project's `plugins/` folder, keeping the whole folder intact.

This plugin replaces the stock projectile engine files. Three other plugins in the same collection touch some of the same files: **SceneStackExPlugin**, **DynamicActorPlugin** and **SpritesheetChangeBufferPlugin**. Compatibility variants ship for every combination of them and are selected automatically. Because this plugin is applied last of the four, its merged build also resolves the clash between DynamicActorPlugin and SpritesheetChangeBufferPlugin.


### 1. Define a Slot

Add **Load Dynamic Projectile Into Slot** wherever the projectile should be set up, usually a scene's **On Init**, since a slot lasts until something overwrites it.

The *Projectile* tab is the stock projectile definition: sprite sheet, animation state, speed, animation speed, life time, initial offset, collision group and collide-with mask.

The *Dynamic projectile* tab is this plugin's behaviour. Start from a **Preset** and you are done in one dropdown:

| Preset | Behaviour |
|--------|-----------|
| Bullet | Straight, dies on walls |
| Lobbed shot | Arcs under gravity |
| Grenade | Gravity, reflects off walls and floors |
| Boomerang | Slows, reverses, returns |
| Wave | Weaves as it travels |
| Orbiter | Circles an actor |
| Grapple | Hookshot head or chain link |
| Held weapon | Anchored to an actor |
| Script driven | Delta from two variables |
| Tether | Taut chain between two actors |
| Leash | Chain that hangs and drags |
| Fire trail | Hops, leaving a tail behind |

Choosing **Custom (choose components)** instead reveals the individual fields, and only the ones the chosen behaviour actually reads.

### 2. Launch From the Slot

Add **Launch Dynamic Projectile From Slot**. The *Source* tab picks where the shot starts and where it aims; the *Dynamic projectile* tab carries the per-launch parameters.

Set **Slot Behaviour** on that tab to whatever the slot holds. It has no effect on the projectile and only decides which fields are shown, since the launch event cannot know what the slot contains.

### 3. Add Impact Scripts (optional)

**Set Projectile Removal Script**, **Set Projectile Tile Hit Script**, **Set Projectile Actor Hit Script** and **Set Projectile Tile Enter Script** register a script that runs when *any* projectile is removed, hits a wall, touches an actor, or crosses into a new tile. Set them once per scene; they persist until cleared or the scene changes.

Each slot has a matching checkbox (*Run On Remove script* and friends) so an individual slot can opt out.

**Set Projectile Tile Hit Script** holds one script per side of the tile. Run it once per side, or leave it on **Any** to fill all four at once.

### 4. Trim the Behaviours You Do Not Use (optional)

Every behaviour except Default has an on/off switch under **Settings → Engine → Dynamic Projectiles**. These take effect at build time, so turning one off removes its code from the ROM rather than skipping it while the game runs. The behaviours are by far the biggest lever on the plugin's ROM cost.

The three shared impact scripts have their own switches, **Enable script: Tile Hit**, **Actor Hit** and **Tile Enter**, which remove them from the ROM entirely. **Tile hit script per face** sits under the first of those and decides whether the tile hit trigger keeps four scripts or one.

The events refuse to compile against a behaviour or script that has been switched off, naming the offending scene, rather than silently producing a projectile that does nothing. The one exception is the per-slot checkboxes: a slot ticking *Run Tile Hit script* while that switch is off simply has the flag cleared, since a field cannot be hidden based on an engine setting and the flag would be dead weight either way. Turning the switch back on restores it without touching the slot.

---

## Size Limits and Restrictions

### The projectile pool can run out

All live projectiles come from a fixed pool, sized by **Max concurrent projectiles**. A launch that finds the pool full is **silently dropped**. Remember how many entries a shot really uses: most behaviours take one, but a **hookshot takes four**, the head plus three chain links.

### The strand pool can run out

**Chain** and **Trail** each need a strand buffer from a second pool, sized by **Max chains + trails** and **Points per chain / trail**. A chain or trail launched when every strand is busy is silently not launched, exactly like a full projectile pool.

### Redefining a slot changes projectiles already in flight

A live projectile stores only an index into the slot table, so redefining a slot changes the behaviour of anything already flying from it. Only the lifetime and the per-launch parameters are truly per-projectile.

### Order matters if you mix with the stock events

*Load Dynamic Projectile Into Slot* writes the whole definition, so running the stock *Load Projectile Into Slot* afterwards on the same slot wipes the behaviour back to Default.

### The tile enter script is not a per-tile trace

The crossing is detected by comparing the cell the origin point started an update pass in with the one it ended in, so a projectile moving more than a cell per pass reports where it landed, not every cell on the line between. Slow it down, or widen the **Tile enter grid**, if you need every one.

### "Infinite lifetime" moved from a flag to Life Time 0

Older versions had an *Infinite lifetime* checkbox on the definition. That flag bit is now *Run Tile Enter script*, and a projectile never expires by being given a **Life Time of 0** instead. Slots saved with the old checkbox keep their stored *Life Time*, so re-check any that relied on it.

### Stock engine files are replaced

This plugin replaces the stock projectile engine files. Compatibility variants are included for **SceneStackExPlugin**, **DynamicActorPlugin** and **SpritesheetChangeBufferPlugin** and are selected automatically; any other plugin touching the same files needs a manual merge.

---

## Behaviours Reference

Every numeric field on both events is a script **value**, so it accepts a variable or an expression as well as a fixed number.

---

### Default

A straight-line shot. This is the stock behaviour plus this plugin's tile collision, bounce and stop handling, and its impact scripts.

**Slot fields:** Tile Collision Behaviour · Gravity · Tile Collision Override
**Launch parameters:** none

---

### Arc

https://github.com/user-attachments/assets/0d3824fa-a344-497f-a642-c7efe43db31b

Thrown upward, then pulled back down by gravity. Gravity is applied every other frame.

**Slot fields:** Tile Collision Behaviour · Gravity · Tile Collision Override
**Launch parameters:** **Launch Height** sets how high it is thrown before gravity takes over

---

### Boomerang

https://github.com/user-attachments/assets/e3679382-6c15-4327-ba99-926d23725815

Sheds speed as it travels, reverses, and comes back.

**Slot fields:** Tile Collision Behaviour · Gravity · Tile Collision Override
**Launch parameters:** **Range** sets how quickly it sheds speed. Higher values bring it back sooner, so it travels **less** far

---

### Sine Wave

https://github.com/user-attachments/assets/842b81e3-ade3-430f-b2c8-fd8149c46fea

Weaves from side to side, perpendicular to its direction of travel.

**Slot fields:** Tile Collision Behaviour · Gravity · Tile Collision Override · *Wave Amplitude* (how far it weaves) · *Wave Frequency* (how tight the zigzag)
**Launch parameters:** **Starting Phase** sets where in the wave it begins, so shots fired together do not overlap

---

### Orbit

https://github.com/user-attachments/assets/ce1145a0-3f12-457a-a6d1-f4701c3c7cbe



Circles an actor at a fixed radius, following it as it moves.

**Slot fields:** Tile Collision Behaviour · *Orbit Radius* · *Orbit Speed*
**Launch parameters:** *X / Y Offset* (offset from the actor's centre) · *Starting Angle* · *Actor To Circle* (actor index, 0 = player)

Orbiters share one target actor, the one set by the most recent launch. Space several around the same actor by giving each a different starting angle.

---

### Hookshot

https://github.com/user-attachments/assets/e085e38c-01ad-4d5b-97ca-22b3f740b2dc

A grappling hook: a head that flies out, up to three chain links that span the gap behind it, and a set of reactions for what happens when the head lands. See [Hookshot Detail](#hookshot-detail) below.

**Slot fields:** Tile Collision Behaviour (*Pass through* / *React to tiles*) · Gravity · *Hookshot: On Tile Hit* · *Hookshot: On Actor Hit*
**Launch parameters:** *Chain Link* (0 = head, 1–3 = links) · *Anchored To Actor* (the actor it is fired from)

Launch the head and its links as separate shots from the same slot, varying only *Chain Link*.

---

### Anchor

https://github.com/user-attachments/assets/bc1ca11b-af4d-4bd5-be93-95f38bc4698b

Rigidly pinned to an actor at a fixed offset, whichever way that actor turns. Useful for held weapons and carried objects.

**Slot fields:** Tile Collision Behaviour
**Launch parameters:** *X / Y Offset* · *Attached To Actor*

Unlike Orbit, each anchored projectile carries its own target, so several can hang off different actors at once.

---

### Custom

https://github.com/user-attachments/assets/29c73909-2e20-42ce-9271-9ec0c8c0c56a

Moves by whatever two script variables contain, read again every frame. Everything else, meaning collision, lifetime and impact scripts, behaves normally.

**Slot fields:** Tile Collision Behaviour · *Delta X Variable* · *Delta Y Variable*
**Launch parameters:** none

---

### Chain

https://github.com/user-attachments/assets/5d4f7d9b-3c17-4fca-abe8-4f483373bdf8

A run of sprites strung between two actors. A chain has no velocity, no aim and no tile collision; the launch position and angle only decide which of the sprite's four direction animations the links are drawn with.

**Slot fields:** *Chain Type* · *Links* (counting the one on each actor) · *Slack* and *Catch-Up Speed* (loose only)
**Launch parameters:** *Strung From Actor* · *Strung To Actor* (actor indexes, 0 = player)

| Chain Type | Placement |
|------------|-----------|
| **Straight** | Links are spread evenly along the line between the two actors, recomputed every pass. |
| **Loose** | Each link chases its neighbour, but only once the gap between them opens past **Slack**, and then only by **Catch-Up Speed** pixels per update. That tolerance is what makes a chain hang and drag instead of snapping into a line. A Catch-Up Speed of 0 makes it rigid, closing the whole gap in one step. |

A loose chain is seeded straight when launched and only then starts relaxing, so a leash appears taut for a frame and then sags.

Give the slot a **Collide With** mask and every link becomes dangerous along its whole length; leave it empty and the chain is decoration. The two end links are never tested, because they sit on the actors the chain is tied to and would report a hit on its own anchors every pass.

A chain is **not** removed when it scrolls off screen, unlike every other behaviour: both its ends are actors, which the engine keeps alive on or off screen, and a leash cut the moment its midpoint scrolled away would never come back. It ends when its lifetime runs out.

---

### Trail

https://github.com/user-attachments/assets/63cda598-eb4a-4046-9ae4-25615ea18b8c

Travels like a plain shot, with speed, gravity, bounce, stop and tile collision all behaving normally, and also records where it has been, hanging a sprite off its own history every few updates.

**Slot fields:** Tile Collision Behaviour · Gravity · Tile Collision Override · *Trail Segments* · *Trail Spacing*
**Launch parameters:** *Trail Head* (this projectile / an actor) · *Head Actor*

The tail is tested for actor collisions along with the head, so the whole trail is dangerous rather than just the sprite leading it. Widening the spacing stretches the tail further behind at no extra sprite cost, but it costs strand points: a trail needs `segments × spacing` of them.

Spacing starts at 2 rather than 1, because the newest entry in the history is where the projectile itself is drawn, so a spacing of 1 would stack the first segment on top of the head.

**Hanging a trail off an actor.** Set *Trail Head* to *An actor* and the projectile stops being a shot and becomes pure tail: it records where that actor has been and draws segments along it. This is how to give a trail to something that already moves on its own: a walking NPC, a platform, the player. With a head actor the projectile hands everything positional over to that actor, so it is not drawn, does not move, ignores tile collision, is not removed off screen, and never reports a hit on its own head.

Successive chain links and tail segments step on through the slot's animation, wrapping at its end. With a one-frame animation they all look alike; with a two-frame one they alternate.

---

### Hookshot Detail

The hookshot decides for itself what to do when its head lands, so a working grapple needs no scripting.

| Field | Options |
|-------|---------|
| *Hookshot: On Tile Hit* (needs *Tile Collision Behaviour* = React to tiles) | **Return** · Pull Player · Remove Hook · Stay Where It Hit |
| *Hookshot: On Actor Hit* | **Remove Hook** · Return · Pull Player · Pull Actor · Stay Where It Hit · Stick To Actor |

- **Return** turns the head around and brings it back.
- **Pull Player** drags the firing actor over to where the head landed.
- **Remove Hook** makes the hook and its chain disappear on impact.
- **Stay Where It Hit** parks the head at the impact point, with the chain still spanning the gap. Recall it later by setting the hookshot state to Returning.
- **Pull Actor** reels the actor that was hit back in. The plugin picks up whichever actor it touched, so nothing has to be known in advance.
- **Stick To Actor** latches the head onto the actor it hit and rides along with it, chain and all, keeping the offset it made contact at. A hook that catches an enemy's shoulder stays on the shoulder.

Set *On Tile Hit* to **Pull Player** and *On Actor Hit* to **Pull Actor** for the usual grappling-hook behaviour: walls pull you to them, enemies get pulled to you. The defaults (Return / Remove Hook) preserve the original plugin's behaviour.

**Pull Player** and **Stay Where It Hit** leave the hook in play until something ends it, so pair them with a recall or with firing again, since a new shot clears the old one.

Only the head reacts: chain links never trigger tile or actor collisions and never end the shot by scrolling off screen, and a hookshot stops looking for actors once it lands, so a target's hit script fires once rather than every frame of a pull.

**The chain follows its source.** The links are placed each frame at 3/4, 1/2 and 1/4 of the way along the line from the *Anchored To Actor* actor to the head, so walking around with a hook parked or stuck keeps the chain spanning the gap. That parameter is an actor index, so pointing it at any actor gives an NPC a hookshot.

**Pulls stop at obstacles.** A pull moves an actor by writing its position directly, which nothing else in the engine gets a say in, so on its own it would reel somebody into a wall and leave them standing inside it. With **Hookshot pulls stop at obstacles** on (the default), each step is tested before it is committed and the pull ends the moment it is blocked. **Pull Player** stops the firing actor short of a solid tile or actor. **Pull Actor** stops the reeled-in actor short of the same, including the firing actor itself, so a catch is never dragged inside the player. Testing is per axis, so a pull dragged along a wall slides down it. An actor counts as solid when it has collision enabled, and only the edge being moved towards is tested, so one-way tiles still block just the side they face.

**Pulls put the player back on the grid.** A pull moves in fractions of a pixel, so it almost never stops exactly on a grid point, and a top-down scene only reads input while the player is on one, so an actor left between two can end up unable to move at all. Every pull therefore realigns the source actor before it lets go, to the nearest point on the grid chosen by **Hookshot pull realigns to grid**. If the aligned spot turns out to be inside something, the actor gives that cell back, and failing that stays where the pull stopped it. Off grid is better than inside a wall.

---

## Engine Settings

Found under **Settings → Engine → Dynamic Projectiles**. Indented entries belong to the behaviour above them and only appear while it is switched on.

| Setting | Range | Default | Description |
|---------|-------|---------|-------------|
| **Max concurrent projectiles** | 1–20 | 6 | Size of the live projectile pool. A launch that finds it full is dropped. Each entry costs 16 bytes of WRAM. |
| **Max projectile slots** | 5 to 20 | 5 | How many definitions are held. Five is the minimum, because GB Studio always fills the first five itself. Each slot costs 32 bytes of memory. |
| **Off-screen margin (tiles)** | 0–10 | 2 | How far past the screen edge a projectile may travel before it is retired. 0 retires it the moment its origin leaves the screen. |
| **Tile collision detection** | Origin point / Bounding box | Origin point | Whether tile tests use a single lookup at the projectile's position, or every tile its bounds rect covers. Bounding box makes large projectiles stop as soon as any part of them touches a wall, at the cost of several tile reads per projectile per frame. |
| **Enable tile collision override** | on or off | **off** | Builds the per-slot **Tile Collision Override**. Off, every tile test compiles as it did without the feature and the field is ignored. See [Tile collision override](#tile-collision-override). |
| **Enable script: Tile Hit** | on / off | on | Compile-time switch for the shared tile hit trigger. Off removes its global, its native and its trigger from the ROM. |
| ↳ **Tile hit script per face** | on / off | on | Whether the tile hit trigger keeps one script per face of the tile, or a single shared one. Off collapses the four slots to one and folds away the work of deciding which face was struck; the event's *Tile Face* must then stay on *Any*. |
| **Enable script: Actor Hit** | on / off | on | Same for the actor hit trigger. |
| **Enable script: Tile Enter** | on / off | on | Same for the tile enter trigger, including the per-pass cell comparison in the update loop. |
| ↳ **Tile enter grid** | 8x8 / 16x16 tiles | 8x8 | Cell size the *Set Projectile Tile Enter Script* counts crossings against. 16x16 fires a quarter as often, which suits metatile-based scenes. |
| **Enable behaviour: …** | on / off | on | One compile-time switch per behaviour except Default. |
| ↳ **Hookshot pull realigns to grid** | Off / 8px / 16px | 8px | Grid a pull puts the source actor back on. Set it to match your top-down scene, or *Off* for scene types where snapping the player is wrong. |
| ↳ **Hookshot pulls stop at obstacles** | on / off | on | Whether a pull stops short of solid tiles and actors. |
| ↳ **Max chains + trails** | 1–8 | 2 | Strands available to Chain and Trail. Shown while either behaviour is on. |
| ↳ **Points per chain / trail** | 2–32 | 16 | Points in one strand. A chain needs one per link; a trail needs `segments × spacing`. |

The grid setting cannot be read from the scene itself: the top-down grid size lives in a file GB Studio does not compile into a project with no top-down scene, so reading it would fail to link in a platformer.

Collision spreading is the **stock** *Collision checks* field under **Settings → Engine → Projectiles**, not one of this plugin's. The plugin honours it, so all three values work as they do for stock projectiles.

---

## Tile Collision Modes

What a projectile does when it reaches a solid tile, set per slot by **Tile
Collision Behaviour**:

| Mode | What happens |
|---|---|
| **Pass through** | No tile tests at all. The projectile flies until its lifetime runs out or it leaves the screen. |
| **Remove projectile** | The first solid tile removes it, running the removal and tile hit scripts. One lookup for any solid side, so it works for every behaviour including the ones that are placed rather than moved. |
| **Bounce (perfect reflect)** | The struck axis reverses exactly, with no rebound strength and no damping. The other axis keeps running, so a shot skimming a floor keeps its forward speed. |
| **Stop on impact** | Both axes drop to zero. The projectile halts where it struck and lives out its lifetime there, still animating. Under gravity it settles onto the surface instead of resting on one frame and falling the next. |

Bounce and Stop both act on the velocity, so they only apply to the behaviours
that travel by it (Default, Arc, Boomerang, Sine, Trail, and a Hookshot head).
Orbit, Held and Scripted are placed by other means and never fill in a velocity;
for those, **Remove projectile** is the mode that reacts to tiles.

### Tile collision override

Off by default. Tick **Enable tile collision override** in the engine settings to
use it. While it is off, every tile test a projectile makes compiles exactly as it
did before the feature existed, and setting a slot's override to a non-zero constant is
reported as a compile error naming the setting rather than silently doing nothing.

Each tile test asks whether the tile carries the collision bit for the side being
entered. **Tile Collision Override** replaces that mask outright when it is non-zero, so
one slot can react to a fixed set of tile bits regardless of which direction it is
testing. **0 is normal collision**, where the side being approached is what gets
projectile is approaching from. The bits are top `1`, bottom `2`, left `4`, right `8`. Above those, `16`-`112` is a value
range: `16` is a ladder and `32` to `112` are the six slopes. `128` is a value no engine code
reads at all.

**Property bits (16 and above)** make any tile carrying them solid from every
direction at once. Setting `16` makes ladder tiles solid for that slot alone, giving a
grapple that only catches tiles you marked, a shot stopped by a window the player
walks past.

**Direction bits (1-8)** replace the direction being tested rather than adding to
it: setting `1` makes the projectile react only to a tile's floor bit, from
whichever side it touches the tile, which suits a shot that should only
ever be stopped by ground, never by a wall or ceiling.

No override value can make an *empty* tile solid: a tile with no bits set matches no
mask. Blank space stays passable.

A **Chain** never tests tiles, so its slot borrows this same definition byte for
**Catch-Up Speed**, the same trick it already plays with **Tile Collision Behaviour**,
which it reuses as *Chain Type*. Nothing to set: the event shows whichever of the
two the behaviour actually uses.

### Painting the extra collision values

A tile collision override is only useful if the tile bits it tests can be painted, and the top half of
the collision byte is mostly unreachable in the stock editor: only the Platformer palette
offers anything above `0x0F`, and `0x80` could not be painted anywhere.

This plugin therefore declares the collision tile palettes itself, adding the bits each
scene type leaves spare as **Extra** swatches, one per possible value:

| Scene type | Spare bits | Extra swatches added |
|---|---|---|
| **Top Down** | `0x10 0x20 0x40 0x80` | 15: Extra 16 … 240 |
| **Shmup** | `0x10 0x20 0x40 0x80` | the same 15 |
| **Point and Click** | `0x10 0x20 0x40 0x80` | the same 15, plus the four direction bits |
| **Platformer** | 128 | 1: Extra 128. 16 to 112 is the ladder and slope range |
| **Adventure** | `0x20 0x40 0x80` | 7: Extra 32, 64, 96, 128, 160, 192, 224 |
| **Logo** | none | nothing. No actors, no projectiles, nothing reads a collision tile |

Ladder and the six slopes stay Platformer-only: they are a value space `platform.c` reads,
not free bits. Each Extra swatch paints one exact value over the whole spare-bit space, so
*Extra 48* is `0x10` **and** `0x20` in a single click.

**CollisionExPlugin declares the same palettes**, and so does DynamicActorPlugin. That is deliberate
and safe: GB Studio applies a plugin's scene type override as a shallow replacement, so
whichever loads last decides the swatches, and all three are kept identical so
the result is the same in any combination. It also means this plugin needs none of the
others installed to have its own override values paintable. If you edit one, edit all three.

For the full reference, including the collision value layout and why a tile carrying both
a direction and an Extra value highlights only the direction, see
[CollisionExPlugin](https://github.com/gb-studio-dev/gb-studio-plugins/tree/main/plugins/Mico27/CollisionExPlugin#painting-the-extra-collision-values).

---

## Events Reference

All events appear under the **Projectiles** group in the script editor.

---

### Load Dynamic Projectile Into Slot

Shown in scripts as *Load Dynamic Projectile Into Slot N : &lt;preset&gt;*.

Writes a complete projectile definition into a slot: everything the stock *Load Projectile Into Slot* event writes, plus this plugin's behaviour. Both halves are written in one event because the order matters. The stock half copies a whole definition into the slot and would wipe the behaviour if it ran second.

**Projectile tab**

| Field | Description |
|-------|-------------|
| Projectile Slot | Slot to write (0 to *Max projectile slots* - 1). A plain number, not one of five buttons. |
| Sprite Sheet / Animation State | The projectile's appearance. |
| Speed / Animation Speed | Travel speed and animation rate. |
| Life Time | Seconds before it expires on its own. **0 means it never does**, so it then only goes away by hitting something, leaving the screen, or a script removing it. |
| Initial Offset | Distance in front of the source to spawn at, in pixels. |
| Loop Animation | Whether the animation repeats or holds on its last frame. |
| Destroy On Hit | Unchecked makes it a *strong* projectile that survives contact. |
| Collision Group / Collide With | Standard GB Studio collision groups. |

**Dynamic projectile tab**

| Field | Description |
|-------|-------------|
| Preset | Ready-made behaviour, or *Custom* to choose components. |
| Behaviour | Which behaviour this slot holds (Custom only). |
| Tile Collision Behaviour | Pass through · Remove projectile · Bounce (perfect reflect) · Stop on impact. |
| Gravity | Downward pull, applied every other frame. Clamped to -8…7. |
| Tile Collision Override | Advanced. Changes what counts as solid for this slot. See [Tile collision override](#tile-collision-override). Needs the engine setting of the same name. |
| *(behaviour shape fields)* | Wave amplitude and frequency, orbit radius and speed, chain type, links, slack and catch-up, trail segments and spacing, hookshot reactions, and the custom movement variables. See [Behaviours Reference](#behaviours-reference). |
| Run Tile Enter script | Whether this slot runs the shared tile enter script. Off by default, since it fires far more often than the impact scripts. |
| Ignore player collision | Pass through the player. |
| Run On Remove / Tile Hit / Actor Hit script | Whether this slot triggers each shared impact script. All on by default. |

---

### Launch Dynamic Projectile From Slot

Fires a projectile from a slot. Same as the stock launch event, but the slot is a plain number so it reaches the extra slots, and it adds a source-from-position mode, a position-target aim mode, and the per-launch parameters.

**Source tab**

| Field | Description |
|-------|-------------|
| Projectile Slot | Slot to launch from. Must be below *Max projectile slots*, and something must have been defined in it first. |
| Launch From | **Actor** (with a pixel *Offset X / Y*) or **Position** (a fixed point, no actor involved). |
| X / Y | Position source only. Values with a tiles / pixels unit toggle, so a random spawn column can be an expression like `16 + rnd(120)` in pixels. |
| Launch At | *Fixed Direction* · *Angle* · *Angle Variable* · *Actor Direction* · *Actor Target* · *Position Target*. The actor-relative options need an actor source. |
| Start Frame | How many frames into the animation this shot starts, counted from the first frame of whichever direction it faces. 0 starts at the beginning. Nothing checks the range, so keep it inside the animation's length or the shot starts on whatever frame follows. |

*Position Target* computes the angle at runtime the same way the stock *Actor Target* does, so a turret can aim at a fixed point in the room. With a **Position** source you get a shot fired from one coordinate towards another with no actors involved at either end.

**Dynamic projectile tab**

| Field | Description |
|-------|-------------|
| Slot Behaviour | Display only. It decides which parameters below are shown, and the real behaviour comes from the slot. |
| *(per-launch parameters)* | See each behaviour in [Behaviours Reference](#behaviours-reference). |

---

### Launch Dynamic Projectile From Slot By Index

The same, with every actor given as a number rather than picked from the scene: the firing actor, the **Actor Direction** actor, the **Actor Target** actor, Anchor's **Attached To Actor** and Trail's **Head Actor**. Use it when the actor has to come from a variable.

---

### Set Hookshot State

Drives the active hookshot's state machine by hand, whatever its impact reactions are set to.

| Field | Description |
|-------|-------------|
| Set Hookshot State | Firing · Returning · Pull Player · Pull Actor · Remove. |
| Actor to pull | The actor a *Pull Actor* state reels in. |

*Returning* recalls a hook that is parked or stuck; *Remove* clears the whole hookshot, which is the usual way to tidy up before firing a new one.

---

### Set Hookshot State By Index

As above, with the actor given as an index script value.

---

### Set Projectile Removal Script

Registers a script that runs whenever any projectile is removed, whether it expired, left the screen, or was destroyed on impact.

| Field | Description |
|-------|-------------|
| Action | Set script · Clear script. |
| On Removal | The script to run. |

---

### Set Projectile Tile Hit Script

Registers a script that runs whenever a projectile's *Tile Collision Behaviour* reacts to a solid tile. A projectile in the Bounce mode runs it on **every** bounce, so keep it short.

There is **one script per face of the tile**, so a shot landing on a floor can do something different from one hitting a wall. Run the event once per face you care about; each slot is independent and holds until cleared or the scene changes.

| Field | Description |
|-------|-------------|
| Action | Set script · Clear script. |
| Tile Face | Any · Top · Right · Bottom · Left. *Any* writes all four slots at once, which is the default. Naming a single face needs the **Tile hit script per face** engine setting; with that off there is one shared script and the event refuses anything but *Any*. |
| On Tile Hit | The script to run. |

**Face means the side of the tile that was struck**, not the way the projectile was travelling. A shot falling onto a floor hits its **Top**, and one flying right into a wall hits that wall's **Left**. Bounce and Stop on impact already test each face separately, so they report it exactly. *Remove projectile* mode does a single lookup for any solid side, so there the face is read back off the direction of travel.

The per-slot **Run Tile Hit script** tickbox is still one switch covering all four faces, because there is no room in the definition for four.

---

### Set Projectile Actor Hit Script

Registers a script that runs whenever a projectile touches an actor in its collision mask. A *strong* projectile runs it for as long as it overlaps.

| Field | Description |
|-------|-------------|
| Action | Set script · Clear script. |
| On Actor Hit | The script to run. |

---

### Set Projectile Tile Enter Script

Registers a script that runs whenever a projectile's origin point crosses into a new cell of the **Tile enter grid**. Only slots with **Run Tile Enter script** ticked trigger it, since it fires far more often than the impact scripts. A shot crossing the screen runs it once per tile.

The crossing is detected by comparing the cell the origin point started the pass in with the one it ended in, so a projectile fast enough to jump a whole tile reports where it landed rather than every tile on the line between. A shot that leaves the screen still reports the tile it left through, before it is retired.

| Field | Description |
|-------|-------------|
| Action | Set script · Clear script. |
| On Tile Enter | The script to run. |

---

### Set Projectile Lifetime

Globally suspends lifetime countdown, so nothing expires on its own until it is switched back. Slots given a **Life Time** of 0 are unaffected either way, since they never expire.

| Field | Description |
|-------|-------------|
| Infinite Lifetime (Global) | Default · Infinite Lifetime. |

---

### Pause Projectiles

Freezes projectile updates. Paused projectiles keep their pool entry and resume where they left off.

| Field | Description |
|-------|-------------|
| Pause All Projectiles | Off · Pause All Projectiles · Pause on Locked Script. |

*Pause on Locked Script* freezes them automatically whenever a script holds the VM lock, which is the usual way to stop projectiles moving during dialogue.

---

### Show All Projectiles / Hide All Projectiles

Stops or resumes drawing every projectile. They keep moving while hidden, since this only affects what is drawn. No fields.

---

## Engine Fields Reference

These runtime fields are written by the engine just before an impact script runs. Read them with the stock **Engine Field Store** event; they do not appear on the Settings screen, since setting them by hand achieves nothing.

| Field | Description |
|-------|-------------|
| `Last Hit: X` / `Last Hit: Y` | Where the projectile was, in pixels. Refreshed on every removal whether or not a script is attached. |
| `Last Hit: Behaviour` | Which behaviour the projectile was using (Default … Trail). |
| `Last Hit: Actor` | The actor that was hit. Set by the **actor hit** script only, and stale for the other two. |

The remaining fields are the working state the events drive: the per-launch parameters, the hookshot state and the actor that was resolved. They can be read and written with **Engine Field Store** and **Engine Field Update**, but every launch overwrites them.

---

## FAQ

**How do I make a hookshot like Zelda's?**
Load a slot with the **Hookshot** behaviour, choose what happens on impact, such as **Pull Player**
to swing across a gap or **Pull Actor** to reel an enemy in, and launch it. The chain, the return
and the pull are all handled for you.

**How do I make a boomerang?**
Use the **Boomerang** behaviour. **Range** decides how quickly it turns round, and a higher value
brings it back sooner, so it travels less far.

**How do I make a bullet that weaves as it flies?**
Use the **Wave** behaviour and set its amplitude and frequency. **Starting Phase** staggers shots
fired together so they do not overlap.

**How do I make something orbit a character, like a shield spinner?**
Use the **Orbit** behaviour with the character as the target. Space several out by giving each a
different starting angle.

**How do I give a moving actor a trail?**
Use the **Trail** behaviour and set **Trail Head** to **An actor**. The projectile then draws only
the tail, following wherever that actor goes.

**My projectiles stopped appearing after a while.**
The pool is full. Raise **Max concurrent projectiles**, shorten the **Life Time**, or remember that
a hookshot takes four entries rather than one.

**How do I run a script when a shot hits a wall or an enemy?**
Use **Set Projectile Tile Hit Script** or the actor hit script event, and tick the matching **Run**
box on the slot.

**Can a projectile bounce off walls?**
Yes. Set the slot's tile collision behaviour to bounce, and choose between a damped rebound and a
perfect reflection.

**Can I use these alongside the stock projectile events?**
Yes. Both write the same slots, so a slot loaded by one can be launched by the other.

**Where do I put the Load event?**
Usually the scene's **On Init**. A slot lasts until something overwrites it, so it only needs
loading once.

**How do I fire at a fixed point rather than an actor?**
Use a **Position Target**. Combined with a **Position** source you get a shot from one coordinate
to another with no actors involved at all.

**My pull dragged the player into a wall.**
Turn on **Hookshot pulls stop at obstacles**, which is the default. Each step is tested before it
is taken and the pull stops when blocked.

**How do I keep the ROM cost down?**
Turn off the behaviours you do not use. They dominate the total, and Hookshot alone is 2,897
bytes.

**Does it work with the DynamicActor, SceneStackEx or SpritesheetChangeBuffer plugins?**
Yes, in any combination. Compatibility variants ship for all of them and are selected
automatically.

---

## Media

`DynamicProjectileExample/` is a complete GB Studio project demonstrating every behaviour and every event. Open it in GB Studio 4.3+, or build it:

```bash
gb-studio-cli make:rom DynamicProjectileExample/DynamicProjectileExample.gbsproj DynamicProjectileExample/build/DynamicProjectileExample.gb
```

It boots into a menu; each entry is a self-contained demo scene. In every demo **B** fires and **START** returns to the menu.

| Scene | Shows | Controls |
|-------|-------|----------|
| 1 - Arc | Launch height and gravity, *Remove projectile* tile collision, and the hit scripts on a non-hookshot projectile | B: lob a shot |
| 2 - Boomerang | Range, and a Life Time of 0 so the throw only ends by returning | B: light throw, A: heavy throw |
| 3 - Sine Wave | Amplitude / frequency / phase, with A's amplitude read from a variable | B: gentle wave, A: wider each press |
| 4 - Orbit | Three orbiters evenly spaced by starting phase; the third is fired with the By Index launch event | B: add 3 orbiters |
| 5 - Hookshot | Head + three chain links, all four impact reactions, and the chain tracking the player as they walk | B: fire hook, A: pick mode, SELECT: recall |
| 6 - Anchor | Two anchored projectiles on different actors at once | B: attach to player, A: attach to the target |
| 7 - Custom | Per-frame delta driven by two variables, plus an on-removal script | B: fire |
| 8 - Gravity & Bounce | *Bounce (perfect reflect)*, *Stop on impact* and *Remove projectile*, with a separate tile hit script per tile face so the print names the side struck | B / A / SELECT: one behaviour each |
| 9 - On Removal | A removal script reading position and behaviour back out of the `Last Hit` fields | B: fire |
| 10 - Global Controls | Pause / hide / lifetime switches, plus per-slot *Ignore player collision* | B: fire, A: options menu |
| 11 - Extra Slots | Slots 5–7, past what stock GB Studio can address, plus all four launch sources | B / A / SELECT, and the d-pad |
| 12 - Chain | Both placement modes strung from the player to the target. Walk around to see the difference | B: straight chain, A: loose chain |
| 13 - Trail | A trail that hops under gravity, one that flies straight at the target with a longer tail, and a tail hung off the player | B / A / SELECT |
| 14 - Tile Enter | A tile enter script counting cells as a slow shot crosses the room, and an identical slot with the opt-in unticked | B: reporting shot, A: silent shot |

Notes on the demos:

- The walled arena has solid borders and four pillars so the tile-collision and bounce behaviours have something to react to.
- The **Target** dummy (actor index 1) is in collision group 1, so shots using the default collision mask are destroyed by it and print `HIT!`.
- Scene 10 resets the global pause / hide / lifetime switches on the way out, since those engine fields persist across scene changes.
- One hookshot uses four of the six default pool entries, so the hookshot demo clears any hook still out before firing a new one.
- Scenes 12 and 13 use a 4 second lifetime rather than an infinite one: the strand pool holds two, so a third chain or trail is dropped until one expires.

---

<!-- SETTINGCOST:BEGIN -->
### What each engine setting costs

Each setting changes what gets compiled. Figures are what you **get back by turning
the setting off**. Rows marked *off by default* show what turning it **on** costs, and
sliders show the cost per step. "none" means that budget does not move.

| Setting | Bank 0 | WRAM | Banked ROM |
|---|---|---|---|
| Max concurrent projectiles *(slider 1–20, default 6)* | none | 16 B/step | none |
| Max projectile slots *(slider 5–20, default 5)* | none | 32 B/step | none |
| Off-screen margin (tiles) *(slider 0–10, default 2)* | none | none | 1.2 B/step |
| Tile collision detection → *Bounding box* | none | none | +271 B |
| Enable tile collision override *(off by default, so this is the cost of turning it on)* | none | none | +112 B |
| Enable script: Tile Hit | none | **12 B** | **12 B** |
| Tile hit script per face | none | **9 B** | **204 B** |
| Enable script: Actor Hit | none | **3 B** | **125 B** |
| Enable script: Tile Enter | none | **3 B** | **7 B** |
| Tile enter grid → *16x16 tiles* | none | none | +8 B |
| Enable behaviour: Arc | none | none | **23 B** |
| Enable behaviour: Boomerang | none | none | **222 B** |
| Enable behaviour: Sine Wave | none | none | **138 B** |
| Enable behaviour: Orbit | none | none | **187 B** |
| Enable behaviour: Hookshot | none | **6 B** | **2,897 B** |
| Hookshot pull realigns to grid → *Off (leave where it stopped)* | none | none | -304 B |
| Hookshot pull realigns to grid → *16px grid* | none | none | +9 B |
| Hookshot pulls stop at obstacles | none | none | **954 B** |
| Enable behaviour: Anchor | none | none | **183 B** |
| Enable behaviour: Custom | none | none | **104 B** |
| Enable behaviour: Chain | **128 B** | **4 B** | **1,398 B** |
| Enable behaviour: Trail | **201 B** | none | **789 B** |
| Max chains + trails *(slider 1–8, default 2)* | none | 66 B/step | none |
| Points per chain / trail *(slider 2–32, default 16)* | none | 8 B/step | none |

Turning off every on-by-default switch above frees **329 B** of bank 0, **37 B** of WRAM, **7,229 B** of banked ROM. That is the
span between the plugin at its fullest and stripped to nothing, so treat it as a
ceiling. You keep whatever your game actually uses.

- **Max concurrent projectiles**: going from 1 to 20 moves WRAM by +304 B.
- **Max projectile slots**: going from 5 to 20 moves WRAM by +480 B.
- **Off-screen margin (tiles)**: going from 0 to 10 moves banked ROM by +12 B.
- **Max chains + trails**: going from 1 to 8 moves WRAM by +462 B.
- **Points per chain / trail**: going from 2 to 32 moves WRAM by +240 B.

- **Tile hit script per face** only applies when *Enable script: Tile Hit* is enabled.
- **Tile enter grid** only applies when *Enable script: Tile Enter* is enabled.
- **Hookshot pull realigns to grid** only applies when *Enable behaviour: Hookshot* is enabled.
- **Hookshot pulls stop at obstacles** only applies when *Enable behaviour: Hookshot* is enabled.
- **Max chains + trails** only applies when *Enable behaviour: Chain* is enabled.
- **Points per chain / trail** only applies when *Enable behaviour: Chain* is enabled.

<details><summary>How these were measured</summary>

GB Studio 4.3.0-e1. This plugin's engine code was compiled with the toolchain and
flags GB Studio itself uses, and the size of each part of the result was read back and
sorted into the three budgets: the fixed bank 0, work RAM, and switchable ROM banks.

Two caveats. Only this plugin's own engine sources are measured, so a setting that also
changes a shared data structure can move a few more bytes elsewhere. And each setting is
toggled on its own, so a few measure slightly *negative* when enabling their code lets
the compiler drop a fallback path, and a setting that gates other settings shows only
its own contribution.

</details>
<!-- SETTINGCOST:END -->

## Memory Footprint

Measured against the stock GB Studio **4.3.0-e1** engine at default engine settings, report of 2026-08-13. Figures are the difference against a stock project: a file that replaces a stock engine file counts only the change, which is why a plugin can come out negative. Each event you use also compiles a few bytes of script into your project, on top of the fixed cost below.

| Budget | Cost |
|---|---|
| Bank 0 (HOME) | +85 bytes |
| WRAM | +103 bytes |
| Banked ROM | +9,050 bytes |

- **Bank 0:** 85 bytes are resident in the non-switchable bank (`projectiles.c`); everything else lives in a switchable bank. See [Bank 0 (HOME) Usage](#bank-0-home-usage).
- **WRAM:** 103 bytes at the default pool sizes with every behaviour on. The plugin reshapes the stock pool rather than adding to it. GB Studio puts a full 28-byte definition inside every live projectile, 41 bytes per entry, while this plugin's entry is 16 bytes pointing into a shared table. That saving is what pays for its own data and the strand pool.
- **Banked ROM:** 9,050 bytes with every behaviour on. 7 of those land in stock files the plugin does not ship, which compile slightly differently once it is installed. Behaviours dominate the figure, so switching off the ones a game does not use is by far the biggest saving. Hookshot alone is 2,897 bytes. See [What each engine setting costs](#what-each-engine-setting-costs).
- **Tuning:** the pool is 16 B × *Max concurrent projectiles*, the slot table 32 B × *Max projectile slots*, and the strand pool 4 B × *Max chains + trails* × *Points per chain / trail* (66 B and 8 B per step respectively). The strand pool disappears entirely when both Chain and Trail are off, which also returns 128 B and 201 B of bank 0.
- **Engine WRAM headroom:** a stock GB Studio 4.3.0 project leaves about **854 bytes** of WRAM free (the engine has 7,776 bytes to work with and uses 6,922 of them). With this plugin installed roughly **751 bytes** remain. Adding more global variables to your project does not change that figure, because script memory is a fixed 3,584 byte block at stock engine settings.
- **SRAM:** not used. Save slots and cartridge requirements are unaffected.

---

## Credits

- Original **CustomProjectile** plugin: fredrikofstad.
- Pause-on-locked-script idea: NukeOTron.

---

<!-- BANK0:BEGIN -->
## Bank 0 (HOME) Usage

Bank 0 is the 16 KB fixed ROM bank shared by the GB Studio engine core, the
interrupt handlers and the GBDK runtime. Extra banked ROM is cheap to add,
bank 0 is not, so bank 0 is usually the first thing a project runs out of.

| | Bytes |
|---|---|
| Bank 0 used by this plugin | **+85** |
| Bank 0 free with this plugin installed | **1,366** of 16,384 (92% used) |

Everything else this plugin adds lives in banked ROM.

| Module | This plugin | Stock engine | Bank 0 cost |
|---|---|---|---|
| `core/projectiles.c` | 1,293 | 1,208 | +85 |

A module that replaces a stock engine file costs only the *difference*, because
the stock version's bank 0 bytes were being spent anyway.

<details><summary>How this was measured</summary>

GB Studio 4.3.0-e1, default engine settings. Each module was compiled with the
toolchain and flags GB Studio itself uses, and the bank 0 size the compiler
recorded was read back. The stock column is the same compile of the engine file
the module replaces.

The "free" figure assumes a stock project with this plugin and nothing else.
Your own number will differ, because other plugins and any engine settings that
change what the core compiles move it too.

</details>
<!-- BANK0:END -->

## Changelog

Grouped by the date each change was merged into the official
[gb-studio-plugins](https://github.com/gb-studio-dev/gb-studio-plugins) repository.

Only bug fixes, new features and feature changes are listed. Engine version
bumps, patch regeneration, packaging fixes and documentation edits are omitted.

### 2026-08-09

- Replaced the definition's *Bounce* strength with a **Tile Collision Override**,
  which replaces the tile collision mask a projectile tests outright when non-zero, so one
  slot can react to a fixed set of tile bits regardless of which direction it is testing.
  Compiled out unless the new **Enable tile collision override** engine setting is
  ticked. A Chain borrows the same byte for its *Catch-Up Speed*, as it already does
  with the collision field.
- Reworked *Tile Collision Behaviour* into **Pass through / Remove projectile /
  Bounce (perfect reflect) / Stop on impact**. Bounce now negates the struck axis
  exactly instead of rebounding at a set strength, and the new Stop on impact mode
  halts the projectile where it struck. **Removes the old *Bounce (only floor)*
  mode**. A slot that used it now bounces off every face, and any **Bounce** value it
  had is ignored.

- Added the spare collision tile bits to every scene type's paint palette as **Extra**
  swatches, one per value each scene type leaves free. CollisionExPlugin
  declares the same palettes; the arrays are byte-identical, so this plugin's tile collision
  override has its tile values paintable whether or not that plugin is installed.

### 2026-08-08

- Fixed projectile bouncing.

### 2026-08-07

- Renamed the "Custom Projectiles" settings group to "Dynamic Projectiles".
- Added an initial animation frame when shooting a projectile.
- Added an onTileEnter event, and split the tile collision handling.
- Added more `#define` engine settings.

### 2026-08-03

- Fixed the stock projectile applying inverted gravity.

### 2026-08-02

- Initial release: arc, boomerang, sine wave, orbit, hookshot, anchor and custom projectile types, with gravity, bounce and tile collision.
