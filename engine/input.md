---
layout: default
permalink: /engine/input/
title: Input
parent: Engine

nav_order: 6
---

# Input System - The Bridge Between Player and Game

Input is the conversation between player and game. Every jump, every turn, every action starts with the player pressing a button or moving a stick. Without responsive, reliable input handling, even the most beautiful game feels sluggish and frustrating. Players notice input lag immediately. It breaks the illusion that they're actually controlling the character.

Traktor's input system provides a unified way to handle input from any device: keyboard and mouse for PC players, gamepads for console-style play, and touch screens for mobile devices. More importantly, it handles all the details that make input feel good. Dead zones on analog sticks so slight drift doesn't move your character, delta tracking for smooth mouse look, and frame-by-frame state so you can distinguish between "held down" and "just pressed."

Think of the input system as the translator that turns physical button presses into meaningful game events. Whether a player presses W on the keyboard, pushes up on an analog stick, or taps an on-screen button, you can map all of those to a single "move forward" action. This means players can use whatever control scheme feels natural to them, and you don't have to write separate code for each input device.

## Accessing Input in Your Scripts

**IMPORTANT:** Input is only accessible in **Stage** classes, NOT in ScriptComponents. ScriptComponents receive data from the Stage, but cannot access input directly.

### Using InputMapping (Recommended)

The recommended approach is to use **InputMapping** in Stage classes, which provides a high-level abstraction over raw input devices:

```lua
import(traktor)

GameStage = GameStage or class("GameStage", world.Stage)

function GameStage:new(params, environment)
    Stage.new(self, params, environment)

    -- Get input mapping from environment
    self._inputMapping = environment.input.inputMapping
end

function GameStage:update(contextObject, totalTime, deltaTime)
    -- Check mapped states
    if self._inputMapping:isStatePressed("STATE_JUMP") then
        -- Jump
    end

    -- Get analog values
    local moveX = self._inputMapping:getStateValue("MOVE_X")
    local moveZ = self._inputMapping:getStateValue("MOVE_Z")
end
```

### Using Raw Input (Alternative)

You can also access raw input by storing it from the environment in the constructor:

```lua
function GameStage:new(params, environment)
    Stage.new(self, params, environment)

    -- Store input from environment
    self._input = environment.input
end

function GameStage:update(contextObject, totalTime, deltaTime)
    -- Use stored input member
    -- Check keyboard
    if self._input:isKeyDown("W") then
        -- Move forward
    end

    -- Check mouse
    local mousePos = self._input:getMousePosition()

    -- Check gamepad
    if self._input:isGamepadConnected(0) then
        local stickX = self._input:getAxisValue(0, "LeftStickX")
    end
end
```

From C++, you access the input server directly through the environment:

```cpp
IInputServer* inputServer = environment->getInputServer();
InputState state = inputServer->getState();

// Check keyboard
if (state.isKeyDown(KeyCode::W))
{
    // Move forward
}

// Check mouse
Vector2 mousePos = state.getMousePosition();
if (state.isMouseButtonDown(0))
{
    // Fire weapon
}
```

## InputMapping API

The **InputMapping** provides the following methods for querying input states:

```lua
-- State queries (for buttons/keys)
local isDown = self._inputMapping:isStateDown("STATE_JUMP")          -- Currently held
local isUp = self._inputMapping:isStateUp("STATE_JUMP")              -- Currently not held
local pressed = self._inputMapping:isStatePressed("STATE_JUMP")      -- Just pressed this frame
local released = self._inputMapping:isStateReleased("STATE_JUMP")    -- Just released this frame
local changed = self._inputMapping:hasStateChanged("STATE_JUMP")     -- Changed this frame

-- Analog values (for sticks/axes)
local value = self._inputMapping:getStateValue("MOVE_X")             -- Current value (-1.0 to 1.0)
local prevValue = self._inputMapping:getStatePreviousValue("MOVE_X") -- Previous frame value

-- Other methods
self._inputMapping:reset()                                            -- Reset all states
self._inputMapping:setValue("CUSTOM_STATE", value)                    -- Set custom state value
local idleDuration = self._inputMapping:getIdleDuration()             -- Time since last input
```

**Note:** InputMapping states are defined in the editor and automatically updated by the engine. You only need to query them.

## Input Mapping: Flexibility for Players

Hardcoding keys in your game scripts is a mistake. Different players prefer different control schemes—WASD vs arrow keys, different button layouts, remapped controls for accessibility. **Input mapping** lets you define logical actions ("Jump", "Fire", "Forward") and bind them to physical inputs in configuration files:

```xml
<!-- InputMapping.xdi -->
<InputMapping>
    <Action name="Forward" keys="W,Up" gamepadButton="LeftStickUp"/>
    <Action name="Back" keys="S,Down" gamepadButton="LeftStickDown"/>
    <Action name="Jump" keys="Space" gamepadButton="A"/>
    <Action name="Fire" mouse="0" gamepadButton="RightTrigger"/>
</InputMapping>
```

In your scripts, check for actions instead of specific keys:

```lua
-- Use mapped states in Stage update
if self._inputMapping:isStatePressed("STATE_JUMP") then
    -- Jump (works with any configured input)
end

-- Get analog values
local moveX = self._inputMapping:getStateValue("MOVE_X")
local moveZ = self._inputMapping:getStateValue("MOVE_Z")
```

Now players can remap controls without you changing any code. You can even provide multiple default mappings for different play styles, and let players customize them in an options menu.

## Best Practices

**Use input mapping, not hardcoded keys.** Define actions and map them to inputs. This gives players flexibility and makes your code cleaner.

**Support multiple devices.** Test with keyboard/mouse and gamepad. Many PC players prefer controllers for certain genres. Make sure both work well.

**Provide sensitivity options.** Different players have different preferences. Let them adjust mouse sensitivity and stick sensitivity in options.

**Handle gamepad disconnects gracefully.** Always check if a gamepad is connected before reading input. Display a "reconnect controller" message if a player's controller disconnects mid-game.

**Apply dead zones to analog sticks.** Most controllers have some stick drift. A dead zone around 0.1 to 0.2 prevents this from causing unwanted input.

**Use "just pressed" for discrete actions.** Jumping, shooting, opening menus. These should trigger once per button press, not every frame while held.

**Normalize diagonal movement.** When a player presses W+D simultaneously, the movement vector's length is √2. Normalize it so diagonal movement isn't faster than cardinal movement.

**Respond immediately.** Input should feel instant. Avoid multi-frame delays between pressing a button and seeing the result.

## Debugging Input

When input doesn't work as expected, log the input mapping state:

```lua
-- Debug input mapping states in Stage
log:info("STATE_JUMP pressed: " .. tostring(self._inputMapping:isStatePressed("STATE_JUMP")))
log:info("MOVE_X value: " .. tostring(self._inputMapping:getStateValue("MOVE_X")))
log:info("MOVE_Z value: " .. tostring(self._inputMapping:getStateValue("MOVE_Z")))
```

Common issues:

**Input doesn't register:** Make sure you're checking input in the update loop, not just once. Verify the correct key names and button names.

**Gamepad input is jittery:** Apply a dead zone to analog sticks. Even brand-new controllers have slight drift.

**Mouse look is too sensitive or too slow:** Provide a sensitivity slider. Different players have vastly different preferences.

**Jump triggers multiple times:** Use `wasKeyPressed` instead of `isKeyDown` for discrete actions.

## See Also

- [Scripting](scripting.md) - Using input from Lua
- [Runtime System](runtime.md) - Input server setup

## References

- Source: `code/Input/`
