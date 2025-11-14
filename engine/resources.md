---
layout: default
permalink: /engine/resources/
title: Resources
parent: Engine

nav_order: 7
---

# Resource Management - Your Game's Asset Library

Imagine you're building a massive game world with thousands of 3D models, textures, sounds, and scripts. You can't load everything into memory at once. That would consume gigabytes and make startup painfully slow. But you also can't manually track what's loaded, when to load it, and when to unload it. That's a recipe for crashes, hitches, and headaches.

This is where Traktor's **Resource system** shines. Think of it as a smart librarian for your game's assets. You ask for a resource by its unique identifier (GUID), and the system handles all the details: finding it in the database, loading it into memory, caching it so you don't load it twice, and even automatically reloading it when you make changes during development. The system is lazy (resources only load when actually needed), efficient (shared resources are loaded once), and automatic (reference counting handles cleanup).

The resource system manages everything your game needs: meshes, textures, materials, sounds, scripts, animations, and more. It's one of those invisible systems that, when done right, you never think about. It just works.

## Assets vs Resources: The Two-Database System

Traktor uses a clever two-database architecture that separates authoring from runtime:

**Source Database** contains **Assets** - these are what you see and edit in the editor:
- SceneAsset, TextureAsset, MeshAsset, ShaderAsset, etc.
- Can reference external files (.png, .blend, .fbx, etc.)
- Defined in `.Editor` modules and never shipped with your game
- Editable, human-friendly representations

**Output Database** contains **Resources** - these are what your game actually loads:
- SceneResource, TextureResource, MeshResource, ShaderResource, etc.
- Store optimized binary data in database data channels
- Platform-specific and ready for runtime use
- Compact, optimized representations

The **Pipeline** transforms assets into resources. When you edit a texture in Photoshop and save the .png file, the pipeline detects the change, rebuilds the TextureAsset into a TextureResource, and your running game can hot-reload it automatically.

This separation means:
- Mobile/console targets can stream content directly from your PC running the editor
- Hot-reloading works seamlessly - change an asset, see it update in the running game
- Final game packages contain only a single compact database with optimized resources
- Editor tools never pollute runtime code

## The Proxy Pattern: Handles to Resources

At the heart of the resource system is the **Proxy** pattern. A `Proxy<T>` wraps a resource handle that provides reference-counted access to loaded resources.

```cpp
// Declare resource proxy
resource::Proxy<render::Mesh> m_meshResource;

// Create resource ID from GUID
// The GUID identifies the resource in the output database
resource::Id<render::Mesh> meshId(Guid(L"{12345678-1234-5678-1234-567812345678}"));

// Bind the ID to the proxy
// This loads the resource immediately if not already cached
if (resourceManager->bind(meshId, m_meshResource))
{
    // Binding successful - resource is loaded and ready
    render::Mesh* mesh = m_meshResource;
    // Use mesh
}
else
{
    // Binding failed - resource couldn't be loaded
}

// Check if resource is available
if (m_meshResource)
{
    // Resource is loaded and ready to use
}
```

**Important:** Resources are loaded by **GUID** (Global Unique Identifier), not by path. Each asset in the editor has a unique GUID that identifies it. You can copy an asset's GUID by right-clicking it in the Database view and selecting "Copy GUID" or similar.

**Note on type parameters:** Both `Id<T>` and `Proxy<T>` use **runtime resource types**. These are the types your game actually uses at runtime (e.g., `render::Mesh`, `render::Shader`, `render::ITexture`). The types can often be the same: `Id<render::Shader>` binds to `Proxy<render::Shader>`.

This approach has key benefits: **cacheable resources are shared** (only loaded once in memory even if bound multiple times), the proxy automatically handles hot-reloading for you, and reference counting ensures resources are only unloaded when no longer needed.

## The Resource Manager: Your Asset Librarian

The **IResourceManager** is the central hub for loading resources. It tracks what's loaded, manages the cache, and handles hot-reloading during development:

