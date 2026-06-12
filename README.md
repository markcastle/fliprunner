# Flip Runner Racing

A remastered, browser-native rebuild of the original FlipRunner: a 2D physics driving game in the hill-climb tradition. One car, three inputs (gas, brake, parachute), ten levels of hills, jumps, collapsing bridges, ice and ravines. The whole game, physics engine, renderer, audio synthesiser, levels and UI included, lives in a single `index.html` file with zero dependencies and zero network requests.

## What's in it

- **10 levels** across grass and snow themes: a tutorial drive, rolling hills, jumps, collapsing wood bridges, ice fields, big parachute drops, ice bridges, a fuel-scarcity level, frozen heights, and a final gauntlet that combines everything.

- **Controls**: arrows/WASD for gas and brake, Space for the chute, R to restart, P to pause. On touch: GAS and BRAKE buttons, plus you can hold either half of the screen, and the chute button pulses when it's deployable.

- **Mechanics from the original**: fuel that drains and refills from cans, coins and gems, flip bonuses on landing, a big air bonus, individually collapsing bridge planks (ice breaks twice as fast as wood), zero-grip ice, a speedometer dial, 3-star scoring (finish / 55% of coins / 80% of coins plus all gems), and level unlock progression saved to localStorage with an in-memory fallback so it still works in sandboxed previews.

- **Audio**: a synthesized engine, pickup/crash/win sound effects and a toggleable chiptune loop, all generated live with WebAudio. No audio assets.

Ferries are the one feature descoped from the original. The terrain DSL (described below) makes them addable later as a moving platform type, following the same pattern as bridges.

## How to play

The goal of every level is simple: reach the chequered flag without crashing and without running out of fuel. Everything else is about how stylishly you get there.

### Controls

| Action | Keyboard | Touch |
| --- | --- | --- |
| Gas | Up arrow, Right arrow, W or D | GAS button, or hold the right half of the screen |
| Brake / reverse | Down arrow, Left arrow, S or A | BRAKE button, or hold the left half of the screen |
| Parachute | Space | The chute button (pulses when you can use it) |
| Restart level | R | The circular arrow button, top left |
| Pause | P or Esc | The pause button, top right |

### The rules of the road

- **Fuel** drains while you drive and trickles away even when idle. Grab the orange fuel cans to refill to 100%. Run dry away from a slope and the level is over.

- **Coins and gems** are scattered along the route, often along the flight path of a jump. Gems are rarer and required for a 3-star finish.

- **Flips earn coins.** Hold gas in the air to rotate backwards, brake to rotate forwards. Land a full rotation and you bank bonus coins. Land on your head and you bank a restart.

- **The parachute** can only be opened in the air, and it auto-releases the moment you touch down. It kills your fall speed and levels the car out, which is the difference between life and death on the big drops. It also kills your forward speed, so don't open it mid-jump unless you enjoy the bottom of ravines.

- **Bridges collapse.** Every plank starts falling shortly after you touch it. Wood gives you a moment, ice gives you about half a moment. Keep your speed up and never, ever stop on one.

- **Ice has no grip.** You can't accelerate or brake meaningfully on it, so all your planning has to happen before you get there. Build speed on solid ground before an icy climb, and shed speed before an icy descent into something pointy.

- **Stars**: 1 star for finishing, 2 for finishing with at least 55% of the coins, 3 for at least 80% of the coins plus every gem. Progress and best stars are saved automatically.

## How to deploy (Cloudflare Pages, web dashboard, no Wrangler)

The game is a single static file, which makes this about as easy as deployment gets.

1. On your machine, create a folder called `fliprunner` (any name works).

2. Log in to the Cloudflare dashboard at `https://dash.cloudflare.com`.

3. In the left sidebar go to **Workers & Pages**, then click **Create**, choose the **Pages** tab, and select **Upload assets** (the direct upload option, not the Git integration).

4. Give the project a name, for example `fliprunner`. This becomes your default subdomain: `fliprunner.pages.dev`.

5. Drag the `fliprunner` folder into the upload area (or click to browse and select it), then click **Deploy site**.

6. That's it. Cloudflare gives you the live URL on the success screen. The game runs entirely client-side, so there's nothing to configure: no build command, no environment variables, no functions.

To ship an update later, open the project in **Workers & Pages**, go to the **Deployments** tab, click **Create deployment** and upload the folder again. Each deployment is versioned and you can roll back from the same screen. If you want a custom domain, the **Custom domains** tab on the project walks you through adding one.

One thing worth knowing: player progress is stored in the browser's localStorage under the key `fliprunner_save_v1`, scoped to your domain. Redeploying doesn't wipe anyone's progress, but moving to a different domain effectively starts everyone fresh.

## Tech deep dive

This section is the honest story of how the game works and how it got built, including the bugs that the test harness caught along the way. If you want to modify the game, this is your map.

### Architecture at a glance

