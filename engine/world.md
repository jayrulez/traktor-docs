---
layout: default
permalink: /engine/world/
title: World
parent: Engine

nav_order: 3
---

# World System - Building Your Game Universe

Imagine your game as a stage where actors perform. The **World System** is that stage. It's where everything in your game exists and interacts. Whether it's the player character, enemies, power-ups, or environmental objects, they all live in the world.

Traktor uses an **Entity-Component** architecture, which is a fancy way of saying "build complex things from simple, reusable pieces." Think of it like building with Lego: instead of having one giant, custom-molded block for each type of object, you have small,standard pieces that click together in different ways.

![TODO: Diagram showing World containing multiple Entities, each Entity containing multiple Components]

## Understanding the Pattern

The World System has three key players:

**Worlds** are containers. They hold all the entities in a scene and coordinate their updates. When you load a level, you're loading a World. When the game ticks forward one frame, the World updates everything inside it.

**Entities** are game objects. The player, an enemy, a crate, a light source, whatever. But here's the twist: entities themselves are quite simple. They're basically just a position in space (a transform) and a collection of components. An entity without components is just an invisible point floating in your scene.

**Components** are where the magic happens. They're modular pieces of functionality that you attach to entities. Want an entity to be visible? Add a MeshComponent. Want it to be affected by physics? Add a RigidBodyComponent. Want it to think and make decisions? Add a ScriptComponent. Each component adds one capability, and by combining components, you create rich, complex game objects.

This approach. Composition over inheritance. Is incredibly powerful. Instead of writing a new class for every type of enemy or object, you just mix and match existing components in new ways.

## Core Concepts

### World

The `World` class is the container for all entities in your scene:

```cpp
class World : public Object
{
public:
    // Create world
    explicit World(
        resource::IResourceManager* resourceManager,
        render::IRenderSystem* renderSystem
    );

    // Destroy world resources
    void destroy();

    // World components
    void setComponent(IWorldComponent* component);
    IWorldComponent* getComponent(const TypeInfo& componentType) const;
    template<typename T> T* getComponent() const;
    const RefArray<IWorldComponent>& getComponents() const;

    // Entity management
    void addEntity(Entity* entity);
    void removeEntity(Entity* entity);
    bool haveEntity(const Entity* entity) const;

    // Entity queries
    Entity* getEntity(const Guid& id) const;
    Entity* getEntity(const std::wstring& name, int32_t index = 0) const;
    RefArray<Entity> getEntities(const std::wstring& name) const;
    RefArray<Entity> getEntitiesWithinRange(const Vector4& position, float range) const;
    const RefArray<Entity>& getEntities() const;

    // Update all entities
    void update(const UpdateParams& update);
};
```

**World Components** are global systems that operate on the entire world (e.g., physics world, lighting system).

### Entity

Entities are the game objects in your world:

```cpp
class Entity : public Object
{
public:
    // Create entity
    Entity(
        const Guid& id,
        const std::wstring_view& name,
        const Transform& transform,
        const EntityState& state = EntityState(),
        const RefArray<IEntityComponent>& components = RefArray<IEntityComponent>()
    );

    // Destroy entity resources
    void destroy();

    // World
    void setWorld(World* world);
    World* getWorld() const;

    // Identity
    const Guid& getId() const;
    const std::wstring& getName() const;

    // State management
    EntityState setState(const EntityState& state, const EntityState& mask, bool includeChildren);
    const EntityState& getState() const;
    void setVisible(bool visible);
    bool isVisible() const;
    void setDynamic(bool dynamic);
    bool isDynamic() const;
    void setLocked(bool locked);
    bool isLocked() const;

    // Transform
    void setTransform(const Transform& transform);
    Transform getTransform() const;
    Aabb3 getBoundingBox() const;

    // Component management
    void setComponent(IEntityComponent* component);
    IEntityComponent* getComponent(const TypeInfo& componentType) const;
    template<typename T> T* getComponent() const;
    const RefArray<IEntityComponent>& getComponents() const;

    // Update
    void update(const UpdateParams& update);
};
```

**Key Methods:**

- **setState()** - Set entity state with mask and optional children propagation. Returns the previous state.
- **getBoundingBox()** - Get the combined bounding box of all components in entity space

**Entity State Flags:**
- **Visible:** Entity is rendered
- **Dynamic:** Entity can move/change
- **Locked:** Entity cannot be modified (editor only)

### Components

Components add functionality to entities. All components derive from `IEntityComponent`:

```cpp
class IEntityComponent : public Object
{
    virtual void destroy() = 0;
    virtual void setOwner(Entity* owner) = 0;
    virtual void setWorld(World* world) {}
    virtual void setState(const EntityState& state, const EntityState& mask, bool includeChildren) {}
    virtual void setTransform(const Transform& transform) = 0;
    virtual Aabb3 getBoundingBox() const = 0;
    virtual void update(const UpdateParams& update) = 0;
};
```

**Component Lifecycle Methods:**

- **destroy()** - Called when the component is destroyed, cleanup resources here
- **setOwner()** - Called when the component is attached to an entity
- **setWorld()** - Called when the entity is added to or removed from a world (optional, default is empty)
- **setState()** - Called when the entity's state changes (visible, dynamic, locked). Optional, default is empty
- **setTransform()** - Called when the entity's transform is modified
- **getBoundingBox()** - Returns the component's bounding box in entity space
- **update()** - Called every frame to update component logic

## Common Components

### Entity Transform (Built-in)

Every entity has a transform as a built-in property (not a component):

```cpp
// Set position, rotation, scale
Transform transform(
    Vector4(x, y, z, 1.0f),     // Position
    Quaternion::identity(),      // Rotation
    Vector4(1.0f, 1.0f, 1.0f)   // Scale
);
entity->setTransform(transform);

// Get transform
Transform current = entity->getTransform();

// Transform is part of Entity itself, not a component
// Entity stores it directly and notifies all components when it changes
// via IEntityComponent::setTransform()
```

### Mesh Component

Renders 3D geometry. MeshComponent is an abstract base class - you typically use concrete implementations like StaticMeshComponent:

```cpp
// MeshComponent is abstract - use specific implementations
// Example: StaticMeshComponent for static geometry
// Components are typically created through entity factories
// when loading scenes from the editor
```

### Script Component

Attaches Lua scripts for behavior:

```cpp
// ScriptComponent requires a runtime class proxy and properties
Ref<ScriptComponent> scriptComp = new ScriptComponent(
    clazz,        // resource::Proxy<IRuntimeClass> - the script class
    properties    // PropertyGroup* - script properties
);
entity->setComponent(scriptComp);

// Scripts are typically attached through the editor
// by adding ScriptComponentData to entities
```

### Light Component

Provides illumination:

```cpp
Ref<LightComponent> lightComp = new LightComponent(
    LightType::Directional,        // Light type
    Vector4(1.0f, 1.0f, 1.0f),    // Color
    true,                          // Cast shadow
    1.0f,                          // Near range
    100.0f,                        // Far range
    10.0f,                         // Radius
    0.0f,                          // Flicker amount
    0.0f                           // Flicker filter
);
entity->setComponent(lightComp);

// You can also modify properties after creation
lightComp->setLightType(LightType::Point);
lightComp->setColor(Vector4(1.0f, 0.8f, 0.6f));
lightComp->setCastShadow(false);
```

**Light Types:**
- **Directional:** Sun-like lighting (e.g., for outdoor scenes)
- **Point:** Omnidirectional light source (e.g., light bulb)
- **Spot:** Cone-shaped light (e.g., flashlight)

### Rigid Body Component

Adds physics simulation:

```cpp
// RigidBodyComponent requires a physics Body and event handlers
Ref<RigidBodyComponent> bodyComp = new RigidBodyComponent(
    body,             // Body* - physics body (already configured with mass, shape, etc.)
    eventCollide,     // IEntityEvent* - collision event handler
    0.5f              // transformFilter - transform smoothing factor
);
entity->setComponent(bodyComp);

// The physics Body is typically created through the physics system
// and configured with mass, shape, and other properties before
// being passed to the component
```

### Sound Component

Plays sound effects and music (from Spray module):

```cpp
// SoundComponent requires a SoundPlayer and Sound resource
Ref<SoundComponent> soundComp = new SoundComponent(
    soundPlayer,   // sound::SoundPlayer* - the sound player system
    soundProxy     // resource::Proxy<sound::Sound> - the sound resource
);
entity->setComponent(soundComp);

// Control playback
soundComp->play();
soundComp->stop();
soundComp->setVolume(0.8f);
soundComp->setPitch(1.2f);
soundComp->setParameter(L"filter", 0.5f);
```

**Note:** Components are typically created through entity factories when loading scenes from the editor, rather than being manually constructed in code. The editor provides a user-friendly interface for configuring component properties.

## Working with Entities

### Creating Entities in Code

```cpp
// Create entity
Ref<Entity> entity = new Entity(
    Guid::create(),              // Unique ID
    L"MyEntity",                 // Name
    Transform::identity(),       // Transform
    EntityState::All             // State (visible, dynamic)
);

// Add components
entity->setComponent(new MeshComponent(...));
entity->setComponent(new ScriptComponent(...));

// Add to world
world->addEntity(entity);
```

