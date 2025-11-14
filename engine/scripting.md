---
layout: default
permalink: /engine/scripting/
title: Scripting
parent: Engine

nav_order: 5
---

# Scripting with Lua

Scripting in Traktor is where your game comes to life. While the engine provides systems for rendering, physics, and audio, it's your Lua scripts that define what makes your game unique. The gameplay rules, character behaviors, UI interactions, and everything that makes players say "just one more level."

Traktor uses **Lua 5.x** as its scripting language. If you've never used Lua before, don't worry. It's one of the easiest programming languages to learn, and it's been battle-tested in countless games from World of Warcraft to Angry Birds.

![TODO: Screenshot of the Lua script debugger showing breakpoints, variable inspection, and call stack]

## Why Scripts Matter

Think of C++ as the engine's muscle - it's fast, efficient, and handles the heavy lifting like rendering thousands of polygons or simulating complex physics. Scripts are the brain - they make decisions, respond to player input, and implement game rules that would be tedious to hard-code in C++.

The best part? Scripts hot-reload instantly. Change your code, save the file, and see the results immediately in your running game. No recompilation, no waiting, no losing your place in the level you're testing.

## Quick Reference

### Basic Script Template

```lua
import(traktor)

MyScript = MyScript or class("MyScript", world.ScriptComponent)

function MyScript:new()
    -- Initialize member variables
    self._speed = 10.0
end

function MyScript:update(contextObject, totalTime, deltaTime)
    -- Update logic
end
```

### Key Patterns

These patterns show the correct way to use Traktor's Lua API:

