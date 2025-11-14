---
layout: default
permalink: /engine/weather/
title: Weather
parent: Engine

nav_order: 16
---

# Weather System - Atmosphere and Immersion

Weather transforms a game world from a static backdrop into a living, breathing environment. A beautiful sky gradient at sunset, falling rain or snow, dramatic storm clouds. These atmospheric touches set mood and make your world feel alive.

Traktor's weather system provides two editor-configured components:

**SkyComponent** renders the sky with configurable colors, intensity, saturation, and optional cloud rendering. You set sky colors (above and below the horizon), adjust intensity and saturation, and enable or disable clouds. The sky shader handles atmospheric scattering to create realistic sky gradients based on your color settings.

**PrecipitationComponent** renders rain or snow effects using a particle mesh. You configure a mesh (rain streaks or snow flakes), tilt rate (how much wind affects the angle), parallax distance (creates depth), and opacity. The precipitation follows the camera, creating the illusion of weather filling the entire level.

**Important:** These components are configured in the editor only. They are **not exposed to the Lua scripting API**. Sky colors, cloud settings, and precipitation parameters are static, per-level configuration set during level design.

![TODO: Screenshot showing weather effects with sky and precipitation components in the editor]

## SkyComponent: Rendering the Sky

Traditional skyboxes are static images wrapped around your scene. Traktor's **SkyComponent** is more flexible. It uses a shader-based approach with configurable colors and atmospheric effects.

The SkyComponent provides these parameters (configured in the editor):

**Sky colors** define the appearance of the sky:
- `skyOverHorizon` - Color above the horizon (typically blue during day)
- `skyUnderHorizon` - Color below the horizon (typically lighter near horizon)

**Intensity** (float) - Overall brightness of the sky. Higher values make the sky brighter.

**Saturation** (float) - Color saturation of the sky. Lower values create a washed-out look, higher values create vibrant colors.

**Clouds** (boolean) - Enable or disable cloud rendering. When enabled, clouds are rendered using the assigned cloud shader and texture.

**Cloud colors** (when clouds are enabled):
- `cloudAmbientTop` - Ambient color for the top of clouds
- `cloudAmbientBottom` - Ambient color for the bottom of clouds

**Shader and texture** - References to the sky shader and optional texture resources used for rendering.

The sky shader handles atmospheric scattering calculations, creating realistic gradients between your configured horizon colors. The result is a procedural sky that looks natural without requiring a pre-rendered skybox texture.

```cpp
// C++ - Create sky component
Ref<SkyComponent> sky = new SkyComponent(skyData, irradianceGrid, shader, texture);
entity->setComponent(sky);
```

**Note:** SkyComponent is not exposed to the Lua API. Sky configuration is done entirely in the editor.

## PrecipitationComponent: Rain and Snow Effects

The PrecipitationComponent renders weather effects like rain or snow using a particle mesh that follows the camera, creating the illusion of weather filling your entire level.

The PrecipitationComponent provides these parameters (configured in the editor):

**Mesh** - A reference to a StaticMesh resource that defines the appearance of precipitation particles. For rain, this is typically vertical streaks. For snow, this is usually snowflake sprites. The mesh is instanced many times to fill the view.

**Tilt rate** (float) - How much the precipitation tilts with camera movement, simulating wind. Higher values make rain or snow appear to be blown at an angle. Default is 6.0.

**Parallax distance** (float) - Controls depth perception by making closer particles move faster than distant ones. Creates a sense of 3D depth in the precipitation. Default is 1.0.

**Depth distance** (float) - Distance at which particles fade out. Controls how far you can see precipitation. Default is 1.0.

**Opacity** (float) - Overall transparency of precipitation particles. Lower values make rain or snow more subtle and transparent. Default is 0.1.

The precipitation system is designed to be lightweight and efficient. The mesh follows the camera so particles are always visible, and the parallax system creates the illusion of depth without actually simulating thousands of individual raindrops or snowflakes.

```cpp
// C++ - Create precipitation component
Ref<PrecipitationComponent> precipitation = new PrecipitationComponent(precipitationData);
entity->setComponent(precipitation);
```

**Note:** PrecipitationComponent is not exposed to the Lua API. Precipitation configuration is done entirely in the editor.

To create rain, assign a mesh with vertical rain streak textures. To create snow, assign a mesh with snowflake sprites and adjust the tilt rate for gentle drifting.

## Using Weather Components Together

The SkyComponent and PrecipitationComponent work together to create atmospheric environments:

**Clear sunny day:**
- SkyComponent with bright blue `skyOverHorizon`, lighter blue `skyUnderHorizon`
- High intensity (1.0+), moderate saturation
- Clouds disabled
- No precipitation

**Overcast rainy day:**
- SkyComponent with gray `skyOverHorizon` and `skyUnderHorizon`
- Lower intensity (0.5-0.7), lower saturation
- Clouds enabled with gray cloud colors
- PrecipitationComponent with rain streak mesh, moderate opacity

**Snowy winter:**
- SkyComponent with pale, cold colors
- Clouds enabled
- PrecipitationComponent with snowflake mesh, low tilt rate for gentle drift

**Dramatic sunset:**
- SkyComponent with orange/red `skyOverHorizon`, purple `skyUnderHorizon`
- High intensity and saturation for vibrant colors
- Clouds enabled (optional) with warm cloud colors

These combinations are set up in the editor for each level. The weather system provides the visual atmosphere, while other systems (lighting, audio, particles) contribute additional effects like thunder sounds or wind audio.

## Performance Considerations

The Weather module's components are designed to be efficient, but there are still considerations:

**Sky shader complexity** - The sky shader performs atmospheric scattering calculations. This is typically inexpensive since it only renders the skybox, but complex custom sky shaders can impact performance.

**Precipitation mesh density** - The precipitation mesh is instanced across the view. Keep the mesh simple (low vertex count) since it's rendered many times. Hundreds of triangles per particle is too much; aim for simple quads or low-poly sprites.

**Precipitation opacity** - Lower opacity values mean more blending, which can cause overdraw (redrawing the same pixel many times). If precipitation is causing performance issues, check if reducing opacity or particle density helps.

**Cloud rendering** - When clouds are enabled, cloud rendering adds cost. The performance impact depends on your cloud shader and texture complexity.

**Quality settings** - Provide options to disable clouds or reduce precipitation opacity for lower-end hardware.

## Best Practices

**Match sky colors to your lighting** - The sky color should complement your directional light color and intensity. A bright blue sky with dim orange lighting looks inconsistent.

**Subtle precipitation looks more realistic** - Lower opacity with appropriate mesh density creates convincing rain or snow. Heavy, opaque precipitation can look artificial.

**Use audio to sell weather** - The Weather module only provides visuals. Combine it with audio (rain sounds, wind ambience, thunder) to create convincing atmospheric effects.

**Test precipitation at different camera angles** - Make sure the parallax and tilt settings look good whether the player is looking up, down, or horizontally.

**Consider performance on target platform** - Test weather effects on your target hardware early. Precipitation can be expensive on low-end devices due to overdraw and instancing overhead.

## See Also

- [World System](world.md) - Weather components
- [Render System](render.md) - Atmospheric rendering and sky
- [Sound System](sound.md) - Weather sound effects
- [Effects](effects.md) - Rain and snow particle effects

## References

- Source: `code/Weather/`
