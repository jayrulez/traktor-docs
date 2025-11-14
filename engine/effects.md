---
layout: default
permalink: /engine/effects/
title: Effects
parent: Engine

nav_order: 11
---

# Effects System - Coordinated Visual and Audio Impact

Watch an explosion in a modern game. The billowing smoke, the bright flash, the cascading sparks, the thunderous boom, all perfectly synchronized. That's not just particles or just sound. It's an **integrated effect** where visual (VFX) and audio (SFX) combine to create impact. Fire crackles with sound, explosions flash and roar, magic spells shimmer with ethereal whooshes. Effects are what make games feel alive and responsive.

Traktor's effects system (the **Spray module**) provides an integrated approach to game effects:

**Visual Effects (VFX):**
- **Particles** - GPU-accelerated particle emitters for fire, smoke, sparks, magic
- **Trails** - Motion streaks for bullets, missiles, fast-moving objects
- **Meshes and points** - Rendered geometry as part of effects

**Audio Effects (SFX):**
- **Sound components** - Attach sounds to effects that play synchronized with visuals
- **Sound events** - Trigger sounds at specific points in an effect's timeline

**Effect Management:**
- **Layers** - Combine multiple emitters, sounds, and visual elements
- **Sequences** - Time-based choreography of effect elements
- **Event spawning** - Effects can trigger other effects

All of this is controlled through a single **EffectComponent** that handles playback, looping, and timing. The visual editor lets you preview effects in real-time, adjusting particles, trails, sounds, and timing until everything feels perfect.

![TODO: Screenshot showing effect editor with particle emitters, sound components, and timeline]

## How Effects Work

An effect in the Spray module can contain multiple coordinated elements working together:

### Visual Elements

**Emitters** spawn particles. You configure where they spawn (a point, a box volume, a sphere, or even the surface of a mesh), how many spawn per second (emission rate), and their initial properties (velocity, size, color, rotation). An emitter for an explosion might burst out hundreds of particles all at once, while an emitter for a torch flame continuously spawns particles at a steady rate.

**Particles** themselves are simple entities. They have position, velocity, size, color, rotation, and age. They're typically rendered as textured quads (billboards) that always face the camera, though they can also be oriented based on velocity or other factors.

**Trails** create motion streaks by rendering ribbons that follow moving objects. Perfect for bullets, missiles, or any fast-moving projectile that needs a sense of speed.

**Forces** affect particles after they spawn. Gravity pulls particles down, wind pushes them horizontally, vortex forces create spiraling motion, and turbulence adds realistic randomness. Forces make particles feel like they exist in the world rather than just floating arbitrarily.

**Modifiers** change particle properties over their lifetime. A particle might start small and grow larger, or begin opaque and fade to transparent. Color can shift. Flames start bright yellow, then turn orange and red before fading to black. These changes happen smoothly using curves or gradients.

**Rendering** determines how particles look on screen. Additive blending makes particles glow and layer bright effects. Alpha blending makes smoke and fog look soft and translucent. Textures provide detail. A fireball texture, a spark sprite, a smoke puff.

### Audio Elements

**Sound components** attach audio to effects, playing sounds when the effect plays. The sound can loop with the effect or play once as a one-shot.

**Sound events** trigger sounds at specific points in the effect's timeline. An explosion might have a flash sound at the start and a rumble sound 0.2 seconds later.

### Effect Layers and Timing

**Layers** combine multiple emitters, sounds, and other elements into a single effect. An explosion effect might have a layer for the initial flash, another for smoke, and a third for debris particles, all coordinated.

**Sequences** choreograph when different elements play over time. You can delay when an emitter starts, control when sounds trigger, and create complex multi-stage effects.

## Adding Effects to Your Scene

Setting up an effect is straightforward:

```cpp
// C++ - Create effect
Ref<EffectComponent> effect = new EffectComponent(effectResource);
entity->setComponent(effect);
```

From Lua, you can control the effect dynamically:

