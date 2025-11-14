---
layout: default
permalink: /engine/physics/
title: Physics
parent: Engine

nav_order: 4
---

# Physics System

Imagine a crate falling off a ledge, tumbling through the air, and landing with a satisfying crash. Or a player character climbing stairs, feeling each step beneath their feet. Maybe it's a racing car drifting around a corner, tires gripping the road. These moments feel real because of physics simulation.

The Physics System in Traktor simulates realistic physical behavior - gravity, collisions, forces, and constraints. It's what makes objects fall, bounce, collide, and interact in ways that feel natural to players. Without physics, your game world would feel static and lifeless. With it, even simple interactions become believable and engaging.

Traktor supports two physics backends: **Jolt Physics** (modern, high-performance) and **Bullet Physics** (mature, feature-rich). Both backends expose the same Traktor API, so you can switch between them without changing your game code. This means you can prototype with one engine and ship with another, or choose the best engine for your target platform.

## How Physics Works: The Big Picture

Before diving into code, let's understand how physics fits into your game. At its core, the physics system has three main layers:

**PhysicsManager** - The physics world that simulates all bodies and handles queries. Think of it as the universe in which all physics happens. It tracks every object, detects collisions, solves constraints, and steps the simulation forward each frame. The PhysicsManager is created and managed by the PhysicsServer (runtime/IPhysicsServer), which is one of the core runtime servers.

**Body** - Individual physics objects with mass, velocity, and collision shapes. A Body represents anything that participates in physics simulation: a falling crate, a bouncing ball, a static wall. Bodies can be dynamic (simulated by physics), static (never moves), or kinematic (animated by your code). Each Body has properties like mass, friction, and restitution that determine how it behaves when forces are applied or when it collides with other bodies.

**Components** - Connect physics Bodies to world Entities. Components are the bridge between the physics simulation and your game entities. The main components are:
- **RigidBodyComponent** - Attaches a Body to an Entity, synchronizing transforms
- **CharacterComponent** - Character controller for player movement with collision, slopes, and stairs
- **VehicleComponent** - Vehicle simulation with wheels, suspension, and engine physics

This architecture separates physics simulation (Bodies in the PhysicsManager) from game logic (Entities with Components). Your scripts interact with Components, which in turn control Bodies in the physics world.

## Creating Your First Physics Body

Let's start with the basics: creating a physics body. Bodies are created through the PhysicsManager, not directly. Here's how you create a simple dynamic box that will fall under gravity:

```cpp
// Get physics manager (from physics server or world)
PhysicsManager* physics = physicsServer->getPhysicsManager();

// Create body description with shape
Ref<BodyDesc> bodyDesc = new BodyDesc();
bodyDesc->setShape(shapeDesc);  // Collision shape (we'll create this next)

// Create the body
Ref<Body> body = physics->createBody(
    resourceManager,
    bodyDesc,
    L"MyBody"  // Debug tag (optional, useful for debugging)
);

// Enable the body (bodies are created disabled by default)
body->setEnable(true);

// Set mass (0 = static, > 0 = dynamic)
body->setMass(10.0f, Vector4::zero());  // 10 kg, default inertia

// Set kinematic flag if using animated motion
body->setKinematic(false);  // false = fully simulated
```

**What's happening here?** We first create a `BodyDesc` that describes what kind of body we want. Then we ask the PhysicsManager to create the actual Body. The Body starts disabled - this is by design, so you can configure it fully before it participates in simulation. Once enabled, it will start interacting with other bodies and responding to gravity.

The mass determines whether a body is **static** (mass = 0, never moves) or **dynamic** (mass > 0, simulated). Static bodies are perfect for walls, floors, and terrain. Dynamic bodies are for objects that move and react to forces like crates, balls, or debris.

The **kinematic flag** is for special cases where you want to animate a body directly (setting its position each frame) but still have it interact with the physics world. Elevators, moving platforms, and scripted objects are often kinematic.

## Connecting Physics to Entities: RigidBodyComponent

A Body by itself just exists in the physics world. To make it part of your game, you need to attach it to an Entity using a `RigidBodyComponent`. The component synchronizes the Body's transform with the Entity's transform, making the physics simulation visible in your game world.