```cpp
class IResourceManager
{
    // Bind resource ID to a proxy
    // The proxy will automatically load the resource when accessed
    template<typename ResourceType, typename ProductType>
    bool bind(const Id<ResourceType>& id, Proxy<ProductType>& outProxy);

    // Reload specific resource (hot-reload)
    bool reload(const Guid& guid, bool flushedOnly);

    // Reload all resources of a specific type
    void reload(const TypeInfo& productType, bool flushedOnly);

    // Unload all resources of a specific type
    void unload(const TypeInfo& productType);

    // Unload externally unused, resident resources
    void unloadUnusedResident();

    // Get resource manager statistics
    void getStatistics(ResourceManagerStatistics& outStatistics) const;

    // Add/remove resource factories
    void addFactory(const IResourceFactory* factory);
    void removeFactory(const IResourceFactory* factory);
};
```

The key method is **`bind()`** which takes a resource `Id` and binds it to a proxy. The proxy will handle loading the resource when you first access it, and it also handles hot-reloading automatically.

### Accessing the Resource Manager

From C++, you get the resource manager through the resource server:

```cpp
// Get resource manager from environment
IResourceManager* resourceManager = environment->getResourceServer()->getResourceManager();

// Create a resource ID from a GUID
resource::Id<render::Mesh> meshId(Guid(L"{12345678-1234-5678-1234-567812345678}"));

// Bind to a proxy
resource::Proxy<render::Mesh> meshProxy;
if (resourceManager->bind(meshId, meshProxy))
{
    // Binding successful
    render::Mesh* mesh = meshProxy;  // Access the resource
}
```

**Getting the GUID:** In the Traktor editor, you can get a resource's GUID by:
1. Right-clicking the asset in the Database view
2. Selecting "Copy GUID" or "Copy ID" from the context menu
3. The GUID will be copied to your clipboard in the format `{xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx}`

From Lua, resources are typically loaded automatically through component data, but you can access them if needed through the context.

## Loading Different Resource Types

The beauty of the resource system is that it works the same way regardless of what you're loading. The pattern is always: create an `Id` with the resource type, bind it to a proxy with the same (or compatible) type, and access when ready.

**Meshes** (3D models):

```cpp
resource::Id<render::Mesh> meshId(Guid(L"{AAAAAAAA-AAAA-AAAA-AAAA-AAAAAAAAAAAA}"));
resource::Proxy<render::Mesh> meshProxy;
resourceManager->bind(meshId, meshProxy);

render::Mesh* mesh = meshProxy;
if (mesh)
{
    // Use mesh for rendering
}
```

**Textures** (images):

```cpp
resource::Id<render::ITexture> textureId(Guid(L"{BBBBBBBB-BBBB-BBBB-BBBB-BBBBBBBBBBBB}"));
resource::Proxy<render::ITexture> textureProxy;
resourceManager->bind(textureId, textureProxy);

render::ITexture* texture = textureProxy;
```

**Shaders** (materials and rendering properties):

```cpp
resource::Id<render::Shader> shaderId(Guid(L"{CCCCCCCC-CCCC-CCCC-CCCC-CCCCCCCCCCCC}"));
resource::Proxy<render::Shader> shaderProxy;
resourceManager->bind(shaderId, shaderProxy);

render::Shader* shader = shaderProxy;
```

**Sounds** (audio files):

```cpp
resource::Id<sound::Sound> soundId(Guid(L"{DDDDDDDD-DDDD-DDDD-DDDD-DDDDDDDDDDDD}"));
resource::Proxy<sound::Sound> soundProxy;
resourceManager->bind(soundId, soundProxy);

sound::Sound* sound = soundProxy;
```

**Scenes** (levels and game worlds):

```cpp
resource::Id<world::SceneResource> sceneId(Guid(L"{EEEEEEEE-EEEE-EEEE-EEEE-EEEEEEEEEEEE}"));
resource::Proxy<world::SceneResource> sceneProxy;
resourceManager->bind(sceneId, sceneProxy);

world::SceneResource* scene = sceneProxy;
```

The pattern is always the same: create an `Id<ResourceType>` from the GUID, bind it to a `Proxy<ResourceType>`, and access when ready. The proxy keeps the resource loaded and automatically handles hot-reloading.

## Resource Caching and Sharing

The resource manager automatically caches loaded resources. When you bind a resource:

1. **Cacheable resources** (determined by the factory) are stored in a shared cache
2. **Multiple bind calls** with the same GUID return handles to the same cached resource
3. **Non-cacheable resources** create exclusive handles for each bind