```lua
-- Lua - Control effect playback
local effect = self.owner:getComponent(spray.EffectComponent)

-- Enable/disable the effect
effect.enable = true
effect.enable = false

-- Loop control
effect.loopEnable = true  -- Effect repeats
effect.loopEnable = false  -- Effect plays once

-- Check if finished
if effect.finished then
    log:info("Effect has completed")
end

-- Reset effect to beginning
effect:reset()
```

**Note:** Effects are primarily configured in the editor (emitters, forces, colors, sounds, timing, etc.). The Lua API provides playback control. Starting, stopping, looping, and resetting effects. But not fine-grained control over individual particles or emission parameters.

## Creating Effects in the Editor

Effects are created and edited visually in the Traktor editor:

1. **Create an Effect asset** in the database
2. **Add visual elements:**
   - **Emitters** - Choose emitter type (point, box, sphere, mesh surface), set emission rate, define spawn area
   - **Trails** - Configure trail width, lifetime, and rendering properties
   - **Particles** - Set initial properties (velocity, size, color, rotation)
   - **Forces** - Gravity, wind, turbulence for particle movement
   - **Modifiers** - Size/color curves over lifetime, rotation over time
3. **Add audio elements:**
   - **Sound components** - Attach sounds that play with the effect
   - **Sound events** - Trigger sounds at specific times in the effect
4. **Configure layers** - Organize multiple emitters, sounds, and elements
5. **Set up timing** - Use sequences to choreograph when elements play
6. **Assign materials** - Texture and blending mode (additive for fire/sparks, alpha for smoke/fog)
7. **Preview and iterate** - The editor shows the effect in real-time with audio, so you can tweak and see/hear results instantly

## Common Effect Examples

### Explosion

An explosion effect combines visuals and audio. Particles burst outward, a bright flash, lingering smoke, and a thunderous boom:

**Visual configuration:**
- **Emitter:** Sphere emitter with radial velocity
- **Emission:** High burst rate (500-1000 particles), very short duration (0.1-0.2 seconds)
- **Initial velocity:** Radial from center, high speed (5-20 units/second)
- **Forces:** Gravity pulls particles down after initial burst
- **Size:** Starts small, rapidly grows, then shrinks
- **Color:** Bright yellow/white flash → orange → red → dark smoke
- **Lifetime:** 1-3 seconds
- **Blending:** Additive for initial flash, alpha for smoke

**Audio configuration:**
- **Sound component:** Explosion sound (loud, low-frequency boom)
- **Volume:** High (1.0)
- **3D positioning:** Enabled for distance attenuation

**Result:** A bright, expanding sphere of fire that quickly transitions to billowing smoke, synchronized with a powerful explosion sound.

### Fire

Continuous fire like a torch, campfire, or burning wreckage:

**Visual configuration:**
- **Emitter:** Point or small box at base of fire
- **Emission:** Continuous, moderate rate (50-100 particles/second)
- **Initial velocity:** Upward (2-5 units/second) with slight randomness
- **Forces:** Turbulence for flickering, slight upward wind
- **Size:** Starts small, grows moderately
- **Color:** Red at base → orange → yellow at top → transparent
- **Lifetime:** 1-2 seconds
- **Blending:** Additive for glow
- **Texture:** Soft gradient or flame texture

**Audio configuration:**
- **Sound component:** Fire crackling loop
- **Volume:** Medium (0.5-0.7)
- **Loop:** Enabled (continuous)
- **3D positioning:** Enabled for spatial audio

**Result:** Rising, flickering flames with realistic turbulence and color variation, accompanied by continuous crackling sounds.

### Smoke

Smoke from fire, explosions, chimneys, or engines:

**Configuration:**
- **Emitter:** Point or box at smoke source
- **Emission:** Continuous, medium rate (20-50 particles/second)
- **Initial velocity:** Slow upward drift (1-3 units/second)
- **Forces:** Wind for directional drift, turbulence for natural movement
- **Size:** Starts small, grows significantly over lifetime (3-5x initial size)
- **Color:** Gray or white, fades to transparent slowly
- **Lifetime:** 3-5 seconds
- **Blending:** Alpha with soft particles (depth fade)
- **Texture:** Soft, irregular smoke puff

**Visual result:** Slowly rising, expanding clouds of smoke that dissipate naturally.

### Sparks

