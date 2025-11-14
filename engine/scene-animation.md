---
layout: default
permalink: /engine/scene-animation/
title: Scene Animation
parent: Engine

nav_order: 10
---

# Scene Animation - Theater System

Imagine a sweeping camera shot that glides through your game world, revealing a castle in the distance. Or a dramatic cutscene where the camera circles around two characters locked in conversation. Maybe it's a complex choreographed sequence with doors opening, characters running, and lights fading - all perfectly synchronized to music. This is what Traktor's scene animation system makes possible.

The **Theater module** provides Traktor's scene animation capabilities. It's called "Theater" because it works like directing a play or film: you organize animations into named **acts** (scenes), each act contains multiple **tracks** (one per actor/entity), and each track follows a smooth path with keyframed positions and rotations. When you play an act, all tracks animate simultaneously, creating cinematic sequences that rival pre-rendered cutscenes.

This system is perfect for:
- **Cutscenes:** Intro sequences, story beats, victory celebrations
- **Camera Animation:** Establishing shots, flyovers, tracking shots
- **Choreographed Sequences:** Multiple entities moving in perfect sync
- **Environmental Animation:** Doors opening, platforms moving, lights fading

Unlike skeletal character animation (which animates bones within a character), scene animation moves entire entities through the world. Think of it as animating the camera, props, and set pieces rather than individual character joints.

## Architecture: Acts, Tracks, and Paths

The Theater system has three main components:

**TheaterComponent** - A world component that manages all acts and handles playback. You add this component to your world (not to individual entities), and it controls all scene animations:

```cpp
class TheaterComponent : public world::IWorldComponent
{
    bool play(const std::wstring& actName);
    void stop();
    bool isPlaying() const;
};
```

**Act** - A named animation sequence with a defined duration. Each act can contain multiple tracks that play simultaneously:

```cpp
class Act : public Object
{
    const std::wstring& getName() const;
    float getStart() const;    // Always 0
    float getEnd() const;      // Duration in seconds
};
```

Think of acts like video clips: "INTRO" might be a 5-second opening shot, "BOSS_ENTER" might be a 10-second dramatic entrance, and "VICTORY" might be an 8-second celebration sequence.

**Track** - A single entity's animation within an act. Each track specifies which entity to animate and the path it should follow:

```cpp
class Track : public Object
{
    const Guid& getEntityId() const;         // Entity to animate
    const Guid& getLookAtEntityId() const;   // Optional: entity to face
    const TransformPath& getPath() const;    // Animation path
};
```

An act with three tracks might animate:
- Track 1: Camera following a spline path
- Track 2: Character running along a path
- Track 3: Door rotating open

All three animate simultaneously, perfectly synchronized.

## TransformPath: Smooth Keyframed Animation

The heart of Theater is the `TransformPath` class, which provides spline-based interpolation between keyframes. This is what makes animations smooth - you set keyframes at specific times, and the path automatically interpolates between them using TCB (Tension-Continuity-Bias) splines.

### Keyframe Structure

Each keyframe captures position and orientation at a specific time:

```cpp
struct TransformPath::Key
{
    float T;                    // Time (seconds)
    Vector4 tcb;                // Tension/Continuity/Bias for TCB spline
    Vector4 position;           // Position in world space
    Vector4 orientation;        // Orientation (quaternion-like vector)
    float values[4];            // Custom values (not used by Theater)

    Transform transform() const;  // Convert to Transform
};
```

**TCB parameters** control how the spline curves between keyframes:
- **Tension (X):** How tight the curve is (0 = smooth, positive = tight, negative = loose)
- **Continuity (Y):** How the curve transitions (0 = smooth, positive = sharp corner, negative = overshoot)
- **Bias (Z):** Direction of the curve (0 = balanced, positive = biased toward next key, negative = toward previous key)

For most animations, TCB of (0, 0, 0) produces smooth, natural motion. Adjust these only when you need specific curve shapes.

### TransformPath Methods

