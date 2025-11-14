---
layout: default
permalink: /engine/animation/
title: Animation
parent: Engine

nav_order: 9
---

# Skeletal Animation - Bringing Characters to Life

Animation is what transforms static 3D models into living, breathing characters. A character standing perfectly still might be technically impressive, but it feels lifeless. Add a subtle breathing animation, a shift of weight, a head turn. Suddenly, they're alive. Animation is the difference between a mannequin and a performance.

Traktor's skeletal animation system gives you the tools to create convincing character motion. Think of skeletal animation like puppetry: a character has an internal "skeleton" of interconnected bones, and you animate those bones. The character's mesh (skin) follows the bones automatically, deforming naturally as the skeleton moves. This is far more efficient and flexible than animating every vertex manually.

Beyond basic playback, Traktor provides **animation state graphs** (state machines that control which animations play and when), **pose controllers** (systems for blending and controlling animations), **inverse kinematics** (procedural animation that solves for bone positions to reach a target), **ragdoll physics** (physics-driven character animation for impacts and falls), **cloth simulation**, and various utility components for path following and procedural motion.

![TODO: Screenshot showing a character with skeleton visible, animation timeline, and state graph editor]

## The Foundation: Animated Mesh and Skeleton

To animate a character, you need two components working together: the **AnimatedMeshComponent** displays the character's visual mesh, and the **SkeletonComponent** manages the underlying bone structure and determines the character's pose each frame.

Setting up an animated mesh is straightforward:

```cpp
// C++ - Create animated mesh
Ref<AnimatedMeshComponent> animMesh = new AnimatedMeshComponent();
animMesh->setMesh(skeletalMeshResource);
entity->setComponent(animMesh);
```

The skeleton component is where the animation magic happens:

```cpp
// C++ - Create skeleton
Ref<SkeletonComponent> skeleton = new SkeletonComponent();
skeleton->setSkeleton(skeletonResource);
skeleton->setPoseController(poseController);
entity->setComponent(skeleton);
```

The **pose controller** determines what pose the skeleton is in at any given moment. Whether that's playing a single animation, blending between multiple animations based on a state graph, or solving IK constraints.

## Controlling Animation

### Simple Animation Playback

For straightforward cases. A spinning prop, a looping idle animation. The **SimpleAnimationController** does exactly what it says. Give it an animation, tell it whether to loop, and it plays:

```cpp
Ref<SimpleAnimationController> controller = new SimpleAnimationController();
controller->setAnimation(animationResource);
controller->setLoop(true);
controller->setPlaybackSpeed(1.0f);
```

### Animation State Graphs: The Choreographer's Blueprint

Real character animation is complex. A player character might have dozens of animations: idle, walk, run, jump, land, attack, hit reactions, death animations. You need to transition smoothly between them based on gameplay conditions. Speed, ground contact, input, health. Managing this with manual code becomes a nightmare fast.

That's what **animation state graphs** solve. Think of a state graph as a choreographer's blueprint: it defines a set of animation states (idle, walk, run) and the rules for transitioning between them (walk when speed > 0.1, run when speed > 5.0). The system handles the transitions, blending smoothly from one animation to the next.

```cpp
Ref<AnimationGraphPoseController> controller = new AnimationGraphPoseController();
controller->setStateGraph(stateGraphResource);
```

A typical state graph structure might look like this:

```
State Graph
├── State: "Idle"
│   ├── Animation: Idle.anim
│   └── Transitions → Walk (when speed > 0.1)
├── State: "Walk"
│   ├── Animation: Walk.anim
│   ├── Transitions → Idle (when speed <= 0.1)
│   └── Transitions → Run (when speed > 5.0)
└── State: "Run"
    ├── Animation: Run.anim
    └── Transitions → Walk (when speed <= 5.0)
```

From your scripts, you drive the state graph by setting parameters and triggering events:

```lua
import(traktor)

AnimationController = AnimationController or class("AnimationController", world.ScriptComponent)

function AnimationController:update(contextObject, totalTime, deltaTime)
    local skeleton = self.owner:getComponent(animation.SkeletonComponent)
    local controller = skeleton:getPoseController()

    -- Set state graph parameters
    local velocity = self:getVelocity()
    local speed = velocity:length()

    controller:setParameter("Speed", speed)
    controller:setParameter("Grounded", self._isGrounded)
end

function AnimationController:jump()
    -- Trigger state changes (called from Stage based on input)
    local skeleton = self.owner:getComponent(animation.SkeletonComponent)
    local controller = skeleton:getPoseController()
    controller:trigger("Jump")
end
```

The state graph reads these parameters and triggers, automatically transitioning between states and blending animations smoothly. You define the logic visually in the editor, not in code.

### Retargeting Animation

Sometimes you have an animation created for one skeleton and want to use it on another. Maybe you have multiple character models with slightly different proportions. **RetargetPoseController** maps animation from one skeleton to another:

```cpp
Ref<RetargetPoseController> controller = new RetargetPoseController();
controller->setSourceSkeleton(sourceSkeleton);
controller->setTargetSkeleton(targetSkeleton);
```

This is powerful for reusing animations across multiple characters.

## Creating State Graphs in the Editor

State graphs are created visually:

1. Create a **StateGraph asset** in the editor
2. Add animation states (idle, walk, run, attack, etc.)
3. Define transitions between states with arrows
4. Set transition conditions (when speed > 0.5, when jump button pressed, etc.)
5. Assign the state graph to an **AnimationGraphPoseController**

The visual editor makes it easy to see the flow of your animation logic at a glance.

## Inverse Kinematics: Solving for Motion