Metal sparks, electrical arcs, or magic sparkles:

**Configuration:**
- **Emitter:** Point with wide spray angle
- **Emission:** Short bursts or continuous low rate
- **Initial velocity:** High speed with gravity
- **Forces:** Strong gravity (particles arc and fall)
- **Size:** Small and constant
- **Color:** Bright yellow/white, fades quickly
- **Lifetime:** 0.5-1 second
- **Blending:** Additive
- **Trails:** Motion trails enabled for streak effect
- **Texture:** Bright point or small star

**Visual result:** Bright arcing streaks that fall and fade quickly, like real sparks.

### Rain

Falling rain for weather effects:

**Configuration:**
- **Emitter:** Large horizontal plane above camera
- **Emission:** High continuous rate (200-1000 particles/second)
- **Initial velocity:** Straight down, fast (10-20 units/second)
- **Forces:** Slight wind for angled rain
- **Size:** Small vertical streaks
- **Color:** White or light blue, semi-transparent
- **Lifetime:** Based on fall distance
- **Blending:** Alpha
- **Collision:** Particles die on ground impact
- **Texture:** Elongated raindrop or line

**Visual result:** Dense curtains of falling rain that react to wind and disappear on impact.

### Magic Spell Effect

Fantasy-style magical effects:

**Configuration:**
- **Emitter:** Sphere or vortex around caster's hand
- **Emission:** Moderate continuous rate
- **Initial velocity:** Swirling motion (vortex force)
- **Forces:** Vortex for spiral, gentle upward wind
- **Size:** Small particles, slight growth
- **Color:** Bright magical color (blue, purple, green), glowing
- **Lifetime:** 1-2 seconds
- **Blending:** Additive for glow
- **Texture:** Sparkle or ethereal wisp

**Visual result:** Swirling, glowing particles that orbit the caster before dissipating.

## GPU Acceleration and Performance

Traktor's particle system runs entirely on the GPU, which provides massive performance benefits:

**Thousands of particles simultaneously:** Modern GPUs can simulate and render 50,000+ particles at 60fps. CPU systems struggle beyond a few thousand.

**Parallel updates:** All particles update simultaneously across GPU cores. CPU systems update one at a time, sequentially.

**Efficient rendering:** Particles are batched and rendered in a single draw call per effect, minimizing overhead.

**Sorting for transparency:** The system automatically sorts particles for correct alpha blending, ensuring smoke layers properly without flickering.

## Particle Collision

Particles can collide with the world, bouncing off surfaces or dying on impact:

**Collision detection** uses simplified geometry (typically a height map or collision mesh) rather than full world geometry, keeping it fast even with thousands of particles.

**Bounce behavior:** Particles can reflect off surfaces with configurable elasticity (bounciness) and friction.

**Die on collision:** Particles can disappear when they hit a surface. Perfect for rain splashing on the ground or sparks hitting walls.

**Spawn secondary effects:** On collision, particles can trigger other effects. Rain creates splash particles, sparks create impact flashes.

## Trails for Motion Blur

**Motion trails** (also called ribbon trails) connect moving objects to their previous positions, creating streaks. These are configured in the particle effect editor, not via scripting.

Trails are defined by parameters:
- **Width** - How wide the trail ribbon is
- **Age** - How long the trail persists before fading
- **Length threshold** - Minimum distance before creating new trail segments
- **Break threshold** - Distance at which the trail breaks (creating gaps)

Perfect for sparks, tracer bullets, magic projectiles, or fast-moving objects. Trails give motion a sense of speed and energy and are set up as part of your effect asset in the editor

## Best Practices

**Use GPU-accelerated particles for everything.** Don't fall back to CPU particles unless absolutely necessary. GPU particles are almost always faster.

**Limit overdraw.** Large, overlapping transparent particles can kill performance due to overdraw (redrawing the same pixel many times). Use smaller particles or reduce emission rates if frame rate drops.

**Use texture atlases.** Pack multiple particle textures into a single atlas to reduce texture switches and improve batching.

**Tune emission rates.** More particles aren't always better. Often, well-designed particles at lower rates look better and run faster than dense clouds of simple particles.