Here's the RigidBodyComponent constructor:

```cpp
// RigidBodyComponent constructor (from RigidBodyComponent.h)
explicit RigidBodyComponent(
    Body* body,                    // Physics body
    world::IEntityEvent* eventCollide,  // Optional collision event
    float transformFilter          // Transform smoothing (0-1)
);
```

**Usage example:**

```cpp
// Create physics body first
Ref<Body> body = physics->createBody(...);

// Create rigid body component
Ref<RigidBodyComponent> rigidBody = new RigidBodyComponent(
    body,
    collisionEvent,  // or nullptr if you don't need collision events
    0.5f            // transform filter (smooths jittery motion)
);

// Attach to entity
entity->setComponent(rigidBody);

// Access the underlying Body to apply forces, check velocity, etc.
Body* physicsBody = rigidBody->getBody();
```

**Transform filter** is a smoothing parameter (0 to 1) that helps reduce visual jitter when physics runs at a different frequency than rendering. A value of 0 means no smoothing (use the physics transform directly), while 1 means maximum smoothing. Values around 0.5 work well for most objects.

**Important:** In the editor, you don't manually create RigidBodyComponent like this. Instead, you add a `RigidBodyComponentData` to your entity. The component data specifies shape, mass, friction, restitution, and other properties. When the scene loads, the component factory automatically creates the Body and RigidBodyComponent for you. This code is for when you need to create physics bodies dynamically at runtime.

## Applying Forces: Making Things Move

Now that you have a physics body, let's make it move! There are two main ways to apply motion: **forces** (continuous, over time) and **impulses** (instant velocity changes).

### Forces: Continuous Application

Forces are like pushing a shopping cart - the longer you push, the faster it goes. Forces are applied every frame and accumulate over time:

```cpp
// Apply force at a point (continuous, over time)
// Perfect for: engines, thrusters, wind, magnets
body->addForceAt(
    Vector4(0, 1, 0),     // at: world position where force is applied
    Vector4(0, 100, 0),   // force: world direction and magnitude (Newtons)
    false                 // localSpace: false = world space, true = body-local space
);

// Apply torque (rotational force)
// Perfect for: spinning wheels, motors, angular momentum
body->addTorque(Vector4(0, 50, 0), false);
```

Use forces for things that push continuously: rocket engines, propellers, or a character's movement force. The physics engine integrates these forces into velocity over time, giving smooth, realistic motion.

### Impulses: Instant Velocity Changes

Impulses are like kicking a ball - instant velocity change. They're perfect for impacts, jumps, explosions, and collisions:

```cpp
// Apply impulse at a point (instant velocity change)
// Perfect for: explosions, hits, kicks, collisions
body->addImpulse(
    Vector4(0, 1, 0),     // at: world position
    Vector4(10, 0, 0),    // impulse: world direction and magnitude
    false                 // localSpace
);

// Apply linear impulse (affects entire body)
// Perfect for: jumps, launches, bounces
body->addLinearImpulse(Vector4(0, 10, 0), false);

// Apply angular impulse (instant rotation change)
// Perfect for: spinning after being hit
body->addAngularImpulse(Vector4(0, 5, 0), false);
```

**When to use which?** Use forces for continuous effects (engines running, wind blowing). Use impulses for one-time events (explosions, jumps, hits). Impulses feel more immediate and responsive, while forces feel more realistic for sustained motion.

## Working with Velocity

Sometimes you want direct control over how fast something moves. Velocity is the rate of change of position:

```cpp
// Set/get linear velocity (movement speed)
body->setLinearVelocity(Vector4(5, 0, 0));  // 5 units/sec in X direction
Vector4 linVel = body->getLinearVelocity();

// Set/get angular velocity (rotation speed)
body->setAngularVelocity(Vector4(0, 1, 0));  // Rotate around Y axis
Vector4 angVel = body->getAngularVelocity();

// Get velocity at a specific point on the body
// (different points on a rotating body have different velocities)
Vector4 vel = body->getVelocityAt(Vector4(1, 0, 0), false);
```