Everything is in one HTML file, organised top to bottom inside a single `<script>` as: utilities and persistence, canvas and input, audio, terrain, levels, game state, physics, per-frame update, rendering, UI wiring and the main loop. There is no framework, no bundler and no external asset. The rendering is Canvas 2D; menus, HUD and touch buttons are plain DOM positioned over the canvas, because DOM is simply better at buttons than canvas is.

The main loop is a standard `requestAnimationFrame` driver. Rendering happens every frame at whatever rate the display runs, but physics runs on a fixed timestep: the frame delta is sliced into substeps of 1/240s and the physics function is called once per slice. This is the classic decoupling that keeps physics deterministic and stable regardless of display refresh rate, and it's also what made headless testing possible, because the same `update(dt)` can be driven from Node at any speed with no canvas at all.

### Terrain: a heightfield plus a DSL

The world is a heightfield: one height sample every 10px, linearly interpolated between samples. Alongside the heights runs a parallel surface array marking each sample as road, ice, or pit (the lethal floor of a ravine). Querying the ground is two array lookups, which is why the physics can afford to do it many times per substep.

Levels are not data files. Each level is a small `make(t)` function that drives a terrain builder DSL:

```js
t.flat(440); t.hint(280, "Bridges collapse. KEEP MOVING!");
const b1 = t.curX(); t.bridge(520, "wood"); t.coins(b1 + 40, 8, 56, 52);
t.jump(280, 65, 280, 140, { coins: 5 });
t.hill(900, 160); t.coinArc(t.curX() - 730, 6, 56, 60, 140);
```

The primitives are `flat`, `hill`, `valley`, `bumps`, `drop`, `rampUp`, `rampDown`, `kicker`, `gap`, `bridge` and `jump`, plus item helpers (`coins`, `coinArc`, `gem`, `fuel`, `hint`) that place pickups relative to the drivable surface via a `topAt(x)` query that knows about bridges. All positions in level code are derived from the build cursor (`t.curX()`) rather than magic numbers, so inserting a feature mid-level doesn't silently misplace everything after it. A seeded PRNG (`mulberry32`) sprinkles trees and rocks deterministically, so a level always looks the same.

Two primitives deserve special mention because they encode hard-won lessons:

- **`kicker(len, h)`** is a quarter-cosine rise that ends at maximum steepness. The naive cosine-eased ramp flattens out at the top, which means the car leaves it travelling horizontally and flies straight into the far wall of whatever gap comes next. A real jump lip has to still be rising where the car leaves it.

- **`jump(run, kick, gapW, land, opts)`** is a complete composite: a kicker, a ravine whose far lip sits `land` pixels *lower* than the launch lip, a cushioning downslope on the landing, and coins laid along the actual flight arc (a shallow descending parabola, not the decorative high arc you might be tempted to draw). The lower landing is what makes jumps forgiving: the car gains clearance for free as it crosses.

### The car: two springs and a helmet

The car is a single rigid body with mass, rotational inertia and two wheels. The wheels aren't bodies; each is a spring-damper contact probed against the terrain (and against bridge planks) at its anchor point. When a wheel penetrates the surface, it pushes back along the surface normal with `N = k * penetration + d * approach_speed`, clamped to a maximum so numerical spikes can't explode, applied at the anchor so it also produces torque. The car's roughly 114px wheelbase makes the world about 28px per metre if you like thinking in real units.

Forces along the surface tangent are where the driving model lives:

- A light rolling resistance proportional to speed.

- A drive force when on the gas, tapering linearly to zero at top speed, clamped by `mu * N * 2.4` so grip limits acceleration.

- A braking force, clamped by `mu * N * 3.2`, which transitions into reverse drive below walking pace.

Ice is nothing more than `mu = 0.12` instead of `1.1`. That one number produces all the ice behaviour: you can barely accelerate, barely brake, and slopes win every argument.

In the air, gas applies backward rotation and brake applies forward rotation, which is the flip mechanic. If the player isn't touching anything during a short hop, the car gently drifts its nose toward the velocity direction, which makes ordinary downhill flights land cleanly without the player thinking about it. When grounded, a spring aligns the chassis to the local slope, which is what stops washboard terrain from accumulating pitch error and flipping you at speed. These two alignment behaviours are the difference between "physics demo" and "game".

The crash rule is: the driver's helmet (a circle above the chassis) touching terrain ends the run, but only if it hits with real velocity *into* the surface (or grinds in very deep). The "into the surface" part matters. An earlier version used total speed, which meant any helmet graze at 700px/s was lethal and fast grazing landings were unsurvivable.

The parachute applies strong velocity-proportional drag (terminal velocity works out around 330px/s, a comfortable landing) plus an attitude controller that wraps angles correctly and steers the car level. It only deploys after 0.12s of airtime and releases itself on touchdown.

### Bridges, pits and dying