```cpp
// First bind - loads the resource and caches it
resource::Id<render::Shader> shaderId(Guid(L"{12345678-1234-5678-1234-567812345678}"));
resource::Proxy<render::Shader> shaderProxy1;
resourceManager->bind(shaderId, shaderProxy1);  // Loads from database

// Second bind - returns handle to the same cached resource
resource::Proxy<render::Shader> shaderProxy2;
resourceManager->bind(shaderId, shaderProxy2);  // Reuses cached resource, no load

// Both proxies point to the same shader instance
T_ASSERT(shaderProxy1.getResource() == shaderProxy2.getResource());
```

This means resources are only loaded once, even if multiple systems or entities reference them.

### Detecting Hot-Reload Changes

The `Proxy` class provides `changed()` and `consume()` methods to detect when a resource has been hot-reloaded:

```cpp
resource::Proxy<render::Shader> shaderProxy;
// ... bind shader ...

// In update loop
if (shaderProxy.changed())
{
    // Resource was hot-reloaded, update dependent state
    rebuildMaterial();

    // Mark change as handled
    shaderProxy.consume();
}
```

### Resident vs Exclusive Resources

Resources can be either **resident** (cacheable, shared) or **exclusive** (non-cacheable, per-bind):

- **Resident resources** persist in the cache until explicitly unloaded or the manager is destroyed
- **Exclusive resources** can have multiple independent handles, each can be loaded/unloaded independently
- The resource factory determines whether a resource type is cacheable

## Automatic Lifetime Management

Resources use **reference counting** to manage their lifetime. When you copy a proxy, the reference count increases. When a proxy goes out of scope or is cleared, the reference count decreases. When the count hits zero, the resource can be unloaded:

```cpp
// Bind resource (refCount = 1)
resource::Id<render::Mesh> meshId(Guid(L"{12345678-1234-5678-1234-567812345678}"));
resource::Proxy<render::Mesh> mesh1;
resourceManager->bind(meshId, mesh1);

// Share resource (refCount = 2)
resource::Proxy<render::Mesh> mesh2 = mesh1;

// Release reference (refCount = 1)
mesh2.clear();  // or mesh2 = resource::Proxy<render::Mesh>();

// Release last reference (refCount = 0, resource may be unloaded)
mesh1.clear();
```

This is automatic and safe. You never need to manually unload resources. When they're no longer referenced, the system takes care of cleanup.

## Hot Reloading: Instant Iteration

One of the most developer-friendly features of Traktor is **hot reloading**. During development, when you modify an asset, the engine automatically reloads it without restarting the game:

Here's how it works:
1. You edit an asset in an external tool (e.g., a texture in Photoshop, a script in your code editor)
2. You save the file
3. The Traktor editor detects the change
4. The asset pipeline rebuilds the asset
5. The running game automatically reloads the resource

No restart, no manual reload. Just instant feedback. This dramatically speeds up iteration.

You can also manually trigger reloads:

```cpp
// Force reload specific resource
Guid resourceId = Guid(L"{12345678-1234-5678-1234-567812345678}");
resourceManager->reload(resourceId, false);  // false = reload all, not just flushed

// Reload all resources of a specific type
resourceManager->reload(type_of<render::Shader>(), false);
```

## Extending the Resource System

If you create custom resource types, you need to register a **resource factory**:

```cpp
class MyResourceFactory : public resource::IResourceFactory
{
    T_RTTI_CLASS;
public:
    virtual bool create(
        resource::IResourceManager* resourceManager,
        const db::Database* database,
        const db::Instance* instance,
        Ref<ISerializable>& outResource
    ) const override
    {
        // Load and create resource
        Ref<MyResource> resource = new MyResource();
        // ... initialize resource ...
        outResource = resource;
        return true;
    }
};
```

The factory tells the resource system how to create instances of your custom type from the database.

## Managing Memory

The resource system is smart about memory, but you can tune it for your specific needs.

### Resource Cache Management

The resource manager automatically caches loaded resources for reuse:

```cpp
// Unload all resources of a specific type
resourceManager->unload(type_of<render::Texture>());

// Unload externally unused, resident resources
// Call this to free memory from resources no longer referenced
resourceManager->unloadUnusedResident();

// Reload a specific resource (hot-reload)
Guid resourceId = Guid(L"{12345678-1234-5678-1234-567812345678}");
resourceManager->reload(resourceId, false);  // false = reload all, not just flushed

// Reload all resources of a type
resourceManager->reload(type_of<render::Shader>(), false);

// Get resource statistics
ResourceManagerStatistics stats;
resourceManager->getStatistics(stats);
log::info << "Resident resources: " << stats.residentCount << Endl;
log::info << "Exclusive resources: " << stats.exclusiveCount << Endl;
```