**Example use case:** Imagine a player kicks a ball. You could apply an impulse, but for precise control, you might directly set the ball's velocity based on the kick direction and strength. Or when a player dies, you might set the ragdoll's initial velocity to match the player's movement velocity for a smooth transition.

## Body Lifecycle and State

Bodies can be enabled/disabled, active/sleeping, and static/dynamic/kinematic. Understanding these states helps you optimize performance and control behavior:

```cpp
// Enable/disable in simulation
body->setEnable(true);   // Add to simulation
bool enabled = body->isEnable();

// Active/sleeping
// Sleeping bodies aren't simulated (performance optimization)
body->setActive(true);   // Wake up the body
bool active = body->isActive();

// Kinematic (animated vs simulated)
body->setKinematic(false);  // false = simulated, true = animated
bool kinematic = body->isKinematic();

// Static check (mass == 0)
bool isStatic = body->isStatic();

// Reset to initial state (position, velocity, etc.)
body->reset();
```

**Performance tip:** The physics engine automatically puts resting bodies to sleep. Sleeping bodies don't participate in simulation, saving CPU time. When you apply a force or move a nearby body, sleeping bodies wake up automatically. You rarely need to manually control active/sleeping state.

## Transform Control

Every body has a transform (position and rotation). You can read and modify it:

```cpp
// Set/get world transform
body->setTransform(Transform(position, rotation));
Transform t = body->getTransform();

// Get center of mass transform
// (different from the body's origin if mass is unevenly distributed)
Transform centerT = body->getCenterTransform();
```

**When to use setTransform:** For kinematic bodies (moving platforms, elevators), you set the transform each frame. For dynamic bodies, you usually apply forces or impulses instead. Setting the transform directly on a dynamic body teleports it, which can cause unrealistic behavior or tunneling through walls.

## Collision Shapes: Defining Physical Form

Every body needs a collision shape that defines its physical boundaries. Traktor supports several primitive shapes, compound shapes, and triangle meshes. The shape you choose dramatically affects performance and behavior.

### Box Shape

Boxes are perfect for crates, walls, buildings, and other rectangular objects. They're fast and stable:

```cpp
Ref<BoxShapeDesc> boxShape = new BoxShapeDesc();
boxShape->extent = Vector4(1.0f, 1.0f, 1.0f);  // Half-extents (1m = 2m box)
boxShape->margin = 0.01f;                       // Collision margin (keeps shapes slightly separated)

bodyDesc->setShape(boxShape);
```

**Extent is half-size:** A box with extent (1, 1, 1) is actually 2x2x2 units. The extent is the distance from the center to each face.

### Sphere Shape

Spheres are the fastest collision shape. Use them for balls, projectiles, or simplified collision:

```cpp
Ref<SphereShapeDesc> sphereShape = new SphereShapeDesc();
sphereShape->radius = 1.0f;
sphereShape->margin = 0.01f;

bodyDesc->setShape(sphereShape);
```

**When to use spheres:** Balls, grenades, simplified character collision. Spheres have the cheapest collision detection and are perfectly symmetric, so they roll smoothly.

### Capsule Shape

Capsules (cylinder with hemispherical ends) are ideal for characters. They don't get stuck on edges like boxes do, and they're stable when standing:

```cpp
Ref<CapsuleShapeDesc> capsuleShape = new CapsuleShapeDesc();
capsuleShape->radius = 0.5f;   // Radius of the cylinder and hemispheres
capsuleShape->length = 2.0f;   // Length of the cylindrical section
capsuleShape->margin = 0.01f;

bodyDesc->setShape(capsuleShape);
```

**Why capsules for characters?** They slide over edges smoothly, don't tip over easily, and approximate human proportions well. Most character controllers use capsules.

### Cylinder Shape

Cylinders work for barrels, cans, or wheels:

```cpp
Ref<CylinderShapeDesc> cylinderShape = new CylinderShapeDesc();
cylinderShape->radius = 1.0f;
cylinderShape->length = 2.0f;
cylinderShape->margin = 0.01f;

bodyDesc->setShape(cylinderShape);
```

### Compound Shape

Combine multiple shapes to create complex objects like a car (box chassis + cylinder wheels):