### Finding Entities

```cpp
// By ID
Ref<Entity> entity = world->getEntity(guid);

// By name
Ref<Entity> entity = world->getEntity(L"Player");

// Multiple entities with same name
RefArray<Entity> enemies = world->getEntities(L"Enemy");

// Within range
RefArray<Entity> nearby = world->getEntitiesWithinRange(
    playerPosition,
    100.0f  // Radius
);
```

### Accessing Components

```cpp
// Get specific component
MeshComponent* mesh = entity->getComponent<MeshComponent>();
if (mesh)
{
    // Use mesh component
}

// Iterate all components
const RefArray<IEntityComponent>& components = entity->getComponents();
for (auto* component : components)
{
    // Process component
}
```

### Modifying Entities

```cpp
// Change transform
Transform t = entity->getTransform();
t.setTranslation(Vector4(10, 0, 0, 1));
entity->setTransform(t);

// Change visibility
entity->setVisible(false);

// Add new component
entity->setComponent(new AudioComponent(...));
```

### Removing Entities

```cpp
world->removeEntity(entity);
```

## Entity Lifecycle

The entity lifecycle follows this sequence (verified from `World.cpp` and `Entity.cpp`):

```
Create → Add to World → Update Loop → Remove from World → Destroy
```

**1. Create** - Entity constructor is called

```cpp
Ref<Entity> entity = new Entity(
    Guid::create(),              // Unique ID
    L"MyEntity",                 // Name
    Transform::identity(),       // Initial transform
    EntityState::All,            // Initial state (visible, dynamic)
    components                   // Optional: components to attach
);
```

Component callbacks: `setOwner()`, `setState()`, `setTransform()`

**2. Add to World** - `world->addEntity(entity)` → `entity->setWorld(world)`

```cpp
world->addEntity(entity);
// Internally calls: entity->setWorld(world)
```

Component callbacks: `setWorld(world)`

**3. Update Loop** - `world->update()` → `entity->update()`

```cpp
// Called every frame by the world
world->update(updateParams);
// Internally: entity->update() for each entity
```

Component callbacks: `update()` each frame

**4. Remove from World** - `world->removeEntity(entity)` → `entity->setWorld(nullptr)`

```cpp
world->removeEntity(entity);
// Internally calls: entity->setWorld(nullptr)
```

Component callbacks: `setWorld(nullptr)`

**5. Destroy** - `entity->destroy()` (MUST be removed from world first!)

```cpp
// IMPORTANT: Entity must be removed from world before destroy
entity->destroy();
// Fatal assertion fails if entity is still in world!
```

Component callbacks: `destroy()`

**Important Notes:**

- **Deferred Add/Remove**: If you add or remove entities during `update()`, the operation is deferred until after all entities have been updated (to avoid iterator invalidation)
- **Must Remove Before Destroy**: `Entity::destroy()` has a fatal assertion `T_FATAL_ASSERT(m_world == nullptr)` - you MUST remove the entity from the world before destroying it
- **World::destroy()** automatically calls `setWorld(nullptr)` and `destroy()` on all entities

### Update Cycle

```cpp
void World::update(const UpdateParams& update)
{
    // Update all entities
    for (Entity* entity : m_entities)
    {
        entity->update(update);
    }
}

void Entity::update(const UpdateParams& update)
{
    // Update all components
    for (IEntityComponent* component : m_components)
    {
        component->update(update);
    }
}
```

**UpdateParams** contains:
```cpp
struct UpdateParams
{
    Object* contextObject;   // Update context object (Stage instance during runtime)
    double totalTime;        // Total time since first update
    double alternateTime;    // Alternative absolute time
    double deltaTime;        // Delta time since last update
};
```

## Entity Hierarchy

Traktor implements entity hierarchies through the **GroupComponent**:

```cpp
class GroupComponent : public IEntityComponent
{
public:
    void addEntity(Entity* entity);
    void removeEntity(Entity* entity);
    void removeAllEntities();

    const RefArray<Entity>& getEntities() const;
    Entity* getEntity(const std::wstring& name, int32_t index) const;
    RefArray<Entity> getEntities(const std::wstring& name) const;
};
```

**How GroupComponent Works:**

1. **Add GroupComponent to parent entity** - The parent entity gets a GroupComponent
2. **Add child entities** - Child entities are added to the GroupComponent using `addEntity()`
3. **Automatic transform propagation** - When the parent entity's transform changes:
   - GroupComponent stores the parent's transform
   - It calculates each child's local transform relative to the parent
   - When parent moves, it applies the new parent transform to each child's local transform
   - Children are updated in world space