**Profile on target hardware.** Particle effects that run fine on a high-end PC might struggle on lower-end devices. Test early and often.

**Use LOD (Level of Detail).** Reduce particle counts at distance. Players won't notice fewer particles on distant effects, but the performance savings add up.

**Avoid huge particles.** Very large particles (filling a significant portion of the screen) cause massive overdraw. Break large effects into multiple smaller emitters.

**Sort carefully.** Transparent particles must be sorted for correct blending, which has a cost. Use additive blending where possible (fire, sparks) to avoid sorting.

## Common Patterns

### One-Shot Effect (Explosion, Impact)

```lua
import(traktor)

ExplosionEffect = ExplosionEffect or class("ExplosionEffect", world.ScriptComponent)

function ExplosionEffect:new()
    self._effect = nil
end

function ExplosionEffect:setup()
    self._effect = self.owner:getComponent(spray.EffectComponent)
    self._effect.loopEnable = false  -- Play once
end

function ExplosionEffect:trigger()
    -- Start the effect
    self._effect.enable = true
    self._effect:reset()  -- Reset to beginning
end

function ExplosionEffect:update(contextObject, totalTime, deltaTime)
    -- Auto-destroy entity when effect finishes
    if self._effect.finished then
        self.owner:destroy()
    end
end
```

### Continuous Effect (Fire, Smoke)

```lua
import(traktor)

TorchFire = TorchFire or class("TorchFire", world.ScriptComponent)

function TorchFire:new()
    self._effect = nil
end

function TorchFire:setup()
    self._effect = self.owner:getComponent(spray.EffectComponent)
    self._effect.loopEnable = true  -- Continuous loop
    self._effect.enable = true  -- Start playing
end

function TorchFire:extinguish()
    -- Stop the fire effect
    self._effect.enable = false
end

function TorchFire:relight()
    -- Restart the fire
    self._effect.enable = true
    self._effect:reset()
end
```

### Conditional Effect (Damage Sparks)

```lua
import(traktor)

DamageSparks = DamageSparks or class("DamageSparks", world.ScriptComponent)

function DamageSparks:new()
    self._effect = nil
    self._health = 100
end

function DamageSparks:setup()
    self._effect = self.owner:getComponent(spray.EffectComponent)
    self._effect.loopEnable = true
    self._effect.enable = false  -- Start disabled
end

function DamageSparks:takeDamage(amount)
    self._health = self._health - amount

    -- Show sparks when health is low
    if self._health < 30 then
        self._effect.enable = true
    else
        self._effect.enable = false
    end
end
```

## Debugging Effects

When effects don't work as expected:

**No particles appear:**
- Check `effect.enable = true` is set
- Verify the effect resource is loaded correctly
- Check material is assigned in the editor
- Ensure blending mode is correct
- For one-shot effects, call `effect:reset()` to restart

**Effect doesn't loop:** Set `effect.loopEnable = true` in your script.

**Particles disappear immediately:** Lifetime might be too short in the effect configuration, or collision might be killing them prematurely.

**Particles don't move:** Check initial velocity and forces are configured in the editor. Zero velocity = static particles.

**Poor performance:** Too many particles or too much overdraw. Reduce emission rate in the editor, make particles smaller, or use additive blending.

**Wrong sorting:** Transparent particles need proper sorting. Check rendering order and blending modes.

**Effect appears finished but keeps playing:** Check if `effect.loopEnable` is set when it shouldn't be.

Use the editor's effect preview to iterate quickly. You can see and hear changes in real-time without restarting the game. The Lua API is for playback control. Most visual and audio tuning happens in the editor.

**No sound plays:** Verify sound components are added to the effect in the editor, check that the sound resource is loaded, ensure volume is not zero.

**Sound doesn't sync with visuals:** Use sound events with specific timing rather than simple sound components for precise synchronization.

## See Also

- [Render System](render.md) - Effect rendering pipeline
- [Scripting](scripting.md) - Controlling effects from Lua
- [Sound System](sound.md) - Audio system (Spray SoundComponent is specialized for effects)

## References

- Source: `code/Spray/` - Complete effects system (VFX + SFX + sequencing)