```cpp
// Insert keyframe (automatically sorted by time)
size_t insert(const Key& key);

// Evaluate transform at specific time
Key evaluate(float at, bool closed) const;

// Get keyframe indices
int32_t getClosestKey(float at) const;
int32_t getClosestPreviousKey(float at) const;
int32_t getClosestNextKey(float at) const;

// Path measurements
float measureLength(bool closed) const;
float measureSegmentLength(float from, float to, bool closed, int32_t steps = 1000) const;
float estimateTimeFromDistance(bool closed, float distance, int32_t steps = 1000) const;

// Path operations
void split(float at, TransformPath& outPath1, TransformPath& outPath2) const;
TransformPath geometricNormalized(bool closed) const;

// Time bounds
float getStartTime() const;
float getEndTime() const;

// Keyframe access
size_t size() const;
const Key& get(size_t at) const;
void set(size_t at, const Key& k);
```

**The closed parameter** determines whether the path loops back to the beginning. For a circular camera orbit, use `closed = true`. For a one-way camera flythrough, use `closed = false`.

**Path measurement** is useful for constant-speed animation. `measureLength()` tells you the total path length, and `estimateTimeFromDistance()` helps you find what time corresponds to a specific distance along the path. This lets you create animations that move at constant velocity rather than varying speed between keyframes.

Located in: `code/Core/Math/TransformPath.h:33`

## Playing Animations from Lua

From your Lua scripts, you control Theater animations through the TheaterComponent:

### Accessing TheaterComponent

Get the component from the world:

```lua
import(traktor)

-- In your stage's create or update method:
local tc = self.world.world:getComponent(theater.TheaterComponent)
if tc ~= nil then
    -- Component is available
end
```

### Playing an Act

Start playing a named act:

```lua
local tc = self.world.world:getComponent(theater.TheaterComponent)
if tc ~= nil then
    -- Play act named "INTRO"
    local success = tc:play("INTRO")
    if success then
        print("Act started")
    else
        print("Act 'INTRO' not found")
    end
end
```

When you call `play()`, all tracks in that act start animating immediately. The animation runs independently - you don't need to update it manually.

### Checking Playback Status

Check if an act is currently playing:

```lua
local tc = self.world.world:getComponent(theater.TheaterComponent)
if tc ~= nil and tc.playing then
    print("Animation is playing")
else
    print("No animation playing")
end
```

This is crucial for transitioning from cutscenes to gameplay. Wait until `tc.playing` is false before giving the player control.

### Stopping Playback

Stop the currently playing act:

```lua
local tc = self.world.world:getComponent(theater.TheaterComponent)
if tc ~= nil then
    tc:stop()
end
```