| Pattern | Correct | Incorrect |
|---------|---------|-----------|
| **Import** | `import(traktor)` at top of script | No import (won't have namespaces) |
| Constructor | `function MyScript:new()` | `function MyScript:create()` |
| Update params | `(contextObject, totalTime, deltaTime)` | `(context, dt)` |
| Transform access | `self.owner.transform` | `self.owner:getTransform()` |
| Component access | `:getComponent(physics.CharacterComponent)` | `:getComponent("CharacterComponent")` |
| Constants | `local MAX < const > = 10` | `local MAX = 10` |

## The Foundation: Importing Traktor

Before we dive into writing scripts, there's one crucial line you need at the top of every script file:

**IMPORTANT:** You must use `import(traktor)` at the beginning of your scripts to access engine namespaces.

```lua
import(traktor)  -- Required to access world, physics, render, animation, etc.
```

Without this import, you won't have access to the engine's namespaces like `world`, `physics`, `render`, `animation`, `sound`, `spray`, and many others. It's like trying to use a library without importing it first. The functions simply won't be available.

## How Scripts Attach to Your Game

In Traktor, scripts become part of your game through the **ScriptComponent**. Think of components as Lego bricks that you attach to entities (game objects). A character entity might have a MeshComponent to make it visible, a RigidBodyComponent to make it collide with things, and a ScriptComponent to make it behave intelligently.

Here's how scripts are attached:

```cpp
// C++ - Attach script to entity
Ref<ScriptComponent> scriptComp = new ScriptComponent();
scriptComp->setScript(scriptResource);
entity->setComponent(scriptComp);
```

```lua
-- Lua - Script structure
import(traktor)

Script = Script or class("Script", world.ScriptComponent)

function Script:new()
    -- Called when entity is created (constructor)
    self._speed = 10.0
    self._health = 100
end

function Script:destroy()
    -- Called when entity is destroyed
    -- Cleanup
end

function Script:update(contextObject, totalTime, deltaTime)
    -- Called every frame
    -- contextObject: Update context object
    -- totalTime: Total elapsed time in seconds
    -- deltaTime: Time since last frame in seconds
end
```

## Script Lifecycle

![TODO: Diagram showing script lifecycle: Load → new() → update() loop → destroy() → Unload]

1. **Load** - Script file is loaded
2. **new()** - Constructor, initialization
3. **update()** - Called every frame
4. **destroy()** - Cleanup

## Accessing the Engine

### Entity Access

```lua
-- Get entity transform (accessed as property)
local T = self.owner.transform
local position = T.translation
local direction = T.axisZ  -- Forward direction
local right = T.axisX      -- Right direction
local up = T.axisY         -- Up direction

-- Set entity transform
T.translation = Vector4(10, 0, 0)
self.owner.transform = T

-- Get components (use full type names with namespaces)
local groupComponent = self.owner:getComponent(world.GroupComponent)
local characterComponent = self.owner:getComponent(physics.CharacterComponent)
local skeletonComponent = self.owner:getComponent(animation.SkeletonComponent)
local vehicleComponent = self.owner:getComponent(physics.VehicleComponent)
```

### World Access

World access is typically done through a Stage class context:

```lua
-- In a Stage class update method
function Stage:update(contextObject, totalTime, deltaTime)
    -- Find entity by name
    local playerEntity = self.world.world:getEntity("Player")

    -- Find multiple entities by name
    local npcEntities = self.world.world:getEntities("NPC")

    -- Add/remove entities
    self.world.world:addEntity(entity)
    self.world.world:removeEntity(entity)
end
```

### Input

**IMPORTANT:** Input is only accessible in **Stage** classes. ScriptComponents do NOT have access to input - they receive game state and commands from the Stage.

Input can be handled through InputMapping or raw input in Stage classes:

```lua
-- In a Stage class - Using InputMapping
function GameStage:new(params, environment)
    Stage.new(self, params, environment)

    -- Get input mapping from environment
    self._inputMapping = environment.input.inputMapping
end

function GameStage:update(contextObject, totalTime, deltaTime)
    -- Check button/key states
    local escape = self._inputMapping:isStatePressed("STATE_ESCAPE")
    local jump = self._inputMapping:isStatePressed("STATE_JUMP")

    -- Get analog axis values
    local moveZ = self._inputMapping:getStateValue("MOVE_Z")
    local moveX = self._inputMapping:getStateValue("MOVE_X")
end
```

Or using raw input:

```lua
-- In a Stage class - Using raw input
function GameStage:new(params, environment)
    Stage.new(self, params, environment)

    -- Store input from environment
    self._input = environment.input
end

function GameStage:update(contextObject, totalTime, deltaTime)
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

### Physics

#### Character Component

```lua
-- Get character component
local characterComponent = self.owner:getComponent(physics.CharacterComponent)

-- Move character (horizontal movement + vertical flag)
local moveVector = Vector4(x, 0, z)
characterComponent:move(moveVector, false)  -- false = not vertical movement

-- Jump
characterComponent:jump()

-- Access properties
local headAngle = characterComponent.headAngle
characterComponent.headAngle = newAngle
```

#### Vehicle Component

```lua
-- Get vehicle component
local vehicleComponent = self.owner:getComponent(physics.VehicleComponent)

-- Control vehicle
vehicleComponent.steerAngle = angle
vehicleComponent.engineThrottle = throttle  -- 0.0 to 1.0
vehicleComponent.maxVelocity = maxVelocity
```

#### Rigid Body Component

```lua
-- Get rigid body component
local body = self.owner:getComponent(physics.RigidBodyComponent)

-- Apply force
body:applyForce(Vector4(0, 100, 0))

-- Apply impulse
body:applyImpulse(Vector4(10, 0, 0))

-- Set velocity
body:setLinearVelocity(Vector4(5, 0, 0))
```

### Audio

```lua
-- Get sound component
local sound = self.owner:getComponent(sound.SoundComponent)

-- Play sound
sound:play()
sound:stop()
sound:setVolume(0.5)

-- 3D positional audio
sound:set3D(true)
sound:setPosition(Vector4(10, 0, 0))
```

## Common Patterns

### State Machine

```lua
import(traktor)

Script = Script or class("Script", world.ScriptComponent)

function Script:new()
    self._state = "idle"
end

function Script:update(contextObject, totalTime, deltaTime)
    if self._state == "idle" then
        self:updateIdle(contextObject, totalTime, deltaTime)
    elseif self._state == "moving" then
        self:updateMoving(contextObject, totalTime, deltaTime)
    elseif self._state == "attacking" then
        self:updateAttacking(contextObject, totalTime, deltaTime)
    end
end

function Script:updateIdle(contextObject, totalTime, deltaTime)
    -- Check for state transition
    self._state = "moving"
end

function Script:updateMoving(contextObject, totalTime, deltaTime)
    -- Movement logic
end
```

### Timer

```lua
import(traktor)

Script = Script or class("Script", world.ScriptComponent)

function Script:new()
    self._timer = 0
    self._interval = 2.0  -- 2 seconds
end

function Script:update(contextObject, totalTime, deltaTime)
    self._timer = self._timer + deltaTime
    if self._timer >= self._interval then
        self:onTimer()
        self._timer = 0
    end
end

function Script:onTimer()
    log:info("Timer triggered!")
end
```

### Health System

```lua
import(traktor)

Script = Script or class("Script", world.ScriptComponent)

function Script:new()
    self._maxHealth = 100
    self._health = self._maxHealth
end

function Script:damage(amount)
    self._health = self._health - amount
    if self._health <= 0 then
        self:die()
    end
end

function Script:heal(amount)
    self._health = math.min(self._health + amount, self._maxHealth)
end

function Script:die()
    -- Death logic
    -- Note: World access typically requires context from Stage
    log:info("Entity died!")
end
```

## C++ to Lua Binding

### Exposing C++ Classes

C++ classes are automatically exposed to Lua through **ClassFactory** implementations. Each module defines a ClassFactory that uses `AutoRuntimeClass` to register classes, methods, properties, and constructors.

**Example ClassFactory implementation:**

```cpp
// MyModuleClassFactory.h
#pragma once

#include "Core/Class/IRuntimeClassFactory.h"

namespace mymodule
{

class MyModuleClassFactory : public traktor::IRuntimeClassFactory
{
    T_RTTI_CLASS;

public:
    virtual void createClasses(traktor::IRuntimeClassRegistrar* registrar) const override;
};

}

// MyModuleClassFactory.cpp
#include "Core/Class/AutoRuntimeClass.h"
#include "Core/Class/IRuntimeClassRegistrar.h"
#include "MyModule/MyModuleClassFactory.h"
#include "MyModule/MyClass.h"

namespace mymodule
{

T_IMPLEMENT_RTTI_FACTORY_CLASS(L"mymodule.MyModuleClassFactory", 0, MyModuleClassFactory, IRuntimeClassFactory)

void MyModuleClassFactory::createClasses(IRuntimeClassRegistrar* registrar) const
{
    // Create runtime class wrapper
    auto classMyClass = new AutoRuntimeClass< MyClass >();

    // Add constructor
    classMyClass->addConstructor();

    // Add properties (getter/setter)
    classMyClass->addProperty("name", &MyClass::setName, &MyClass::getName);
    classMyClass->addProperty("value", &MyClass::setValue, &MyClass::getValue);

    // Add read-only property (getter only)
    classMyClass->addProperty("count", &MyClass::getCount);

    // Add methods
    classMyClass->addMethod("doSomething", &MyClass::doSomething);
    classMyClass->addMethod("calculate", &MyClass::calculate);

    // Add constants
    classMyClass->addConstant("MAX_VALUE", Any::fromInt32(100));
    classMyClass->addConstant("DEFAULT_NAME", Any::fromString(L"Default"));

    // Register the class
    registrar->registerClass(classMyClass);
}

}
```

**Class implementation with Lua-compatible methods:**

```cpp
// MyClass.h
#pragma once

#include "Core/Object.h"
#include "Core/Ref.h"

namespace mymodule
{

class MyClass : public traktor::Object
{
    T_RTTI_CLASS;

public:
    MyClass() = default;

    void setName(const std::wstring& name) { m_name = name; }
    std::wstring getName() const { return m_name; }

    void setValue(int32_t value) { m_value = value; }
    int32_t getValue() const { return m_value; }

    int32_t getCount() const { return m_count; }

    int32_t doSomething(int32_t param);
    float calculate(float a, float b);

private:
    std::wstring m_name = L"";
    int32_t m_value = 0;
    int32_t m_count = 0;
};

}
```

**Using the exposed class from Lua:**

```lua
import(traktor)

-- Create instance using constructor
local obj = mymodule.MyClass()

-- Set properties
obj.name = "MyObject"
obj.value = 42

-- Get properties
log:info("Name: " .. obj.name)
log:info("Value: " .. tostring(obj.value))
log:info("Count: " .. tostring(obj.count))

-- Call methods
local result = obj:doSomething(10)
local calculated = obj:calculate(5.0, 3.0)

-- Access constants
local maxValue = mymodule.MyClass.MAX_VALUE
```

**Key points:**
- `addConstructor()` - Exposes constructor to Lua
- `addProperty(name, setter, getter)` - Property with get/set
- `addProperty(name, getter)` - Read-only property
- `addMethod(name, &Class::method)` - Exposes method
- `addConstant(name, value)` - Class constant
- Methods must use types Lua understands (int32_t, float, std::wstring, Ref<>, etc.)
- All classes automatically work with Lua's garbage collection through reference counting

## Debugging

### Integrated Debugger

The editor includes a Lua debugger:

1. Open script in editor
2. Set breakpoints (click line number)
3. Run game
4. Debugger breaks at breakpoints
5. Inspect variables, step through code

**Debugger Features:**
- Breakpoints
- Step over/into/out
- Variable inspection
- Call stack view
- Expression evaluation

### Logging

```lua
-- Log levels
log:info("Information message")
log:warning("Warning message")
log:error("Error message")
log:debug("Debug message")

-- Formatted output
log:info("Player health: " .. tostring(self.health))
```

### Assertions

```lua
-- Runtime check
assert(self.health > 0, "Health must be positive")

-- Conditional logic
if not player then
    log:error("Player not found!")
    return
end
```

## Performance

### Profiling

Use the integrated profiler to measure script performance:

```lua
-- Profile a function
function Script:expensiveOperation(self)
    profiler:begin("ExpensiveOp")
    -- ... code ...
    profiler:end("ExpensiveOp")
end
```

### Optimization Tips

1. **Cache Lookups:**
```lua
-- Bad: lookup every frame
function Script:update(contextObject, totalTime, deltaTime)
    local characterComponent = self.owner:getComponent(physics.CharacterComponent)
end

-- Good: cache in constructor
function Script:new()
    self._characterComponent = self.owner:getComponent(physics.CharacterComponent)
end
```

2. **Avoid Garbage:**
```lua
-- Bad: creates new table every frame
function Script:update(contextObject, totalTime, deltaTime)
    local pos = { x = 10, y = 0, z = 0 }
end

-- Good: reuse vectors
function Script:new()
    self._moveVector = Vector4(0, 0, 0)
end

function Script:update(contextObject, totalTime, deltaTime)
    self._moveVector.x = 10
    self._moveVector.z = 5
end
```

3. **Use Constants:**
```lua
-- Use const annotation for values that don't change
local MAX_SPEED < const > = 100.0
local JUMP_FORCE < const > = 20.0
```

4. **Use C++ for Heavy Work:**
Move performance-critical code to C++ components.

## Complete Examples

**Note:** ScriptComponents do NOT have access to input. Input is only accessible in Stage classes by storing `environment.input` or `environment.input.inputMapping` in the constructor. ScriptComponents respond to game state and commands passed from the Stage.

### Stage Class Example

```lua
import(traktor)

GameStage = GameStage or class("GameStage", world.Stage)

function GameStage:new(params, environment)
    Stage.new(self, params, environment)

    -- Get input mapping from environment
    self._inputMapping = environment.input.inputMapping

    -- Initialize game state
    self._playerEntity = nil
    self._score = 0
end

function GameStage:update(contextObject, totalTime, deltaTime)
    -- Find player if not cached
    if not self._playerEntity then
        self._playerEntity = self.world.world:getEntity("Player")
    end

    -- Check for escape
    if self._inputMapping:isStatePressed("STATE_ESCAPE") then
        -- Exit game
        log:info("Escape pressed, exiting...")
    end
end
```

## Custom Scripts (Non-Component Classes)

Not all scripts need to be components or stages. You can create custom utility classes for game logic, AI, camera control, and other purposes.

### Camera Controller Example

```lua
import(traktor)

-- Custom class that doesn't inherit from ScriptComponent
FollowCamera = FollowCamera or class("FollowCamera")

function FollowCamera:new(cameraEntity, followEntity)
    self._cameraEntity = cameraEntity
    self._followEntity = followEntity
    self:_calculateTransforms(0)
end

function FollowCamera:update(deltaTime)
    self:_calculateTransforms(0.9)
end

function FollowCamera:reset()
    self:_calculateTransforms(0)
end

-- Private method
function FollowCamera:_calculateTransforms(filterCoeff)
    local ToffsetY < const > = Transform(
        Vector4(0, 2.5, 0),
        Quaternion.identity
    )
    local ToffsetZ < const > = Transform(
        Vector4(0, 0, -2),
        Quaternion.identity
    )

    local D < const > = 2

    local T = ToffsetY * self._followEntity.transform * ToffsetZ
    local targetPosition = T.translation
    local optimalPosition = targetPosition - self._followEntity.transform.axisZ * D

    local cameraPosition = self._cameraEntity.transform.translation
    local d = targetPosition - cameraPosition
    local ln = d.length
    d = d / ln

    local position = Vector4.lerp(optimalPosition, targetPosition - d * D, filterCoeff)

    self._cameraEntity.transform = Transform.lookAt(
        position,
        self._followEntity.transform.translation,
        Vector4(0, 1, 0)
    )
end
```

**Usage from another script:**

```lua
import(traktor)

#using \{ID-OF-FOLLOW-CAMERA-SCRIPT}  -- Reference the FollowCamera script

GameStage = GameStage or class("GameStage", Stage)

function GameStage:new(params, environment)
    Stage.new(self, params, environment)

    local cameraEntity = self.world.world:getEntity("Camera")
    local playerEntity = self.world.world:getEntity("Player")

    -- Create instance of custom FollowCamera class
    self._followCamera = FollowCamera(cameraEntity, playerEntity)
end

function GameStage:update(info)
    -- Update custom camera controller
    self._followCamera:update(info.deltaTime)
end
```

### PID Controller Example

```lua
import(traktor)

-- Utility class for PID control (used in AI, vehicle control, etc.)
PID = PID or class("PID")

function PID:new(Kp, Ki, Kd)
    self._Kp = Kp  -- Proportional gain
    self._Ki = Ki  -- Integral gain
    self._Kd = Kd  -- Derivative gain
    self._integral = 0
    self._previousError = 0
end

function PID:evaluate(dt, current, target)
    local error = target - current

    self._integral = self._integral + error * dt
    local derivative = (error - self._previousError) / dt

    self._previousError = error

    return self._Kp * error + self._Ki * self._integral + self._Kd * derivative
end

function PID:reset()
    self._integral = 0
    self._previousError = 0
end
```

**Usage:**

```lua
import(traktor)

#using \{ID-OF-PID-SCRIPT}

VehicleAI = VehicleAI or class("VehicleAI", world.ScriptComponent)

function VehicleAI:new()
    -- Create PID controller for steering
    self._steeringPID = PID(0.8, 0.01, 0.2)
end

function VehicleAI:update(contextObject, totalTime, deltaTime)
    local vehicleComponent = self.owner:getComponent(physics.VehicleComponent)

    -- Calculate desired angle
    local targetAngle = 0.5
    local currentAngle = 0.0

    -- Use PID to smooth steering
    local steerAngle = self._steeringPID:evaluate(deltaTime, currentAngle, targetAngle)
    vehicleComponent.steerAngle = math.clamp(steerAngle, -0.3, 0.3)
end
```

### Game Logic Class Example

```lua
import(traktor)

-- Custom game logic class (not a component)
PlayerLogic = PlayerLogic or class("PlayerLogic")

function PlayerLogic:new(gameContext, index, autoDrive)
    self._gameContext = gameContext
    self._index = index
    self._autoDrive = autoDrive
    self._lap = 0
    self._kph = 0
end

function PlayerLogic:update(stage, info, throttleFactor)
    local kart = self._gameContext:getKart(self._index)
    local vehicleComponent = kart:getComponent(physics.VehicleComponent)
    local rigidBodyComponent = kart:getComponent(physics.RigidBodyComponent)

    -- Calculate speed
    local mps < const > = rigidBodyComponent.body.linearVelocity.length
    self._kph = mps * (60 * 60) / 1000

    -- Auto-drive logic
    if self._autoDrive then
        vehicleComponent.steerAngle = self:calculateSteerAngle()
        vehicleComponent.engineThrottle = 0.8 * throttleFactor
    end
end

function PlayerLogic:getSpeed()
    return math.floor(self._kph)
end

function PlayerLogic:getLap()
    return self._lap + 1
end

function PlayerLogic:calculateSteerAngle()
    -- AI steering logic here
    return 0.0
end
```

### Key Points for Custom Scripts

1. **No ScriptComponent/Stage inheritance:** Custom classes don't need to inherit from engine base classes
2. **Manual instantiation:** You create instances yourself using `ClassName(...)`
3. **Manual update:** You call update methods yourself when needed
4. **Script references:** Use `#using {GUID}` to reference other script files
5. **Flexible signatures:** Methods can have any signature you want (not constrained to component lifecycle)

## Best Practices

1. **Use Correct Class Pattern:** Always use `ClassName = ClassName or class("ClassName", world.ScriptComponent)`
2. **Import Traktor:** Start scripts with `import(traktor)` to access engine namespaces
3. **Use new() Constructor:** Initialize member variables in `new()`, not `create()`
4. **Correct Update Signature:** Use `update(contextObject, totalTime, deltaTime)` with proper parameter names
5. **Cache Component References:** Get components once in `new()`, not every frame
6. **Use Namespaced Types:** Reference components with full namespace (e.g., `physics.CharacterComponent`)
7. **Use Const Annotation:** Mark constants with `< const >` syntax: `local MAX_SPEED < const > = 100.0`
8. **Access Transform as Property:** Use `self.owner.transform` not `self.owner:getTransform()`
9. **Handle Nil:** Always check for nil before using components
10. **Clean Up:** Release resources in `destroy()`
11. **Profile:** Measure before optimizing

## See Also

- [World System](world.md) - Entity and component system
- [Input System](input.md) - Input handling
- [Physics System](physics.md) - Physics integration

## References

- Source: `code/Script/`
- Lua documentation: https://www.lua.org/manual/5.4/