```cpp
Ref<CompoundShapeDesc> compound = new CompoundShapeDesc();
compound->shapes.push_back(boxShape);
compound->shapes.push_back(sphereShape);

bodyDesc->setShape(compound);
```

**Use compound shapes when:** Your object has multiple distinct parts (like a dumbbell: sphere-cylinder-sphere). Keep compounds simple - too many sub-shapes hurt performance.

### Mesh Shape

Triangle meshes provide exact collision for complex geometry. Use them for static environment geometry (terrain, buildings):

```cpp
Ref<MeshShapeDesc> meshShape = new MeshShapeDesc();
meshShape->mesh = meshResource;  // Mesh resource
meshShape->margin = 0.01f;

bodyDesc->setShape(meshShape);
```

**Warning:** Mesh shapes are expensive for dynamic bodies. Use them only for static geometry (walls, terrain, buildings). For dynamic objects, use primitives or compound shapes instead.

**Collision margin:** The margin is a small buffer (typically 0.01 to 0.04 units) that keeps shapes slightly separated. It improves stability and performance. Without margin, shapes might penetrate slightly and jitter.

**Editor workflow:** In practice, you don't create shapes in code like this. In the editor, you add a `ShapeComponentData` to your entity and configure it visually. At runtime, the component factory creates the shape automatically.

## Character Controller: Player Movement That Feels Good

Walking on stairs, sliding down slopes, jumping over obstacles - character movement is complex. The `CharacterComponent` handles all of this, providing player movement with collision, slope handling, and stair climbing:

```cpp
// CharacterComponent constructor (from CharacterComponent.h)
explicit CharacterComponent(
    PhysicsManager* physicsManager,
    const CharacterComponentData* data,  // Radius, height, step height, etc.
    Body* bodyWide,        // Wide collision shape (normal stance)
    Body* bodySlim,        // Slim collision shape (for crouching/squeezing)
    uint32_t traceInclude, // Collision filter include mask
    uint32_t traceIgnore   // Collision filter ignore mask
);
```

Notice the character uses two bodies: a wide shape for normal movement and a slim shape for crouching or squeezing through tight spaces. The controller automatically switches between them based on your needs.

**Moving the character:**

```cpp
CharacterComponent* character = entity->getComponent<CharacterComponent>();

// Movement (called each frame with input direction)
Vector4 moveDir = Vector4(1, 0, 0);  // World direction from input
character->move(moveDir * speed, false);  // false = don't override vertical velocity

// Jumping (only works when grounded)
if (character->grounded())
{
    character->jump();  // Uses jump impulse from CharacterComponentData
}

// Velocity control
character->setVelocity(Vector4(5, 0, 0));
Vector4 vel = character->getVelocity();

// Head angle (for camera aiming or head bob)
character->setHeadAngle(0.5f);  // Radians
float headAngle = character->getHeadAngle();

// Stop all motion (e.g., when player dies)
character->stop();

// Clear pending impulses (e.g., when respawning)
character->clear();
```

**How movement works:** When you call `move()`, the character controller performs collision sweeps to find where the character can move. It handles slopes (sliding down steep slopes, climbing shallow ones), stairs (automatically stepping up small obstacles), and walls (sliding along them instead of stopping dead).

The `grounded()` check is crucial for jumping - it tells you if the character is standing on solid ground. Without this, players could jump in mid-air, which feels wrong (unless you want double-jumping!).

**Editor setup:** You create CharacterComponent via `CharacterComponentData` in the editor. The data specifies radius, height, step height (how tall stairs can be), jump impulse, and other parameters. At runtime, the factory creates the component with the configured bodies.

## Vehicle Simulation: Racing and Driving

For driving games, Traktor provides a full vehicle simulation with wheels, suspension, engine, and brakes:

```cpp
// VehicleComponent constructor (from VehicleComponent.h)
explicit VehicleComponent(
    PhysicsManager* physicsManager,
    const VehicleComponentData* data,  // Mass, engine, brake, suspension, etc.
    const RefArray<Wheel>& wheels,     // Wheel instances (usually 4)
    uint32_t traceInclude,
    uint32_t traceIgnore
);
```