A bridge is a record over a gap: a row of planks, each with its own tiny state machine (intact, shaking, falling, gone). A wheel touching a plank starts its shake timer: 0.45s for wood, 0.22s for ice, after which it detaches and falls with its own tumble. The physics treats a live plank as a flat surface at bridge height, so the heightfield underneath can be a ravine.

Ravine floors are marked `SURF_PIT`, and a wheel resting on a pit surface ends the run. Importantly this is checked per wheel contact, not at the car's centre. The first version checked the centre and produced two absurd deaths: cars were killed at the moment of takeoff (centre past the lip, rear wheel still on it) and killed for crossing perfectly healthy bridges (the terrain *under* the planks is pit). The fall-out-of-the-world check is relative to the local ground height rather than an absolute altitude, because an absolute kill plane triggered mid-air on The Big Drop, a level that legitimately descends 1900px.

### Rendering and audio

The renderer draws back to front: gradient sky, sun, parallax clouds, sine-composed far hills, then the world under a camera transform (translate, zoom, shake). Terrain is one filled dirt polygon for the visible slice plus stroked grass/snow lips and an ice overlay; only the samples in view are walked. The car is drawn as canvas paths (body, driver, helmet, spinning spoked wheels, antenna), the parachute as quadratic curves with shroud lines back to the chassis. The camera leads the car based on velocity, zooms out with speed and during big air, and decays a shake value fed by hard landings and crashes.

Audio is fully synthesized. The engine is a sawtooth oscillator through a lowpass filter whose pitch and gain track speed and throttle. Effects are short envelope-shaped oscillator blips and filtered noise bursts. The music is a 32-step chiptune loop (square lead, triangle bass) scheduled ahead of time against the AudioContext clock with a lookahead timer, the standard pattern for glitch-free WebAudio sequencing. The context is created lazily on the first user gesture, which is what mobile browsers require.

### Input, saving, and the small stuff

Keyboard, the on-screen buttons and the screen-half holds all funnel into one input state with counters per source, so overlapping touches and keys can't get stuck. Pointer events are used throughout (not touch events) so the same code serves mouse and touch, with multi-pointer tracking for simultaneous gas and brake.

Saving wraps localStorage in try/catch with an in-memory fallback, so the game runs in private browsing and sandboxed iframes; you just lose persistence. The save stores unlocked levels, best stars per level and audio preferences. The tab pausing itself on `visibilitychange` is two lines and saves a lot of unfair deaths.

### How it was built: test-first for a game

The unusual part of this build is that the game was verified headless before a human ever played it. Three Node harnesses mock just enough browser (a Proxy-based canvas context, stub elements, fake localStorage) to load the game script and drive its exposed test hooks:

- **A smoke suite** builds all ten levels and asserts sane totals (width, coin/gem/fuel counts, finite heights), drives level 1 end to end, deploys the chute, and confirms planks collapse.

- **A completability bot** plays every level with simple human-like policies: hold gas, brake when the ground falls away just ahead, level the car toward the slope at the predicted landing point, chute on big drops, cap speed on icy descents. Every level must be finishable by this bot or the build fails.

- **A parameter sweep** built disposable test strips for the jump primitive across kicker heights, gap widths and landing drops, and mapped the envelope where full-speed approaches survive. The shipped jumps all sit inside that envelope.

This caught real, foundational bugs that casual playtesting would have misdiagnosed as "needs tuning":

- **Top speed was 185px/s.** The tangential friction model opposed the car's full velocity instead of wheel slip, so rolling friction grew until it exactly cancelled the engine. The fix (light rolling resistance, with grip only clamping drive and brake forces) took top speed to its intended ~820px/s, and incidentally explained why every jump was falling short.

- **Every gap was a wall.** Cosine-eased ramps flatten at the lip, so cars launched flat and hit the far side. The kicker primitive and lower landings came out of tracing actual trajectories frame by frame.

- **Suspension mushed through on hard landings.** The normal-force clamp was too low to absorb valley-crossing flights, letting the body sink until the helmet reached the ground. Raising the clamp made hill-climb-style flat-out valley sends survivable.

- **The kill plane and the pit rule** both produced the absurd deaths described above, found because the bot's death coordinates pointed at terrain that was obviously innocent.

The final state of the suite: all ten levels build clean, all ten are completable by the bot, the parachute is confirmed genuinely required on The Big Drop (a gas-only run dies there), and the render path is exercised across both themes without exceptions.

### Extending it

The terrain DSL is the intended extension point. A new feature usually means one new primitive (geometry plus any state, following `bridge` as the template for anything dynamic), one rule in the physics if it interacts with the wheels, and a draw function. Ferries from the original would be exactly this: a platform record with a position that oscillates between two x values, treated by `surfaceAt` the same way planks are. New levels are just new `make(t)` functions in the `LEVELS` array; coin/gem totals and star thresholds derive automatically at build time.

If you change physics constants, rerun the bot before trusting your tuning. It's the difference between "feels harder" and "is impossible", and it takes seconds.