4. **State propagation** - When `setState()` is called with `includeChildren = true`, state changes (visible, dynamic, locked) propagate to all children

5. **Bounding box** - GroupComponent computes a combined bounding box of all children in parent space

**Example:**

```cpp
// Create parent entity with GroupComponent
Ref<Entity> parent = new Entity(...);
Ref<GroupComponent> group = new GroupComponent();
parent->setComponent(group);

// Create child entities
Ref<Entity> child1 = new Entity(...);
Ref<Entity> child2 = new Entity(...);

// Add children to the group
group->addEntity(child1);
group->addEntity(child2);

// Add parent to world (children should already be in world)
world->addEntity(parent);

// When parent moves, children move with it automatically
parent->setTransform(Transform(Vector4(10, 0, 0)));
// child1 and child2 are automatically updated in world space

// Query children
Entity* namedChild = group->getEntity(L"ChildName");
const RefArray<Entity>& allChildren = group->getEntities();
```

**Important Notes:**

- **Children must be in the world** - Child entities must be added to the world separately via `world->addEntity()`
- **Local transforms** - GroupComponent maintains the relative transform between parent and children
- **Transform propagation** - Happens automatically when `setTransform()` is called on the parent
- **No multi-level nesting shown** - You can nest groups (add entities with GroupComponents as children), creating deep hierarchies

## Creating Custom Components

### Component Data

First, create a data class for the editor (derives from `IEntityComponentData`):

```cpp
class MyComponentData : public IEntityComponentData
{
    T_RTTI_CLASS;
public:
    // Properties (serialized)
    float m_speed = 1.0f;
    int32_t m_health = 100;

    virtual int32_t getOrdinal() const override;
    virtual void setTransform(const EntityData* owner, const Transform& transform) override;
    virtual void serialize(ISerializer& s) override;
};
```

### Component Runtime

Then create the runtime component (derives from `IEntityComponent`):

```cpp
class MyComponent : public IEntityComponent
{
    T_RTTI_CLASS;
public:
    MyComponent(float speed, int32_t health)
    :   m_speed(speed)
    ,   m_health(health)
    {
    }

    virtual void destroy() override
    {
        // Cleanup
    }

    virtual void setOwner(Entity* owner) override
    {
        m_owner = owner;
    }

    virtual void setWorld(World* world) override
    {
        // Optional: handle world changes
        // Default implementation in base is empty
    }

    virtual void setState(const EntityState& state, const EntityState& mask, bool includeChildren) override
    {
        // Optional: handle state changes
        // Default implementation in base is empty
    }

    virtual void setTransform(const Transform& transform) override
    {
        // Handle transform changes
    }

    virtual Aabb3 getBoundingBox() const override
    {
        return Aabb3();
    }

    virtual void update(const UpdateParams& update) override
    {
        // Update logic here
        Vector4 pos = m_owner->getTransform().translation();
        pos += Vector4(m_speed * update.deltaTime, 0, 0);

        Transform t = m_owner->getTransform();
        t.setTranslation(pos);
        m_owner->setTransform(t);
    }

private:
    Entity* m_owner = nullptr;
    float m_speed;
    int32_t m_health;
};
```

### Component Factory

Register your component so it can be instantiated:

```cpp
class MyComponentFactory : public IEntityComponentFactory
{
    T_RTTI_CLASS;
public:
    virtual Ref<IEntityComponent> createComponent(
        const IEntityComponentData* componentData,
        ...
    ) const override
    {
        const MyComponentData* data = checked_type_cast<const MyComponentData*>(componentData);
        return new MyComponent(data->m_speed, data->m_health);
    }
};
```

## World Components

World components are global systems that operate on the entire world. Unlike entity components (which are attached to individual entities), world components are attached to the World itself and provide world-level functionality:

```cpp
class IWorldComponent : public Object
{
    virtual void destroy() = 0;
    virtual void update(World* world, const UpdateParams& update) = 0;
};
```

**Available World Components:**

**CullingComponent** (World/Entity) - GPU-driven culling system:
- Manages instance rendering and visibility culling
- Builds instance buffers for GPU rendering
- Handles frustum and occlusion culling
- Used internally by world renderer

**EventManagerComponent** (World/Entity) - Entity event management:
- Centralized system for entity events (collision, triggers, etc.)
- Raises and manages entity event instances
- Handles event cancellation

**IrradianceGridComponent** (World/Entity) - Global illumination:
- Stores irradiance grid data for indirect lighting
- Provides ambient lighting information to renderers
- One per world for global GI