**Controlling the vehicle:**

```cpp
VehicleComponent* vehicle = entity->getComponent<VehicleComponent>();

// Steering (input from -1 to 1)
vehicle->setSteerAngle(0.5f);  // Radians
float steer = vehicle->getSteerAngle();
float steerFiltered = vehicle->getSteerAngleFiltered();  // Smoothed for wheel visuals

// Throttle (accelerate or reverse)
vehicle->setEngineThrottle(1.0f);  // -1 to 1 (reverse to forward)
float throttle = vehicle->getEngineThrottle();
float torque = vehicle->getEngineTorque();  // Actual torque being applied

// Braking
vehicle->setBreaking(1.0f);  // 0 to 1 (no brake to full brake)
float brake = vehicle->getBreaking();

// Speed limit (for AI or arcade feel)
vehicle->setMaxVelocity(50.0f);  // Maximum speed
float maxVel = vehicle->getMaxVelocity();

// Access wheels (for wheel rotation, suspension visualization, etc.)
const RefArray<Wheel>& wheels = vehicle->getWheels();
```

**How vehicle physics works:** The vehicle component simulates each wheel independently. Each wheel traces down to find the ground, applies suspension forces, calculates tire friction, and transfers engine torque. The result is realistic weight transfer (front dips during braking, rear squats during acceleration), drifting, and vehicle handling that responds to terrain.

**Steering angle filtered** is a smoothed version of the steering input, useful for animating the visual wheel rotation. The actual physics uses the raw steering angle for responsive handling, but visuals benefit from smoothing to avoid jittery wheel animations.

**Editor setup:** Like character controllers, you create VehicleComponent via `VehicleComponentData` in the editor. You configure chassis mass, engine force, brake force, suspension stiffness, wheel positions, and more. The editor provides a visual setup for wheel placement and properties.

## Physics Queries: Asking Questions About the World

Physics simulation isn't just about moving objects - you also need to ask questions: "Is there a wall in front of me?" "What objects are near the explosion?" "Can I see the target?" Traktor provides several query types for these scenarios.

### Raycasting: Is There Something in That Direction?

Raycasting shoots an invisible ray through the world and tells you what it hits. It's perfect for bullet traces, line-of-sight checks, or ground detection:

```cpp
// Ray cast through the world
QueryResult result;
bool hit = physics->queryRay(
    Vector4(0, 10, 0),    // at: ray origin (start position)
    Vector4(0, -1, 0),    // direction: ray direction (normalized)
    100.0f,               // maxLength: how far to check
    QueryFilter(),        // filter: which bodies to include (default = all)
    false,                // ignoreBackFace: ignore back-facing triangles
    result                // output: what we hit
);

if (hit)
{
    Body* hitBody = result.body;           // What did we hit?
    Vector4 hitPos = result.position;      // Where did we hit it?
    Vector4 hitNormal = result.normal;     // Surface normal at hit point
    float hitDist = result.distance;       // How far away was the hit?
    int32_t material = result.material;    // Material ID (for footstep sounds, etc.)
}
```

**Common uses:**
- **Shooting:** Raycast from gun muzzle in aim direction to check for hits
- **Ground detection:** Raycast down from character to find floor distance
- **Line of sight:** Raycast from AI to player to check if visible
- **Laser sights:** Raycast to show where a laser pointer hits

### Shadow Ray: Fast Occlusion Check

Shadow rays are optimized for yes/no questions - "Is there anything between A and B?" They're faster than full raycasts because they don't return detailed hit information:

```cpp
// Check if ray hits anything (no detailed result)
bool occluded = physics->queryShadowRay(
    origin,
    direction,
    maxLength,
    QueryFilter(),
    PhysicsManager::QtAll  // QtStatic, QtDynamic, or QtAll
);
```

**When to use shadow rays:** AI line-of-sight checks (can the enemy see the player?), occlusion queries for graphics (should I render this?), or pathfinding (is this path clear?).

### Sphere Query: What's Nearby?

Find all bodies within a sphere. Perfect for explosion damage, proximity detection, or finding nearby objects:

