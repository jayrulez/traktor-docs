---
layout: default
permalink: /getting-started/tutorial-02-first-script/
title: "Tutorial 02: Add Your First Script"
parent: Getting Started
nav_order: 5
---

# Tutorial 02: Add Your First Script

Now that you have a scene with a cube, let's bring it to life with code. In this tutorial, you'll create a Lua script that rotates the cube, teaching you the basics of scripting in Traktor.

## About Lua Scripting

Traktor uses **Lua** as its scripting language for gameplay logic. Think of C++ as the engine's muscle - it's fast, efficient, and handles the heavy lifting like rendering and physics. Scripts are the brain - they make decisions, respond to player input, and implement game rules.

The best part? Scripts hot-reload instantly. Change your code, save the file, and see the results immediately in your running game. No recompilation, no waiting, no losing your place.

Lua is one of the easiest programming languages to learn, and it's been battle-tested in countless games from World of Warcraft to Angry Birds. Even if you've never programmed before, you'll find Lua approachable.

---

## Step 1: Add a Script Component to the Cube

First, let's attach a script component to your Cube entity so it can execute code.

1. **Select the Cube** - In the Scene Editor's Entities panel, click on your "Cube" entity under the Objects layer
2. **Add Script Component** - Right-click the "Cube" entity and select **Add Component**, then choose **ScriptComponentData** under the **world** category

![TODO: Screenshot showing ScriptComponentData in the component selector dialog under world category]

The Script Component is now attached, but it doesn't have any script to run yet. Let's create one.

---

## Step 2: Create a New Script

Scripts in Traktor are assets stored in the Database, just like meshes and textures.

1. **Find the Scripts group** - In the Database panel on the left, expand **Source** and you'll see a **Scripts** group
2. **Create a new script** - Right-click the **Scripts** group and select **New Instance**
3. **Choose Script type** - In the dialog that opens:
   - Select **Script** category on the left
   - Select **Script** in the content area
   - Click **OK**

![TODO: Screenshot showing New Instance dialog with Script category and Script type selected]

4. **Name the script** - Enter "RotationScript" as the name and press Enter

You should now see "RotationScript" in the Scripts group.

---

## Step 3: Open the Script Editor

Double-click **RotationScript** in the Database to open it in the script editor.

![TODO: Screenshot showing the script editor with empty script file]

The script editor will open with an empty script file. This is where you'll write your Lua code.

---

## Step 4: Write the Rotation Script

Now let's write a simple script that rotates the cube. Copy this code into the script editor:

```lua
import(traktor)

RotationScript = RotationScript or class("RotationScript", world.ScriptComponent)

function RotationScript:new()
    -- Initialize rotation speed (degrees per second)
    self._rotationSpeed = 45.0
end

function RotationScript:update(contextObject, totalTime, deltaTime)
    -- Get the current transform
    local T = self.owner.transform

    -- Calculate rotation amount for this frame
    local rotationAmount = self._rotationSpeed * deltaTime

    -- Create a rotation quaternion around the Y axis
    local rotation = Quaternion.fromAxisAngle(Vector4(0, 1, 0), math.rad(rotationAmount))

    -- Apply the rotation
    T.rotation = rotation * T.rotation

    -- Update the entity's transform
    self.owner.transform = T
end
```

**Save the script** by pressing **Ctrl+S** or going to **File → Save**.

![TODO: Screenshot showing the script editor with the completed RotationScript code]

---

## Understanding the Script

Let's break down what this code does:

**`import(traktor)`** - This is required at the start of every script. It gives you access to engine namespaces like `world`, `physics`, `render`, and more.

**`RotationScript = RotationScript or class(...)`** - This defines your script class. The pattern `ClassName = ClassName or class(...)` is important - it allows hot-reloading to work properly. Your class inherits from `world.ScriptComponent`.

**`function RotationScript:new()`** - This is the constructor, called when the entity is created. Here we initialize `self._rotationSpeed` to 45 degrees per second. Member variables are prefixed with underscore by convention.

**`function RotationScript:update(contextObject, totalTime, deltaTime)`** - This function is called every frame. The `deltaTime` parameter tells you how much time has passed since the last frame (in seconds).

**Inside update():**
- `self.owner.transform` gets the entity's current position, rotation, and scale
- We calculate how much to rotate based on speed and deltaTime
- We create a rotation quaternion around the Y axis (up)
- We apply the rotation and update the transform

---

## Step 5: Assign the Script to the Component

Now that your script is written, you need to tell the Script Component to use it:

1. **Select the Cube** - Click on your "Cube" entity in the Entities panel
2. **Expand ScriptComponentData** - In the Properties panel, click on **ScriptComponentData** to expand its properties
3. **Browse for script** - Click the **Browse** button next to the **Script** property
4. **Navigate to Scripts** - In the Database browser, expand **Source**, then select the **Scripts** group

![TODO: Screenshot showing Database browser with Scripts group expanded showing RotationScript]

5. **Select RotationScript** - Choose **RotationScript** and click **OK**

The script is now assigned to your Cube!

---

## Step 6: See It in Action

Your cube should now be rotating! If the Scene Editor is in Camera view (from Tutorial 01), you'll see the cube spinning. If you're in Perspective view, switch to Camera view to see the runtime result:

1. Click the view dropdown in the Scene Editor toolbar
2. Select **Camera**

The cube should be smoothly rotating around its vertical axis at 45 degrees per second.

![TODO: Screenshot showing the rotating cube in Camera view with the script running]

---

## Experiment

Try modifying the script to see instant changes:

**Change rotation speed:**
```lua
self._rotationSpeed = 90.0  -- Rotate twice as fast
```

**Rotate on a different axis:**
```lua
-- Rotate around X axis (tip forward/backward)
local rotation = Quaternion.fromAxisAngle(Vector4(1, 0, 0), math.rad(rotationAmount))

-- Rotate around Z axis (roll left/right)
local rotation = Quaternion.fromAxisAngle(Vector4(0, 0, 1), math.rad(rotationAmount))
```

**Rotate in the opposite direction:**
```lua
self._rotationSpeed = -45.0  -- Negative value reverses direction
```

Every time you save the script (Ctrl+S), the changes appear immediately in the Scene Editor. This is hot-reloading in action!

---

## What's Next?

Congratulations! You've written your first script and made your cube come alive with movement.

**Try more complex scripts** - Add input handling, physics, or state machines. Check the [Scripting Documentation](../../engine/scripting/) for more examples.

**Learn the editor tools** - Explore the [Script Editor](../../editor/script-editor/) to understand debugging, breakpoints, and profiling.

**Build a game** - Combine scripts, physics, audio, and more to create interactive experiences.

## See Also

- [Scripting](../../engine/scripting/) - Complete Lua scripting reference
- [World System](../../engine/world/) - Understanding entities and components
- [Scene Editor](../../editor/scene-editor/) - Scene editing tools
