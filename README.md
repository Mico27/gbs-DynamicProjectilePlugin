# gbs-DynamicProjectilePlugin

**Version 4.3.0 — Requires GB Studio ≥ 4.3.0**

A GB Studio engine plugin that replaces the built-in projectile system with a configurable one. A projectile is no longer only a sprite travelling in a straight line: it can arc under gravity, weave, boomerang back, orbit an actor, act as a grappling hookshot, hang off an actor, be driven frame by frame from script variables, string a chain of sprites between two actors, or leave a trail of sprites behind it. Every projectile also gains tile-collision reactions, bouncing, impact scripts, and a set of per-launch parameters.

This is a port of [fredrikofstad's CustomProjectile plugin](https://github.com/fredrikofstad/GBStudioPlugins) (originally written for GB Studio 4.1.x), rebuilt for the reworked 4.3.0 engine and extended well beyond it.

The plugin ships with `DynamicProjectileExample/`, a complete project with one demo scene per behaviour.

---

## Table of Contents

1. [Concepts](#concepts)
2. [Project Setup](#project-setup)
3. [Size Limits and Restrictions](#size-limits-and-restrictions)
4. [Behaviours Reference](#behaviours-reference)
5. [Engine Settings](#engine-settings)
6. [Events Reference](#events-reference)
7. [Engine Fields Reference](#engine-fields-reference)
8. [Media](#media)
9. [Memory Footprint](#memory-footprint)
10. [Credits](#credits)

---

## Concepts

### Slots and Launches

A projectile is described by a **slot** — an entry in a shared definition table holding its sprite, speed, lifetime, collision groups and behaviour. Slots are what you launch from, and a single slot can be launched any number of times. The two core events map onto that split: **Define Projectile Slot** writes a slot, **Launch Projectile From Slot (Extended)** fires one.

Stock GB Studio has the same model — its *Load Projectile Slot* / *Launch Projectile From Slot* events — and this plugin's events use the same compiler helpers, so the two sets interoperate. A slot written by one can be launched by the other.

> **Order matters if you mix them.** *Define Projectile Slot* writes the whole definition, so running the stock *Load Projectile Slot* afterwards on the same slot wipes the behaviour back to Default.

### What Lives Where

A slot is a fixed 32 bytes, which is not enough room for everything, so configuration is split across the two events:

| Lives on | Holds | Applies to |
|----------|-------|------------|
| **Define Projectile Slot** | Sprite, speed, lifetime, collision groups, behaviour, and the behaviour's shape (gravity, bounce, wave size, chain slack, trail length…) | Every projectile launched from that slot |
| **Launch Projectile From Slot** | Source, aim, and the handful of values that vary shot to shot (arc height, orbit target, chain endpoints, trail head…) | That one shot |

Because a live projectile stores only an *index* into the slot table, redefining a slot changes the behaviour of anything already in flight from it. Only the lifetime and the per-launch parameters are truly per-projectile.

### The Projectile Pool

All live projectiles come from a fixed pool, sized by the **Max concurrent projectiles** setting (default 6). A launch that finds the pool full is silently dropped. Bear in mind how many entries a shot actually consumes: most behaviours take one, but a **hookshot takes four** — the head plus three chain links.

### Strands

**Chain** and **Trail** are the two behaviours that draw a *run* of sprites from a single projectile. Those sprite positions cannot be derived from the projectile's own position, so they are kept in a **strand** — a small buffer taken from a second pool, sized by **Max chains + trails** and **Points per chain / trail**.

A chain or trail launched when every strand is busy is silently not launched, exactly as it would be if the projectile pool were full.

---

## Project Setup

### Installation

Copy `src/DynamicProjectilePlugin` into your project's `plugins/` folder, keeping the whole folder intact.

This plugin replaces the stock projectile engine files. Three other plugins in the same collection — **SceneStackExPlugin**, **DynamicActorPlugin** and **SpritesheetChangeBufferPlugin** — touch some of the same files, so pre-merged compatibility variants are included for every combination of them and are selected automatically. Because this plugin is applied last of the four, its merged build also resolves the clash between DynamicActorPlugin and SpritesheetChangeBufferPlugin.


### 1. Define a Slot

Add **Define Projectile Slot** wherever the projectile should be configured — usually a scene's **On Init**, since a slot persists until something overwrites it.

The *Projectile* tab is the stock projectile definition: sprite sheet, animation state, speed, animation speed, life time, initial offset, collision group and collide-with mask.

The *Dynamic projectile* tab is this plugin's behaviour. Start from a **Preset** and you are done in one dropdown:

| Preset | Behaviour |
|--------|-----------|
| Bullet | Straight, dies on walls |
| Lobbed shot | Arcs under gravity |
| Grenade | Gravity, bounces off walls |
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

Add **Launch Projectile From Slot (Extended)**. The *Source* tab picks where the shot starts and where it aims; the *Dynamic projectile* tab carries the per-launch parameters.

Set **Slot Behaviour** on that tab to whatever the slot holds. It has no effect on the projectile — it only decides which parameter fields are shown, since the launch event cannot know what the slot contains.

### 3. Add Impact Scripts (optional)

**Set Projectile Removal Script**, **Set Projectile Tile Hit Script** and **Set Projectile Actor Hit Script** register a script that runs when *any* projectile is removed, hits a wall, or touches an actor. Set them once per scene; they persist until cleared or the scene changes.

Each slot has a matching checkbox (*Run On Remove script* and friends) so an individual slot can opt out.

### 4. Trim the Behaviours You Do Not Use (optional)

Every behaviour except Default has an on/off switch under **Settings → Engine → Custom Projectiles**. These are compile-time switches — turning one off removes its code from the ROM rather than merely skipping it at runtime. The behaviours are by far the biggest lever on the plugin's ROM cost.

The events refuse to compile against a behaviour that has been switched off, naming the offending scene, rather than silently producing a projectile that does nothing.

---

## Size Limits and Restrictions

### The projectile pool can run out

All live projectiles come from a fixed pool, sized by **Max concurrent projectiles**. A launch that finds the pool full is **silently dropped**. Remember how many entries a shot really consumes: most behaviours take one, but a **hookshot takes four** — the head plus three chain links.

### The strand pool can run out

**Chain** and **Trail** each need a strand buffer from a second pool, sized by **Max chains + trails** and **Points per chain / trail**. A chain or trail launched when every strand is busy is silently not launched, exactly like a full projectile pool.

### Redefining a slot changes projectiles already in flight

A live projectile stores only an index into the slot table, so redefining a slot changes the behaviour of anything already flying from it. Only the lifetime and the per-launch parameters are truly per-projectile.

### Order matters if you mix with the stock events

*Define Projectile Slot* writes the whole definition, so running the stock *Load Projectile Slot* afterwards on the same slot wipes the behaviour back to Default.

### Stock engine files are replaced

This plugin replaces the stock projectile engine files. Compatibility variants are included for **SceneStackExPlugin**, **DynamicActorPlugin** and **SpritesheetChangeBufferPlugin** and are selected automatically; any other plugin touching the same files needs a manual merge.

---

## Behaviours Reference

Every numeric field on both events is a script **value**, so it accepts a variable or an expression as well as a fixed number.

---

### Default

A straight-line projectile — the stock behaviour, plus this plugin's tile collision, bounce and impact scripts.

**Slot fields:** Tile Collision Behaviour · Gravity · Bounce
**Launch parameters:** none

---

### Arc

Thrown upward, then pulled back down by gravity. Gravity is applied every other frame.

**Slot fields:** Tile Collision Behaviour · Gravity · Bounce
**Launch parameters:** *Launch Height* — how high it is thrown before gravity takes over

---

### Boomerang

Sheds speed as it travels, reverses, and comes back.

**Slot fields:** Tile Collision Behaviour · Gravity · Bounce
**Launch parameters:** *Range* — how quickly it sheds speed. Higher values bring it back sooner, so it travels **less** far

---

### Sine Wave

Weaves from side to side, perpendicular to its direction of travel.

**Slot fields:** Tile Collision Behaviour · Gravity · Bounce · *Wave Amplitude* (how far it weaves) · *Wave Frequency* (how tight the zigzag)
**Launch parameters:** *Starting Phase* — where in the wave it begins, so shots fired together do not overlap

---

### Orbit

Circles an actor at a fixed radius, following it as it moves.

**Slot fields:** Tile Collision Behaviour · *Orbit Radius* · *Orbit Speed*
**Launch parameters:** *X / Y Offset* (offset from the actor's centre) · *Starting Angle* · *Actor To Circle* (actor index, 0 = player)

Orbiters share one target actor — the one resolved by the most recent launch. Space several out around the same actor by giving each a different starting angle.

---

### Hookshot

A grappling hook: a head that flies out, up to three chain links that span the gap behind it, and a set of reactions for what happens when the head lands. See [Hookshot Detail](#hookshot-detail) below.

**Slot fields:** Tile Collision Behaviour (*Pass through* / *React to tiles*) · Gravity · *Hookshot: On Tile Hit* · *Hookshot: On Actor Hit*
**Launch parameters:** *Chain Link* (0 = head, 1–3 = links) · *Anchored To Actor* (the actor it is fired from)

Launch the head and its links as separate shots from the same slot, varying only *Chain Link*.

---

### Anchor

Rigidly pinned to an actor at a fixed offset, whichever way that actor turns. Useful for held weapons and carried objects.

**Slot fields:** Tile Collision Behaviour
**Launch parameters:** *X / Y Offset* · *Attached To Actor*

Unlike Orbit, each anchored projectile carries its own target, so several can hang off different actors at once.

---

### Custom

Moves by whatever two script variables contain, re-read every frame. Everything else — collision, lifetime, impact scripts — behaves normally.

**Slot fields:** Tile Collision Behaviour · *Delta X Variable* · *Delta Y Variable*
**Launch parameters:** none

---

### Chain

A run of sprites strung between two actors. A chain has no velocity, no aim and no tile collision; the launch position and angle only decide which of the sprite's four direction animations the links are drawn with.

**Slot fields:** *Chain Type* · *Links* (counting the one on each actor) · *Slack* and *Catch-Up Speed* (loose only)
**Launch parameters:** *Strung From Actor* · *Strung To Actor* (actor indexes, 0 = player)

| Chain Type | Placement |
|------------|-----------|
| **Straight** | Links are spread evenly along the line between the two actors, recomputed every pass. |
| **Loose** | Each link chases its neighbour, but only once the gap between them opens past **Slack**, and then only by **Catch-Up Speed** pixels per update. That tolerance is what makes a chain hang and drag instead of snapping into a line. A Catch-Up Speed of 0 makes it rigid, closing the whole gap in one step. |

A loose chain is seeded straight when launched and only then starts relaxing, so a leash appears taut for a frame and then sags.

Give the slot a **Collide With** mask and every link becomes dangerous along its whole length; leave it empty and the chain is decoration. The two end links are never tested — they sit on the actors the chain is tied to, so testing them would report a hit on its own anchors every pass.

A chain is **not** removed when it scrolls off screen, unlike every other behaviour: both its ends are actors, which the engine keeps alive on or off screen, and a leash cut the moment its midpoint scrolled away would never come back. It ends when its lifetime runs out.

---

### Trail

Travels like a plain shot — speed, gravity, bounce and tile collision all behave normally — and additionally records where it has been, hanging a sprite off its own history every few updates.

**Slot fields:** Tile Collision Behaviour · Gravity · Bounce · *Trail Segments* · *Trail Spacing*
**Launch parameters:** *Trail Head* (this projectile / an actor) · *Head Actor*

The tail is tested for actor collisions along with the head, so the whole trail is dangerous rather than just the sprite leading it. Widening the spacing stretches the tail further behind at no extra sprite cost, but it costs strand points: a trail needs `segments × spacing` of them.

Spacing starts at 2, not 1 — entry 0 of the history is the position written this pass, which is where the projectile itself is drawn, so a spacing of 1 would stack the first segment on top of the head.

**Hanging a trail off an actor.** Set *Trail Head* to *An actor* and the projectile stops being a shot and becomes pure tail: it records where that actor has been and draws segments along it. This is how to give a trail to something that already moves under its own logic — a walking NPC, a platform, the player. With a head actor the projectile hands everything positional over to that actor, so it is not drawn, does not move, ignores tile collision, is not removed off screen, and never reports a hit on its own head.

Successive chain links and tail segments step on through the slot's animation, wrapping at its end. With a one-frame animation they all look alike; with a two-frame one they alternate.

---

### Hookshot Detail

The hookshot decides for itself what to do when its head lands, so a working grapple needs no scripting.

| Field | Options |
|-------|---------|
| *Hookshot: On Tile Hit* (needs *Tile Collision Behaviour* = React to tiles) | **Return** · Pull Player · Remove Hook · Stay Where It Hit |
| *Hookshot: On Actor Hit* | **Remove Hook** · Return · Pull Player · Pull Actor · Stay Where It Hit · Stick To Actor |

- **Return** — the head turns around and comes back.
- **Pull Player** — drags the source actor over to where the head landed.
- **Remove Hook** — the hook and its chain disappear on impact.
- **Stay Where It Hit** — the head parks at the impact point and the chain keeps spanning the gap. Recall it later with *Set Hookshot State → Returning*.
- **Pull Actor** — reels the actor that was hit back in. The plugin picks up the actor it actually touched, so no index has to be known in advance.
- **Stick To Actor** — the head latches onto the actor it hit and rides along with it, chain and all, keeping the offset it made contact at. A hook that catches an enemy's shoulder stays on the shoulder.

Set *On Tile Hit* to **Pull Player** and *On Actor Hit* to **Pull Actor** for the usual grappling-hook behaviour: walls pull you to them, enemies get pulled to you. The defaults (Return / Remove Hook) preserve the original plugin's behaviour.

*Pull Player* and *Stay Where It Hit* leave the hook in play until something ends it, so pair them with a recall or with firing again — a new shot clears the old one.

Only the head reacts: chain links never trigger tile or actor collisions and never end the shot by scrolling off screen, and a hookshot stops looking for actors once it lands, so a target's hit script fires once rather than every frame of a pull.

**The chain follows its source.** The links are placed each frame at 3/4, 1/2 and 1/4 of the way along the line from the *Anchored To Actor* actor to the head, so walking around with a hook parked or stuck keeps the chain spanning the gap. That parameter is an actor index, so pointing it at any actor gives an NPC a hookshot.

**Pulls stop at obstacles.** A pull moves an actor by writing its position directly, which nothing else in the engine gets a say in — so on its own it would reel somebody into a wall and leave them standing inside it. With **Hookshot pulls stop at obstacles** on (the default), each step is tested before it is committed and the pull ends the moment it is blocked. *Pull Player* stops the source actor short of a solid tile or actor; *Pull Actor* stops the reeled-in actor short of the same, which includes the source actor itself, so a catch is never dragged inside the player. Testing is per axis, so a pull dragged along a wall slides down it. An actor counts as solid when it has collision enabled, and only the edge being moved towards is tested, so one-way tiles still block just the side they face.

**Pulls put the player back on the grid.** A pull moves in sub-pixel steps, so it almost never stops exactly on a grid point — and a top-down scene only reads input while the player is *on* one, so an actor left between two can end up unable to move at all. Every pull therefore realigns the source actor before it lets go, to the nearest point on the grid chosen by **Hookshot pull realigns to grid**. If the aligned spot turns out to be inside something, the actor gives that cell back, and failing that stays where the pull stopped it — off grid being better than inside a wall.

---

## Engine Settings

Found under **Settings → Engine → Custom Projectiles**. Indented entries belong to the behaviour above them and only appear while it is switched on.

| Setting | Range | Default | Description |
|---------|-------|---------|-------------|
| **Max concurrent projectiles** | 1–20 | 6 | Size of the live projectile pool. A launch that finds it full is dropped. Each entry costs 16 bytes of WRAM. |
| **Max projectile slots** | 5–20 | 5 | Size of the definition table. Five is a floor, not a choice — GB Studio's scene loader always fills slots 0–4 itself. Each slot costs 32 bytes of WRAM. |
| **Off-screen margin (tiles)** | 0–10 | 2 | How far past the screen edge a projectile may travel before it is retired. 0 retires it the moment its origin leaves the screen. |
| **Tile collision detection** | Origin point / Bounding box | Origin point | Whether tile tests use a single lookup at the projectile's position, or every tile its bounds rect covers. Bounding box makes large projectiles stop as soon as any part of them touches a wall, at the cost of several tile reads per projectile per frame. |
| **Enable behaviour: …** | on / off | on | One compile-time switch per behaviour except Default. |
| ↳ **Hookshot pull realigns to grid** | Off / 8px / 16px | 8px | Grid a pull puts the source actor back on. Set it to match your top-down scene, or *Off* for scene types where snapping the player is wrong. |
| ↳ **Hookshot pulls stop at obstacles** | on / off | on | Whether a pull stops short of solid tiles and actors. |
| ↳ **Max chains + trails** | 1–8 | 2 | Strands available to Chain and Trail. Shown while either behaviour is on. |
| ↳ **Points per chain / trail** | 2–32 | 16 | Points in one strand. A chain needs one per link; a trail needs `segments × spacing`. |

The grid setting cannot be read from the scene itself: the top-down grid size lives in a file GB Studio does not compile into a project with no top-down scene, so reading it would fail to link in a platformer.

Collision spreading is the **stock** *Collision checks* field under **Settings → Engine → Projectiles**, not one of this plugin's. The plugin honours it, so all three values work as they do for stock projectiles.

---

## Events Reference

All events appear under the **Projectiles** group in the script editor.

---

### Define Projectile Slot

**`DYNPROJ_EVENT_LOAD_PROJECTILE_SLOT`** — auto-label: *Define Projectile Slot N : &lt;preset&gt;*

Writes a complete projectile definition into a slot: everything the stock *Load Projectile Slot* event writes, plus this plugin's behaviour. Both halves are written in one event because the order matters — the stock half copies a whole definition into the slot and would wipe the behaviour if it ran second.

**Projectile tab**

| Field | Description |
|-------|-------------|
| Projectile Slot | Slot to write (0 to *Max projectile slots* − 1). A plain number, not one of five buttons. |
| Sprite Sheet / Animation State | The projectile's appearance. |
| Speed / Animation Speed | Travel speed and animation rate. |
| Life Time | Seconds before it expires on its own. |
| Initial Offset | Distance in front of the source to spawn at, in pixels. |
| Loop Animation | Whether the animation repeats or holds on its last frame. |
| Destroy On Hit | Unchecked makes it a *strong* projectile that survives contact. |
| Collision Group / Collide With | Standard GB Studio collision groups. |

**Dynamic projectile tab**

| Field | Description |
|-------|-------------|
| Preset | Ready-made behaviour, or *Custom* to choose components. |
| Behaviour | Which behaviour this slot holds (Custom only). |
| Tile Collision Behaviour | Pass through · Remove projectile · Bounce · Bounce (only floor). |
| Gravity | Downward pull, applied every other frame. Clamped to −8…7. |
| Bounce | How hard it rebounds, shown when the collision mode bounces. |
| *(behaviour shape fields)* | Wave amplitude/frequency, orbit radius/speed, chain type/links/slack/catch-up, trail segments/spacing, hookshot reactions, custom delta variables — see [Behaviours Reference](#behaviours-reference). |
| Infinite lifetime | Ignore *Life Time* so these never expire on their own. |
| Ignore player collision | Pass through the player. |
| Run On Remove / Tile Hit / Actor Hit script | Whether this slot triggers each shared impact script. All on by default. |

---

### Launch Projectile From Slot (Extended)

**`DYNPROJ_EVENT_LAUNCH_PROJECTILE_SLOT`**

Fires a projectile from a slot. Same as the stock launch event, but the slot is a plain number so it reaches the extra slots, and it adds a source-from-position mode, a position-target aim mode, and the per-launch parameters.

**Source tab**

| Field | Description |
|-------|-------------|
| Projectile Slot | Slot to launch from. Must be below *Max projectile slots*, and something must have been defined in it first. |
| Launch From | **Actor** (with a pixel *Offset X / Y*) or **Position** (a fixed point, no actor involved). |
| X / Y | Position source only. Values with a tiles / pixels unit toggle, so a random spawn column can be an expression like `16 + rnd(120)` in pixels. |
| Launch At | *Fixed Direction* · *Angle* · *Angle Variable* · *Actor Direction* · *Actor Target* · *Position Target*. The actor-relative options need an actor source. |

*Position Target* computes the angle at runtime the same way the stock *Actor Target* does, so a turret can aim at a fixed point in the room — and with a *Position* source you get a shot fired from one coordinate towards another with no actors involved at either end.

**Dynamic projectile tab**

| Field | Description |
|-------|-------------|
| Slot Behaviour | Display only — decides which parameters below are shown. The real behaviour comes from the slot. |
| *(per-launch parameters)* | See each behaviour in [Behaviours Reference](#behaviours-reference). |

---

### Launch Projectile From Slot By Index (Extended)

**`DYNPROJ_EVENT_LAUNCH_PROJECTILE_SLOT_BY_INDEX`**

Identical to the above, except every actor picker is an actor **index** script value instead — the source actor, the *Actor Direction* actor, the *Actor Target* actor, Anchor's *Attached To Actor* and Trail's *Head Actor*. Use it when the actor has to come from a variable.

---

### Set Hookshot State

**`DYNPROJ_EVENT_SET_HOOKSHOT`**

Drives the active hookshot's state machine by hand, whatever its impact reactions are set to.

| Field | Description |
|-------|-------------|
| Set Hookshot State | Firing · Returning · Pull Player · Pull Actor · Remove. |
| Actor to pull | The actor a *Pull Actor* state reels in. |

*Returning* recalls a hook that is parked or stuck; *Remove* clears the whole hookshot, which is the usual way to tidy up before firing a new one.

---

### Set Hookshot State By Index

**`DYNPROJ_EVENT_SET_HOOKSHOT_BY_INDEX`**

As above, with the actor given as an index script value.

---

### Set Projectile Removal Script

**`DYNPROJ_EVENT_SET_REMOVAL_SCRIPT`**

Registers a script that runs whenever any projectile is removed — expired, off screen, or destroyed on impact.

| Field | Description |
|-------|-------------|
| Action | Set script · Clear script. |
| On Removal | The script to run. |

---

### Set Projectile Tile Hit Script

**`DYNPROJ_EVENT_SET_TILE_HIT_SCRIPT`**

Registers a script that runs whenever a projectile's *Tile Collision Behaviour* reacts to a solid tile. A projectile in a bounce mode runs it on **every** bounce, so keep it short.

| Field | Description |
|-------|-------------|
| Action | Set script · Clear script. |
| On Tile Hit | The script to run. |

---

### Set Projectile Actor Hit Script

**`DYNPROJ_EVENT_SET_ACTOR_HIT_SCRIPT`**

Registers a script that runs whenever a projectile touches an actor in its collision mask. A *strong* projectile runs it for as long as it overlaps.

| Field | Description |
|-------|-------------|
| Action | Set script · Clear script. |
| On Actor Hit | The script to run. |

---

### Set Projectile Lifetime

**`DYNPROJ_EVENT_PROJECTILE_LIFETIME`**

Globally suspends lifetime countdown, so nothing expires on its own until it is switched back. Slots with *Infinite lifetime* are unaffected either way.

| Field | Description |
|-------|-------------|
| Infinite Lifetime (Global) | Default · Infinite Lifetime. |

---

### Pause Projectiles

**`DYNPROJ_EVENT_PAUSE_PROJECTILES`**

Freezes projectile updates. Paused projectiles keep their pool entry and resume where they left off.

| Field | Description |
|-------|-------------|
| Pause All Projectiles | Off · Pause All Projectiles · Pause on Locked Script. |

*Pause on Locked Script* freezes them automatically whenever a script holds the VM lock, which is the usual way to stop projectiles moving during dialogue.

---

### Show All Projectiles / Hide All Projectiles

**`DYNPROJ_EVENT_SHOW_ALL_PROJECTILES`** · **`DYNPROJ_EVENT_HIDE_ALL_PROJECTILES`**

Stops or resumes drawing every projectile. They keep updating while hidden — this only affects rendering. No fields.

---

## Engine Fields Reference

These runtime fields are written by the engine just before an impact script runs. Read them with the stock **Engine Field Store** event; they do not appear on the Settings screen, since setting them by hand achieves nothing.

| Field | Description |
|-------|-------------|
| `Last Hit: X` / `Last Hit: Y` | Where the projectile was, in pixels. Refreshed on every removal whether or not a script is attached. |
| `Last Hit: Behaviour` | Which behaviour the projectile was using (Default … Trail). |
| `Last Hit: Actor` | The actor that was hit. Set by the **actor hit** script only, and stale for the other two. |

The remaining runtime fields are the working state the events drive — the per-launch parameters, the hookshot state and the resolved actor index. They are readable and writable through **Engine Field Store** / **Engine Field Update**, but every launch overwrites them.

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
| 2 - Boomerang | Range, infinite lifetime | B: light throw, A: heavy throw |
| 3 - Sine Wave | Amplitude / frequency / phase, with A's amplitude read from a variable | B: gentle wave, A: wider each press |
| 4 - Orbit | Three orbiters evenly spaced by starting phase; the third is fired with the By Index launch event | B: add 3 orbiters |
| 5 - Hookshot | Head + three chain links, all four impact reactions, and the chain tracking the player as they walk | B: fire hook, A: pick mode, SELECT: recall |
| 6 - Anchor | Two anchored projectiles on different actors at once | B: attach to player, A: attach to the target |
| 7 - Custom | Per-frame delta driven by two variables, plus an on-removal script | B: fire |
| 8 - Gravity & Bounce | *Bounce*, *Bounce (only floor)* and *Remove projectile*, with a tile hit script on each bounce | B / A / SELECT: one behaviour each |
| 9 - On Removal | A removal script reading position and behaviour back out of the `Last Hit` fields | B: fire |
| 10 - Global Controls | Pause / hide / lifetime switches, plus per-slot *Ignore player collision* | B: fire, A: options menu |
| 11 - Extra Slots | Slots 5–7, past what stock GB Studio can address, plus all four launch sources | B / A / SELECT, and the d-pad |
| 12 - Chain | Both placement modes strung from the player to the target — walk around to see the difference | B: straight chain, A: loose chain |
| 13 - Trail | A trail that hops under gravity, one that flies straight at the target with a longer tail, and a tail hung off the player | B / A / SELECT |

Notes on the demos:

- The walled arena has solid borders and four pillars so the tile-collision and bounce behaviours have something to react to.
- The **Target** dummy (actor index 1) is in collision group 1, so shots using the default collision mask are destroyed by it and print `HIT!`.
- Scene 10 resets the global pause / hide / lifetime switches on the way out, since those engine fields persist across scene changes.
- One hookshot uses four of the six default pool entries, so the hookshot demo clears any hook still out before firing a new one.
- Scenes 12 and 13 use a 4 second lifetime rather than an infinite one: the strand pool holds two, so a third chain or trail is dropped until one expires.

---

## Memory Footprint

Measured against the stock GB Studio **4.3.0-e1** engine at default engine settings, by diffing the link maps of the `gbs2` template built with and without the plugin installed. Values are the plugin's *delta* versus the stock engine. **DMG and CGB deltas are identical** — the plugin has no colour-specific paths. Using the plugin's events additionally compiles a few bytes of GBVM script per call into your project's script banks.

| Configuration | ROM | WRAM | Engine WRAM left |
|---------------|-----|------|------------------|
| **All behaviours on (defaults)** | **+8,805 bytes** | **+90 bytes** | **764 bytes** |
| Chain and Trail off, rest on | +5,380 bytes | −46 bytes | 900 bytes |
| All optional behaviours off | +1,175 bytes | −52 bytes | 906 bytes |

- **WRAM:** the figure can go *negative* because the plugin reshapes the pool. Stock GB Studio embeds a full 28-byte definition in every live projectile (41 bytes per entry); this plugin's entry is 16 bytes holding an index into the shared table instead. At default sizes that is 256 bytes of pool against stock's 345 — enough to pay for the plugin's own globals, and for the strand pool too once Chain and Trail are switched off.
- **Engine WRAM headroom:** the stock 4.3.0 engine leaves about **854 bytes** of WRAM free (usable engine WRAM is 7,776 bytes at 0xC0A0–0xDF00; the stock engine uses 6,922). With this plugin installed and everything enabled, roughly **764 bytes** remain. This figure does not depend on how many global variables your project defines: the script memory array is a fixed 3,584 bytes at stock engine settings.
- **Tuning:** the pool is 16 bytes × *Max concurrent projectiles* (96 by default), the slot table 32 bytes × *Max projectile slots* (160), and the strand pool 4 bytes × *Max chains + trails* × *Points per chain / trail* (128). The strand pool disappears entirely when both Chain and Trail are off.
- **ROM:** dominated by the behaviours themselves, so switching off the ones a game does not use is by far the biggest lever. Hookshot is the largest single behaviour.
- **SRAM:** none. Save slots and cartridge requirements are unaffected.

---

## Credits

- Original **CustomProjectile** plugin: fredrikofstad.
- Pause-on-locked-script idea: NukeOTron.

---

<!-- BANK0:BEGIN -->
## Bank 0 (HOME) Usage

Bank 0 is the 16 KB non-switchable ROM bank that the GB Studio engine core,
the interrupt handlers and the GBDK runtime all share. Banked ROM is cheap
(add another bank), bank 0 is not, so it is usually the first thing a project
runs out of.

| | Bytes |
|---|---|
| Bank 0 used by this plugin | **+85** |
| Bank 0 free with this plugin installed | **1,366** of 16,384 (92% used) |

Everything else this plugin adds lives in banked ROM.

| Module | This plugin | Stock engine | Bank 0 cost |
|---|---|---|---|
| `projectiles.c` | 1,293 | 1,208 | +85 |

Modules that replace or patch a stock engine file only cost the *difference*:
the stock version's bank 0 bytes were being spent anyway.

<details><summary>How this was measured</summary>

GB Studio 4.3.2, DMG target, default engine settings. Each module's bank 0
contribution is the `A _HOME size` record that SDCC writes into its `.rel`
object, summed over the engine sources this plugin provides. Stock sizes come
from building projects whose only plugin ships no engine C, so every module in
them is the untouched engine; two such builds were compared and agreed on all
73 shared modules.

The "free" figure is a stock project with this plugin and nothing else. Your
own number will differ: other plugins, and any engine settings that change what
the core compiles, move it independently of this plugin.

</details>
<!-- BANK0:END -->