```cpp
// Find all bodies in sphere
RefArray<Body> bodies;
uint32_t count = physics->querySphere(
    Vector4(0, 0, 0),      // center: sphere center position
    10.0f,                 // radius: how far to check
    QueryFilter(),         // filter: which bodies to include
    PhysicsManager::QtAll, // query types: static, dynamic, or all
    bodies                 // output: bodies found
);

// Process each body found
for (Body* body : bodies)
{
    // Apply explosion force, damage, etc.
}
```

**Example: Grenade explosion:**
```cpp
// Find all bodies within 10 units of explosion
physics->querySphere(explosionPos, 10.0f, QueryFilter(), PhysicsManager::QtAll, bodies);

// Apply force to each body (stronger closer to center)
for (Body* body : bodies)
{
    Vector4 toBody = body->getTransform().translation() - explosionPos;
    float dist = toBody.length();
    float force = 1000.0f * (1.0f - dist / 10.0f);  // Falloff with distance
    body->addLinearImpulse(toBody.normalized() * force, false);
}
```

### Sphere Sweep: Thick Raycast

Sweep a sphere through space - like a raycast but with thickness. Perfect for projectile collision, area checks, or tunnel prevention:

```cpp
// Sweep sphere through space (like thick raycast)
QueryResult result;
bool hit = physics->querySweep(
    Vector4(0, 10, 0),    // at: sphere start position
    Vector4(0, -1, 0),    // direction: sweep direction
    100.0f,               // maxLength: how far to sweep
    1.0f,                 // radius: sphere radius
    QueryFilter(),
    result
);
```

**When to use sweeps:** Projectile collision (sphere sweep from gun to target), continuous collision detection (prevent fast objects from tunneling), or area scanning (check if a space is clear before teleporting).

### Shape Sweep: Custom Shape Movement Prediction

Sweep a body's actual collision shape through space. This is what character controllers use internally to detect collisions before moving:

```cpp
// Sweep body's shape through space
QueryResult result;
bool hit = physics->querySweep(
    body,                  // body: uses body's collision shape
    Quaternion::identity(), // orientation: shape rotation
    Vector4(0, 10, 0),     // at: start position
    Vector4(0, -1, 0),     // direction: sweep direction
    100.0f,                // maxLength: how far to sweep
    QueryFilter(),
    result
);
```

**Use case:** Predict where a body will end up before moving it, or check if a shape fits in a location before spawning an object there.

### Query Filters: Controlling What You Find

Filters control which bodies are included in queries. They use bit masks for flexible filtering:

```cpp
QueryFilter filter;
filter.includeGroup = 0xFFFFFFFF;  // Include all groups (all bits set)
filter.ignoreGroup = 0;            // Ignore no groups
filter.ignoreClusterId = 0;        // Don't ignore any cluster

// Or use convenience constructors
QueryFilter(includeGroup);
QueryFilter(includeGroup, ignoreGroup);
QueryFilter(includeGroup, ignoreGroup, ignoreClusterId);
QueryFilter::onlyIgnoreClusterId(clusterId);
```

**Query Rule:** A body is included if:
```
(body.group & filter.includeGroup != 0) &&
(body.group & filter.ignoreGroup == 0) &&
(body.clusterId != filter.ignoreClusterId)
```

**Practical example:** You want player bullets to hit enemies but not other bullets or the player:
```cpp
// Player group = 0x01, Enemy group = 0x02, Bullet group = 0x04
QueryFilter filter;
filter.includeGroup = 0x02;  // Only include enemies
filter.ignoreGroup = 0x05;   // Ignore player and bullets (0x01 | 0x04)
```