**RTWorldComponent** (World/Entity) - Ray tracing support:
- Manages top-level acceleration structure (TLAS) for ray tracing
- Tracks all ray-traced instances in the world
- Updates TLAS when instances move
- Used by ray tracing renderer

**NavMeshComponent** (Ai module) - Navigation mesh:
- Provides navigation mesh for AI pathfinding
- One per world, accessible to all AI entities
- Used by AI pathfinding systems

**TheaterComponent** (Theater module) - Cinematic sequences:
- Plays cinematic "acts" (sequences of scripted events)
- Manages timeline-based world events
- Used for cutscenes and scripted sequences

**Example: Adding a World Component**

```cpp
// Create world with render system and resource manager
Ref<World> world = new World(resourceManager, renderSystem);

// Add a navigation mesh component
Ref<NavMeshComponent> navMesh = new NavMeshComponent(navMeshProxy);
world->setComponent(navMesh);

// Add an irradiance grid for global illumination
Ref<IrradianceGridComponent> irradiance = new IrradianceGridComponent(irradianceGridProxy);
world->setComponent(irradiance);

// Retrieve a world component
NavMeshComponent* navMeshComp = world->getComponent<NavMeshComponent>();
```

**Note:** Most world components are added automatically by the world renderer or through scene data. You rarely need to manually add world components unless building custom rendering or AI systems.

## Best Practices

### 1. Favor Composition

Instead of:
```cpp
class PlayerEntity : public Enemy, public Character, public Interactive { };
```

Use:
```cpp
Ref<Entity> player = new Entity(...);
player->setComponent(new CharacterController());
player->setComponent(new HealthComponent());
player->setComponent(new InventoryComponent());
```

### 2. Single Responsibility Components

Keep components focused:
- ✓ `HealthComponent` - Manages health
- ✓ `DamageComponent` - Handles damage
- ✗ `PlayerComponent` - Does everything

### 3. Use Entity Queries Wisely

```cpp
// Cache results when possible
RefArray<Entity> enemies = world->getEntities(L"Enemy");

// Avoid expensive queries every frame
for (int i = 0; i < 1000; i++)
{
    world->getEntitiesWithinRange(...);  // Slow!
}
```

### 4. Clean Up Properly

```cpp
void MyComponent::destroy()
{
    // Release resources
    m_resource = nullptr;

    // Call base
    IEntityComponent::destroy();
}
```

### 5. Handle Null Components

```cpp
// Always check for null
if (MeshComponent* mesh = entity->getComponent<MeshComponent>())
{
    // Safe to use mesh
}
```

## Performance Tips

### Spatial Queries

Use spatial data structures for efficient entity queries:

```cpp
// Efficient range query
RefArray<Entity> nearby = world->getEntitiesWithinRange(position, radius);
```

### Component Updates

Only update components that need it:

```cpp
virtual void update(const UpdateParams& update) override
{
    if (!m_active)
        return;  // Skip inactive components

    // Update logic
}
```

### Batch Operations

Process similar entities together:

```cpp
// Get all entities with physics
RefArray<Entity> dynamicEntities;
for (Entity* entity : world->getEntities())
{
    if (entity->isDynamic())
        dynamicEntities.push_back(entity);
}

// Process in batch
for (Entity* entity : dynamicEntities)
{
    // Physics update
}
```

## Integration with Other Systems

### With Scripting

```lua
-- From Lua script
local entity = world:getEntity("Player")
local health = entity:getComponent(game.HealthComponent)
health:damage(10)
```

### With Physics

```cpp
// Physics component automatically syncs with entity transform
RigidBodyComponent* body = entity->getComponent<RigidBodyComponent>();
body->applyForce(Vector4(0, 100, 0));  // Entity will move
```

### With Rendering

```cpp
// Mesh component automatically renders at entity position
MeshComponent* mesh = entity->getComponent<MeshComponent>();
mesh->setMaterial(newMaterial);  // Material changes
```

## Debugging

### Entity Inspector

In the editor, select an entity to see:
- Transform
- State flags
- All components
- Component properties

### Logging

```cpp
log::info << "Entity count: " << world->getEntities().size() << Endl;
log::info << "Entity '" << entity->getName() << "' at "
          << entity->getTransform().translation() << Endl;
```

## See Also

- [Runtime System](runtime.md) - Application and stages
- [Physics System](physics.md) - Physics components
- [Scripting](scripting.md) - Lua integration with entities
- [Render System](render.md) - Rendering components

## References

- Source: `code/World/World.h`
- Source: `code/World/Entity.h`
- Source: `code/World/IEntityComponent.h`
- Source: `code/World/IWorldComponent.h`