This is useful when transitioning between levels or when you need to free memory. The cache ensures that shared resources (used by multiple systems) are only loaded once.

## Best Practices

**Always use Proxies.** Never store raw pointers to resources. Proxies handle reference counting automatically and track hot-reload changes.

**Copy GUIDs from the editor.** Right-click assets in the Database view to copy their GUID for use in code.

**Use Stage transitions for loading.** Let the stage system handle loading entire levels rather than manually binding resources. Stages automatically load all the resources they need during stage transitions.

**Check bind() return value.** Always check if binding succeeded before using the resource:
```cpp
resource::Id<render::Mesh> meshId(...);
resource::Proxy<render::Mesh> meshProxy;
if (resourceManager->bind(meshId, meshProxy))
{
    // Binding successful, resource is loaded
    render::Mesh* mesh = meshProxy;
    // Use mesh safely
}
else
{
    // Binding failed - handle error
    log::error << "Failed to bind mesh resource" << Endl;
}
```

**Keep proxies alive.** Store proxies as member variables, not temporary variables. If the proxy is destroyed, the resource may be unloaded when the reference count reaches zero.

**Understand binding is synchronous.** `bind()` loads the resource immediately if not already cached. For large resources or level loading, use the stage system's async loading or load during a loading screen.

## Common Patterns

### Component Holding a Resource

Components often hold references to resources:

```cpp
class MeshComponent : public IEntityComponent
{
private:
    resource::Proxy<render::Mesh> m_mesh;

public:
    void setMesh(resource::Proxy<render::Mesh> mesh)
    {
        m_mesh = mesh;
    }

    virtual void update(const UpdateParams& update) override
    {
        if (m_mesh)  // Check if resource is loaded
        {
            // Use mesh
        }
    }
};
```

### Loading Between Stages

In most cases, you don't manually load resources. The Stage system handles resource loading for you:

```lua
-- From Lua script - transition to a new stage
-- The stage system automatically loads all resources needed by that stage
contextObject.stage:loadStage("MainMenu")
contextObject.stage:loadStage("Level1")
```

For more control, you can load stages asynchronously and transition when ready:

```lua
-- Load stage asynchronously in C++
Ref<StageLoader> loader = currentStage->loadStageAsync(L"Level1", params);

// Check if ready in update loop
if (loader->isReady())
{
    Ref<Stage> nextStage = loader->getStage();
    currentStage->gotoStage(nextStage);
}
```

The stage system handles all resource dependencies automatically - when you load a stage, all scenes, textures, meshes, and other resources referenced by that stage are loaded together.

## Debugging Resources

### Resource Tracking

During development, you can track resource loading through logging:

```cpp
// Log when resources are bound
resource::Id<render::Mesh> meshId(Guid(L"{12345678-1234-5678-1234-567812345678}"));
resource::Proxy<render::Mesh> meshProxy;
if (resourceManager->bind(meshId, meshProxy))
{
    log::info << "Successfully bound resource" << Endl;
}
else
{
    log::error << "Failed to bind resource" << Endl;
}

// Check if resource is loaded
if (meshProxy)
{
    log::info << "Resource is loaded and ready" << Endl;
}
```

### Hot-Reload Verification

To verify hot-reloading is working, you can manually trigger a reload and detect changes:

```cpp
// Force reload of a specific resource
Guid resourceId = Guid(L"{12345678-1234-5678-1234-567812345678}");
resourceManager->reload(resourceId, false);
log::info << "Triggered reload for resource" << Endl;

// Detect when hot-reload occurs
if (shaderProxy.changed())
{
    log::info << "Shader was hot-reloaded" << Endl;
    shaderProxy.consume();
}
```

The proxy will automatically pick up the new version of the resource.

### Resource Statistics

Monitor resource usage with statistics:

```cpp
ResourceManagerStatistics stats;
resourceManager->getStatistics(stats);

log::info << "Resident resources: " << stats.residentCount << Endl;
log::info << "Exclusive resources: " << stats.exclusiveCount << Endl;
```

## See Also

- [Architecture](architecture.md) - Resource system design
- [Editor](../editor.md) - Asset pipeline and building

## References

- Source: `code/Resource/`