**Cluster IDs** are special: bodies with the same cluster ID never collide with each other. This is perfect for ragdolls (body parts shouldn't collide with each other) or articulated structures (robot arm segments).

## Collision Filtering: Who Collides With Whom?

Beyond query filters, you can control collision between bodies using cluster IDs and user objects:

```cpp
// Cluster ID - bodies in same cluster never collide
body->setClusterId(1);
uint32_t cluster = body->getClusterId();

// User object - attach arbitrary data (for collision callbacks)
body->setUserObject(myGameObject);
Object* userData = body->getUserObject();
```

**Cluster ID use cases:**
- **Ragdolls:** All body parts share the same cluster ID so limbs don't collide with each other
- **Vehicles:** All parts of a car share a cluster ID (chassis, wheels, etc.)
- **Articulated objects:** Robot arms, chains, ropes - parts shouldn't self-collide

**User object** is a way to attach game-specific data to a physics body. When you receive a collision callback, you can retrieve the user object to identify what collided and respond appropriately (e.g., apply damage to the game object).

## Collision Events: Responding to Impacts

Collisions aren't just for physics - they're gameplay events. When a bullet hits an enemy, a player lands on the ground, or objects crash together, you need to respond. Traktor provides collision listeners for this:

```cpp
class MyCollisionListener : public CollisionListener
{
    virtual void notify(const CollisionInfo& info) override
    {
        Body* body1 = info.body[0];        // First body in collision
        Body* body2 = info.body[1];        // Second body in collision
        Vector4 position = info.position;  // Collision point
        Vector4 normal = info.normal;      // Collision normal
        float impulse = info.impulse;      // Collision strength

        // Respond to collision: play sound, spawn particles, apply damage
        if (impulse > 10.0f)
        {
            // Big impact! Play crash sound
        }
    }
};

// Register listener on a specific body
Ref<MyCollisionListener> listener = new MyCollisionListener();
body->addCollisionListener(listener);

// Or register globally on physics manager (receives all collisions)
physics->addCollisionListener(listener);
```

**Collision impulse** tells you how hard the collision was. Light taps have small impulse, crashes have large impulse. Use this to scale sound volume, particle count, or damage.

**Per-body vs global listeners:**
- **Per-body:** Receives collisions for just that body (e.g., player character detects ground contact)
- **Global:** Receives all collisions (e.g., audio system plays impact sounds for any collision)

**RigidBodyComponent collision events:** When you create a RigidBodyComponent, you can pass an `IEntityEvent` for collision events. This is triggered when the body collides, allowing you to respond in your game logic (e.g., Lua scripts receiving collision events).

## Joints: Connecting Bodies Together

Joints constrain how bodies move relative to each other. They're how you build complex mechanical systems: doors, chains, ragdolls, suspension, and more.

### Creating Joints

All joints are created through the PhysicsManager using a joint descriptor:

```cpp
// Create joint using JointDesc subclass
Ref<Joint> joint = physics->createJoint(
    jointDesc,            // JointDesc subclass (defines joint type and properties)
    Transform::identity(), // joint transform (joint anchor point)
    body1,                // first body
    body2                 // second body (or nullptr to connect to world)
);

// Enable the joint
joint->setEnable(true);
```

**Connecting to world:** If body2 is nullptr, the joint connects body1 to the world (an immovable anchor point). This is how you create a door hinge - one body (door) connected to the world.

### Available Joint Types

**AxisJoint** - Single rotational axis (hinge-like). Perfect for wheels, simple hinges.

**BallJoint** - Ball-and-socket, free rotation in all directions. Perfect for ragdoll shoulders/hips.

**ConeTwistJoint** - Cone-constrained rotation. Like a ball joint but limited to a cone angle. Perfect for ragdoll necks, robot arms.

**DofJoint** - 6-DOF (degrees of freedom) constraint with limits. Fully customizable - can lock/limit translation and rotation on all three axes. Perfect for complex mechanical systems.

**HingeJoint** - Hinge with angle limits and optional motor. Perfect for doors, gates, drawbridges.

**Use cases:**
- **Door:** HingeJoint with angle limits (0° to 90° swing)
- **Ragdoll:** BallJoints for shoulders/hips, ConeTwistJoints for elbows/knees
- **Chain:** Multiple BallJoints or DofJoints connecting segments
- **Vehicle suspension:** DofJoint limiting wheel movement to vertical axis

## Physics World Configuration

The physics world itself has global settings that affect all simulation:

```cpp
PhysicsCreateDesc desc;
desc.timeScale = 1.0f;              // Time scale factor (slow motion = 0.5, fast = 2.0)
desc.simulationFrequency = 120.0f;  // Hz (updates per second)
desc.solverIterations = 8;          // Constraint solver iterations (higher = more stable)

physics->create(desc);
```

**Simulation frequency:** Physics runs at a fixed timestep (default 120 Hz = ~8.3ms per step). This ensures stable, reproducible simulation regardless of frame rate. Higher frequencies are more accurate but more expensive. Lower frequencies are faster but less stable.

**Solver iterations:** More iterations produce more accurate constraint solving (joints stay together better, stacks are more stable) but cost more CPU. 8 is a good default. Increase to 16-32 for complex mechanical systems or large stacks.

**Time scale** lets you slow down or speed up time globally. Great for bullet-time effects or debugging.

**Gravity:**

```cpp
physics->setGravity(Vector4(0, -9.81f, 0));  // Earth-like gravity
Vector4 gravity = physics->getGravity();
```

**Gravity constant:** Earth's gravity is about 9.81 m/s². Use this for realistic falling. For low-gravity (moon), use ~1.62. For high-gravity or fast-paced action, increase it. For zero-g space games, use Vector4::zero().

## Performance and Best Practices

Physics simulation can be expensive. Here's how to keep it fast:

**Use simple shapes** - Boxes, spheres, and capsules are much faster than mesh colliders. A box has 6 faces to test, a mesh has thousands. Use primitives whenever possible.

**Static bodies are cheap** - Set mass to 0 for non-moving geometry (walls, floors, terrain). Static bodies don't participate in integration or sleeping, they just exist for collision tests.

**Let bodies sleep** - The engine automatically puts resting bodies to sleep (no simulation cost). Don't continuously wake them up by querying or touching them every frame.

**Cluster IDs** - Use cluster IDs to prevent collision between bodies in the same articulated structure (like ragdolls). This eliminates expensive collision checks between parts that should never collide.

**Query filters** - Use include/ignore groups to filter out irrelevant bodies from queries. Don't raycast against every body if you only care about enemies.

**Compound shapes** - Keep compounds simple (2-4 sub-shapes max). Complex compounds are slower than a single simple shape.

**Mesh shapes** - Only use meshes for static geometry. Never use them for dynamic bodies (it's extremely slow).

**Fixed timestep** - Physics runs at fixed frequency (default 120 Hz) for stable simulation. This decouples physics from rendering, so physics behaves identically at 30 FPS or 300 FPS.

**Collision layers** - Group similar objects and filter collisions. Bullets don't need to collide with other bullets, pickups don't need to collide with each other.

## Monitoring Performance

Keep an eye on physics performance with statistics:

```cpp
PhysicsStatistics stats;
physics->getStatistics(stats);

uint32_t bodyCount = stats.bodyCount;        // Total bodies in world
uint32_t activeCount = stats.activeCount;     // Active (non-sleeping) bodies
uint32_t manifoldCount = stats.manifoldCount; // Contact manifolds (collision points)
uint32_t queryCount = stats.queryCount;       // Queries this frame
```

**What to watch:**
- **Active count:** Keep this low. If all bodies are active all the time, they're not sleeping (performance problem). Make sure bodies can rest.
- **Manifold count:** High manifold counts mean lots of collisions. If it's too high, simplify collision shapes or use fewer dynamic bodies.
- **Query count:** Lots of queries can add up. Cache query results when possible, or use broader filters.

**Performance targets:** On a typical game, you want active bodies under 100-200, and total simulation time under 2-3ms (at 120 Hz frequency). If physics takes more than 5ms, you'll struggle to hit 60 FPS.

## See Also

- [World System](world.md) - Entity and component system
- [Runtime System](runtime.md) - Update loop and servers
- [Scripting](scripting.md) - Controlling physics from Lua

## References

- Source: `code/Physics/PhysicsManager.h`
- Source: `code/Physics/Body.h`
- Source: `code/Physics/World/RigidBodyComponent.h`
- Source: `code/Physics/World/Character/CharacterComponent.h`
- Source: `code/Physics/World/Vehicle/VehicleComponent.h`
- Jolt Physics: https://github.com/jrouwe/JoltPhysics
- Bullet Physics: https://pybullet.org/