Stopping immediately ends the animation and leaves entities at their current positions (doesn't reset to start or jump to end).

### Example: Cutscene State Machine

This example from the Kartong project shows how to use Theater for intro cutscenes with proper state transitions:

```lua
import(traktor)

Game = Game or class("Game", world.IWorldEntityRendererCallback)

function Game:create(environmentClass, environmentObject, worldResourceName)
    -- ... initialization ...

    -- Set initial state to play cutscene
    local tc = self.world.world:getComponent(theater.TheaterComponent)
    if tc ~= nil then
        tc:play("START")
        self._updateFn = Game._updateTheater
    else
        self._updateFn = Game._updateStart
    end
end

function Game:update(info)
    -- Delegate to current update function
    self:_updateFn(info)
end

function Game:_updateTheater(info)
    local tc = self.world.world:getComponent(theater.TheaterComponent)
    if not tc.playing then
        -- Cutscene finished, transition to gameplay
        self._followCamera0:reset()
        self._followCamera1:reset()
        self._updateFn = Game._updateStart
    end
end

function Game:postUpdate(info)
    -- Only update gameplay camera when cutscene is not playing
    local tc = self.world.world:getComponent(theater.TheaterComponent)
    if tc == nil or not tc.playing then
        local dT = info.simulationDeltaTime
        self._followCamera0:update(dT)
        self._followCamera1:update(dT)
    end
end
```

**How this works:**
1. Game starts with `_updateTheater` as the active update function
2. While cutscene plays, `_updateTheater` waits for `tc.playing` to become false
3. When cutscene finishes, it resets cameras and switches to `_updateStart` (gameplay)
4. The `postUpdate` method prevents gameplay cameras from updating during cutscenes

This pattern ensures smooth, glitch-free transitions between cutscenes and gameplay.

## Creating Animations in the Editor

The real power of Theater comes from the visual editor workflow. Here's how to create scene animations:

### 1. Add TheaterComponent to Scene

1. Open your scene in the Scene Editor
2. Select the **World** node in the hierarchy (the root node, not an entity)
3. Right-click and select **Add Component**
4. Choose **Theater Component** from the world components list
5. The component is now part of your scene

The Theater Component is a **world component** (like physics or audio), not an entity component. It manages all scene animations globally.

### 2. Create Acts

1. Select the TheaterComponent in the scene hierarchy
2. In the Properties panel, expand the **Acts** section
3. Click **Add** to create a new act
4. Set the act properties:
   - **Name:** Identifier for the act (e.g., "INTRO", "BOSS_ENTER", "VICTORY")
   - **Duration:** Length of the act in seconds (e.g., 5.0)

**Naming conventions:** Use descriptive names that indicate when the act plays:
- "INTRO" - Opening sequence
- "LEVEL_START" - Level intro
- "BOSS_ENTER" - Boss entrance
- "VICTORY" - Victory celebration
- "CREDITS" - End credits

These names are case-sensitive and must match exactly when you call `tc:play("INTRO")`.

### 3. Create Tracks

1. Expand an act in the Properties panel
2. In the **Tracks** section, click **Add** to create a new track
3. Set the track properties:
   - **Entity ID:** GUID of the entity to animate (use the browse button to select from scene entities)
   - **Look-At Entity ID:** Optional GUID of entity to automatically face (leave blank for manual orientation)
   - **Path:** TransformPath defining the animation

You can add as many tracks as you need - one per animated entity. A complex cutscene might have 10+ tracks for camera, characters, props, and lights.

### 4. Edit Transform Paths

The path editor lets you create smooth keyframed animations:

1. Select a track in the Properties panel
2. Click **Edit Path** to open the path editor
3. Add keyframes by clicking in the timeline at desired times
4. For each keyframe, set:
   - **Time:** When the keyframe occurs (in seconds, e.g., 0.0, 2.5, 5.0)
   - **Position:** World space position (X, Y, Z) - where the entity should be at this time
   - **Orientation:** Rotation (as quaternion or Euler angles) - how the entity should be rotated
   - **TCB:** Tension, Continuity, Bias (start with 0, 0, 0 for smooth motion)

5. The path automatically interpolates between keyframes using TCB spline

**Tips for smooth paths:**
- Start with fewer keyframes and add more only where needed
- Use TCB values of (0, 0, 0) for natural, smooth motion
- Increase tension (first value) to make curves tighter/sharper
- Preview the animation frequently to check interpolation
- Adjust keyframe timing to control animation speed (closer keyframes = faster motion)

### 5. Look-At Feature

The **Look-At Entity ID** creates automatic orientation constraints. When set:
- The entity's **position** follows the animation path (as defined by keyframes)
- The entity's **orientation** is automatically calculated to face the look-at target
- Orientation keyframes in the path are ignored (position keyframes still used)

**Use cases:**
- **Camera tracking a character:** Camera path swoops around while always facing the character
- **Character maintaining eye contact:** Two characters walk along paths while facing each other
- **Spotlight following a target:** Light moves on a path while pointing at a moving object
- **Turret tracking player:** Defensive turret rotates to track player position

**How it works:** Every frame, the Theater system calculates the direction from the animated entity to the look-at target and sets the entity's rotation to face that direction. This happens after path interpolation, so the position comes from the path, but rotation comes from look-at.

Example: Camera circling around a character:
1. Create a circular path around the character's position
2. Set Look-At Entity ID to the character
3. Camera follows the circular path while always facing the character (creating an orbit shot)

## Data Structure Example

Here's what Theater animation data looks like in your scene file. This example from the Kartong project shows a 5-second camera animation:

```xml
<worldComponents>
    <item type="traktor.theater.TheaterComponentData">
        <acts>
            <item type="traktor.theater.ActData">
                <name>START</name>
                <duration>5</duration>
                <tracks>
                    <item type="traktor.theater.TrackData">
                        <!-- Camera entity -->
                        <entityId>{485FEF1B-7170-F04A-915A-312559ED69CF}</entityId>
                        <!-- No look-at (manual orientation) -->
                        <lookAtEntityId>{00000000-0000-0000-0000-000000000000}</lookAtEntityId>
                        <path>
                            <keys>
                                <!-- Start keyframe at time 0 -->
                                <item>
                                    <T>0</T>
                                    <tcb>0 0 0 0</tcb>
                                    <position>-50 10 0 1</position>
                                    <orientation>0 0 0 1</orientation>
                                </item>
                                <!-- End keyframe at time 5 -->
                                <item>
                                    <T>5</T>
                                    <tcb>0 0 0 0</tcb>
                                    <position>0 5 -20 1</position>
                                    <orientation>0 0.3 0 1</orientation>
                                </item>
                            </keys>
                        </path>
                    </item>
                </tracks>
            </item>
        </acts>
    </item>
</worldComponents>
```

**What this does:** Animates the camera from position (-50, 10, 0) to (0, 5, -20) over 5 seconds, with a slight rotation change. The camera smoothly interpolates between these two keyframes using TCB splines.

## Common Use Cases

### 1. Intro Cutscene

Animate the camera from an establishing shot to the gameplay view:

```lua
function Game:create(...)
    local tc = self.world.world:getComponent(theater.TheaterComponent)
    if tc ~= nil then
        tc:play("INTRO")
        self._waitingForCutscene = true
    end
end

function Game:update(info)
    local tc = self.world.world:getComponent(theater.TheaterComponent)
    if tc ~= nil and tc.playing then
        -- Still in cutscene, don't process gameplay
        return
    end

    if self._waitingForCutscene then
        -- Cutscene just finished, do one-time setup
        self._waitingForCutscene = false
        self:startGameplay()
    end

    -- Normal gameplay update
    self:updateGameplay(info)
end
```

### 2. Multiple Cutscenes

Use different acts for different sequences:

```lua
function Game:playCredits()
    local tc = self.world.world:getComponent(theater.TheaterComponent)
    if tc ~= nil then
        tc:play("CREDITS")
    end
end

function Game:playVictory()
    local tc = self.world.world:getComponent(theater.TheaterComponent)
    if tc ~= nil then
        tc:play("VICTORY")
    end
end

function Game:playBossEntrance()
    local tc = self.world.world:getComponent(theater.TheaterComponent)
    if tc ~= nil then
        tc:play("BOSS_ENTER")
    end
end
```

Each act is completely independent - different duration, different tracks, different animations. You can have dozens of acts in one scene for different story beats.

### 3. Synchronized Entity Animation

Create complex choreographed sequences with multiple tracks in one act:

**Act: "BOSS_ENTER" (Duration: 10 seconds)**
- Track 1: Camera - Dramatic sweep from hero to boss entrance
- Track 2: Boss entity - Walks through door and stops at center
- Track 3: Door entity - Opens slowly (rotation animation)
- Track 4: Light entity - Fades in behind boss (position/intensity changes)
- Track 5: Hero entity - Turns to face boss

All five tracks play simultaneously, perfectly synchronized. You set up timing relationships by adjusting keyframe times - if the boss should start walking at 2 seconds, put that track's first keyframe at T=2.0.

### 4. Camera Following Character

Create a tracking shot where the camera follows a moving character:

1. Create a track for the camera with a path (e.g., side-scrolling alongside character)
2. Create a track for the character with their movement path
3. Set the camera track's **Look-At Entity ID** to the character
4. Camera will follow its path while always facing the character

This creates cinematic shots like:
- Orbit shot (camera circles while looking at stationary character)
- Tracking shot (camera moves alongside character, both following paths)
- Reveal shot (camera reveals character from behind, both moving)

## Best Practices

**Plan your timeline before creating keyframes.** Sketch out the sequence on paper or in your head: "0-2s: camera sweeps from left, 2-5s: camera holds on character, 5-8s: camera pulls back." Then create keyframes to match your plan.

**Use descriptive act names.** "INTRO", "BOSS_ENTER", "VICTORY" are immediately clear. "ACT1", "ACT2", "ACT3" are confusing and forgettable.

**Keep acts focused.** One act per sequence makes them easier to manage and reuse. Don't put your entire game's cutscenes in one giant act.

**Test frequently.** Play back animations often during creation. It's easier to fix a bad keyframe now than after you've added 20 more.

**Use look-at sparingly.** While convenient, look-at constrains orientation and limits control. For complex camera moves, manual orientation gives better results.

**Start with TCB (0, 0, 0).** This produces smooth, natural motion. Only adjust TCB if the default interpolation looks wrong.

**Keyframe spacing affects speed.** Closer keyframes = faster motion between them. Wider spacing = slower motion. Use this to create natural-looking speed changes.

**Consider performance.** Each track updates one entity's transform per frame. 50 simultaneous tracks might impact performance. Most scenes need fewer than 10 tracks.

**Preview from multiple angles.** View your animation from different perspectives in the editor to catch issues you might miss from one view.

**Use the game camera view.** Preview animations from the actual game camera (if it's not the animated entity) to see what players will see.

## Performance Notes

- **Spline evaluation** is fast but not free - each track evaluates a TCB spline per frame
- **Path complexity** doesn't matter much - 2 keyframes vs 20 keyframes has minimal impact
- **Number of tracks** is the main performance factor - more tracks = more transform updates
- **Look-at calculations** add minimal overhead - just a vector subtraction and quaternion creation
- **Typical performance:** 10-20 simultaneous tracks should have negligible impact (<0.1ms)

For performance-critical scenes (e.g., gameplay with background animations), limit the number of active tracks. Consider stopping acts when they're not visible.

## Lua API Reference

### TheaterComponent

```lua
-- Access from world
local tc = world:getComponent(theater.TheaterComponent)

-- Methods
local success = tc:play(actName)  -- Play named act, returns boolean (true if act exists)
tc:stop()                          -- Stop current act immediately
local playing = tc.playing         -- Property: true if an act is currently playing
```

**Example usage:**

```lua
import(traktor)

FrontEnd = FrontEnd or class("FrontEnd", world.Stage)

function FrontEnd:create(environmentClass, environmentObject, worldResourceName)
    world.Stage.create(self, environmentClass, environmentObject, worldResourceName)

    -- Play intro animation
    local tc = self.world.world:getComponent(theater.TheaterComponent)
    if tc ~= nil then
        local success = tc:play("START")
        if not success then
            log:error("Failed to play START act - act not found")
        end
    else
        log:warning("TheaterComponent not found in world")
    end
end

function FrontEnd:update(info)
    -- Check if still playing
    local tc = self.world.world:getComponent(theater.TheaterComponent)
    if tc ~= nil and not tc.playing then
        -- Animation finished, transition to next state
        self:gotoMainMenu()
    end
end
```

## Extending Theater

While the existing Theater system handles most use cases, you can extend it:

**Custom path types:** Inherit from TransformPath to create paths with different interpolation (e.g., Catmull-Rom, Bezier, linear)

**Parameter animation:** Use the `values[]` array in TransformPath::Key for custom parameters (e.g., FOV, light intensity, shader parameters)

**Script-driven keyframes:** Dynamically modify paths at runtime by adding/removing keyframes programmatically

**Event triggers:** Add custom logic that fires at specific times during playback (e.g., play sound effect when reaching keyframe 3)

## Troubleshooting

**Act doesn't play:**
- Check act name matches exactly (case-sensitive)
- Verify TheaterComponent exists in world
- Check that act has at least one track
- Verify entity GUIDs in tracks are valid

**Animation looks jittery:**
- Too few keyframes - add more for complex motion
- TCB values too extreme - try (0, 0, 0) first
- Keyframes too close together - spread them out

**Entities don't move:**
- Verify entity GUIDs are correct
- Check that entity isn't controlled by other systems (physics, scripts)
- Ensure act is actually playing (`tc.playing == true`)

**Look-at doesn't work:**
- Verify look-at entity GUID is valid and entity exists
- Check that look-at entity is not the same as animated entity
- Ensure both entities are in the same world

## See Also

- [World System](world.md) - Entity and component system
- [Scripting](scripting.md) - Lua scripting guide
- [Animation](animation.md) - Skeletal character animation (different from scene animation)
- [Render](render.md) - Camera and rendering system

## References

- Source: `code/Theater/TheaterComponent.h:31`
- TransformPath: `code/Core/Math/TransformPath.h:33`
- Act: `code/Theater/Act.h:30`
- Track: `code/Theater/Track.h:29`
- Sample: `build/kartong` (uses Theater for intro cutscenes)