Sometimes you don't want to play back a pre-made animation. You want to solve for a specific goal. **Inverse Kinematics (IK)** is the answer. Instead of specifying every bone rotation, you specify a target position, and IK solves the bone chain to reach that position.

```cpp
// C++ - Create IK component
Ref<IKComponent> ik = new IKComponent();
ik->setTargetJoint("LeftHand");
ik->setTarget(targetPosition);
entity->setComponent(ik);
```

Common uses for IK include:

**Foot placement** keeps feet planted realistically on uneven ground. Even if your character animation assumes flat terrain, IK adjusts the legs so feet contact slopes and stairs naturally.

**Hand targets** let characters reach for specific objects. Whether grabbing a door handle, placing a hand on a wall, or holding a weapon at a precise position, IK solves the arm chain to reach the target.

**Look-at** controls make characters' heads and eyes track objects or other characters, adding life and presence.

From Lua, controlling IK is simple:

```lua
-- Lua - IK example for looking at target (in update function)
local ik = self.owner:getComponent(animation.IKComponent)
local target = self.owner.world:getEntity("Player").transform.translation
ik:setTarget(target)
```

## Ragdoll Physics: Letting Go

When a character is hit by a massive force, falls from a height, or dies, pre-made animations can feel stiff and unconvincing. **Ragdoll physics** turns the character into a physics-driven puppet, letting the physics engine take over. The result is dynamic, reactive motion that looks natural and unique each time.

```cpp
// C++ - Enable ragdoll
Ref<RagdollComponent> ragdoll = new RagdollComponent();
ragdoll->setSkeleton(skeleton);
ragdoll->setRagdollData(ragdollData);
entity->setComponent(ragdoll);

// Activate ragdoll
ragdoll->setEnabled(true);
```

Ragdolls are perfect for character deaths, impacts and collisions, explosions, and any situation where you want physics to drive character motion. The character's bones become rigid bodies connected by joints, tumbling and reacting realistically to forces and collisions.

## Beyond Characters

Animation isn't just for characters. Traktor includes several components for other types of motion:

### Cloth Simulation

The **ClothComponent** simulates realistic cloth physics for capes, flags, banners, and clothing:

```cpp
Ref<ClothComponent> cloth = new ClothComponent();
cloth->setMesh(clothMesh);
entity->setComponent(cloth);
```

Cloth responds to wind, gravity, and character movement, adding dynamic detail to your scenes.

### Path Following

The **PathComponent** animates entities along spline paths. Perfect for camera movements, moving platforms, vehicles on rails, or any object that needs to follow a predefined curve:

```cpp
Ref<PathComponent> path = new PathComponent();
path->setPath(pathResource);
path->setDuration(5.0f);  // Time to traverse path
entity->setComponent(path);
```

### Procedural Motion Components

For simple procedural animations, Traktor provides several utility components:

**RotatorComponent** creates continuous rotation. Spinning fans, rotating platforms, orbiting objects:

```cpp
Ref<RotatorComponent> rotator = new RotatorComponent();
rotator->setAxis(Vector4(0, 1, 0));  // Rotate around Y axis
rotator->setSpeed(1.0f);  // Radians per second
entity->setComponent(rotator);
```

**PendulumComponent** creates swinging pendulum motion. Hanging lights, chains, bells.

**WobbleComponent** adds shake and wobble effects. Jelly-like objects, earthquake effects.

**OrientateComponent** smoothly orients an entity toward a target. Turrets tracking players, eyes following movement.

### Flocking and Swarms

The **BoidsComponent** simulates flocking behavior for groups of entities. Birds, fish schools, insects. Individual entities follow simple rules (stay close to neighbors, match their direction, avoid collisions), creating emergent, organic group motion:

```cpp
Ref<BoidsComponent> boids = new BoidsComponent();
entity->setComponent(boids);
```

## Animation Data Structures

Under the hood, animations are sequences of keyframed poses. The **Animation** class stores these key poses and interpolates between them:

```cpp
// Animation structure
class Animation
{
    struct KeyPose
    {
        float at;      // Time
        Pose pose;     // Skeletal pose
    };

    // Get pose at specific time
    bool getPose(float time, Pose& outPose) const;
};
```

The **Skeleton** class defines the bone hierarchy. Parent-child relationships, bind poses, and joint names:

```cpp
class Skeleton
{
    // Get joint by name
    const Joint* findJoint(const std::wstring& name) const;

    // Get number of joints
    uint32_t getJointCount() const;
};
```

You typically don't work with these classes directly. They're managed by the animation system. But understanding the structure helps when debugging or creating custom animation tools.

## Best Practices

**Use state graphs for complex logic.** Don't try to manage animation states in code. State graphs are visual, easier to debug, and easier to iterate on.

**Optimize bone count.** Every bone has a performance cost. Use the minimum number of bones needed for good deformation. Characters with 50-100 bones are typical; going beyond 200 bones should be carefully considered.

**Use LOD animations.** Distant characters don't need full animation fidelity. Use simpler animations, lower update rates, or even static poses for characters far from the camera.

**Cache controllers.** Don't create new pose controllers every frame. Create them once, store them, and reuse them.

**Use IK sparingly.** IK is more expensive than playing back keyframed animation. Use it where it provides clear benefit. Foot placement, hand targets. Not everywhere.

**Blend with purpose.** Animation blending is powerful but can look floaty if overused. Short, snappy transitions often feel more responsive than long blends.

## See Also

- [World System](world.md) - Animation components
- [Scene Animation](scene-animation.md) - Object/camera animation
- [Physics System](physics.md) - Ragdoll physics

## References

- Source: `code/Animation/`
