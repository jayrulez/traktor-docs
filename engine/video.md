---
layout: default
permalink: /engine/video/
title: Video
parent: Engine

nav_order: 11
---

# Video System - Full-Motion Video Playback

Imagine displaying a pre-rendered cinematic intro, showing live camera feeds in security monitors within your game world, or playing training videos on in-game computer screens. The Video system makes all of this possible by decoding video files and providing them as textures that can be displayed on any surface in your 3D world.

Traktor's Video module decodes video streams (currently supporting Ogg Theora format) and presents them as standard render textures. This means you can use video just like any other texture - apply it to materials, display it on 3D geometry, or render it to the screen. The system handles frame-by-frame decoding, timing synchronization, and texture updates automatically.

Think of video textures as animated textures - instead of loading a static image, you load a video file, and each frame the texture shows the current frame of the video. The decoder runs in a background thread, ensuring smooth playback without blocking your game.

## Use Cases

**In-game cinematics:** Display pre-rendered cutscenes or intro sequences

**Environmental storytelling:** TVs, computer screens, security monitors showing video content in the game world

**Tutorial videos:** In-game help systems with video instructions

**Background animations:** Animated billboards, advertisements, or decorative screens

**Camera feeds:** Simulated security camera footage or surveillance displays

Unlike scene animation (which animates entities in real-time), video playback uses pre-recorded video files. This is perfect when you need complex animation that would be too expensive to render in real-time, or when you want to include pre-rendered content.

## Architecture

The Video system has three main classes:

**VideoTexture** - The main class you interact with. It's an `ITexture` that decodes video and updates its contents each frame. You can use VideoTexture anywhere you'd use a regular texture - in materials, shaders, or UI rendering:

```cpp
class VideoTexture : public render::ITexture
{
public:
    bool create(render::IRenderSystem* renderSystem, IVideoDecoder* decoder);
    void destroy();

    // Inherited from ITexture
    Size getSize() const;
    int32_t getBindlessIndex() const;
    bool lock(int32_t side, int32_t level, Lock& lock);
    void unlock(int32_t side, int32_t level);
    ITexture* resolve();
};
```

**IVideoDecoder** - Interface for video format decoders. Each video format (Theora, etc.) implements this interface:

```cpp
struct VideoDecoderInfo
{
    uint32_t width;      // Video width in pixels
    uint32_t height;     // Video height in pixels
    float rate;          // Frame rate (frames per second)
};

class IVideoDecoder : public Object
{
    virtual bool create(IStream* stream) = 0;
    virtual void destroy() = 0;
    virtual bool getInformation(VideoDecoderInfo& outInfo) const = 0;
    virtual void rewind() = 0;
    virtual bool decode(uint32_t frame, void* bits, uint32_t pitch) = 0;
};
```

**VideoResource** - Serialized video data loaded from the database. The resource factory creates VideoTexture instances from VideoResource.

Located in:
- `code/Video/VideoTexture.h`
- `code/Video/IVideoDecoder.h`
- `code/Video/VideoResource.h`

## How Video Playback Works

The video system uses a threaded decoding architecture for smooth playback:

1. **VideoTexture creation:** You create a VideoTexture with a decoder (loaded from a video file)
2. **Background decoding:** A dedicated thread continuously decodes frames ahead of playback
3. **Frame buffering:** Decoded frames are stored in a buffer (4 frames deep)
4. **Texture updates:** Each frame, the next decoded frame is uploaded to GPU texture memory
5. **Rendering:** The texture is available for rendering like any other texture

This architecture ensures:
- **No frame drops:** Decoding happens ahead of time, so playback is smooth
- **No stuttering:** Main thread never blocks waiting for decoding
- **Minimal latency:** Only a few frames of buffering, so playback starts quickly

The VideoTexture automatically syncs with the video's frame rate, advancing to the next frame at the correct intervals.

## Supported Video Formats

Currently, Traktor supports:

**Ogg Theora** - Open-source video codec, good quality with reasonable file sizes
- Codec: Theora
- Container: Ogg (.ogv, .ogg)
- Free and patent-free
- Widely supported in open-source tools

**Decoder implementation:** `VideoDecoderTheora` (code/Video/Decoders/VideoDecoderTheora.h)

Future versions may add support for additional codecs like VP8/VP9, H.264, or others through additional IVideoDecoder implementations.

## Using Video in Your Game

### Loading Video Resources

Video resources are loaded through the standard resource system:

```cpp
// Load video resource from database
resource::Id<VideoResource> videoId(Guid(L"{12345678-1234-5678-1234-567812345678}"));
resource::Proxy<VideoTexture> videoTexture;

if (resourceManager->bind(videoId, videoTexture))
{
    // Video texture is loaded and ready
    render::ITexture* texture = videoTexture;

    // Use texture in material, shader, or rendering
}
```

**Note:** The GUID comes from your video asset in the Traktor editor. Right-click the video asset in the Database view and copy its GUID.

### Applying Video to Materials

Once you have a VideoTexture, use it like any other texture:

```cpp
// Create material with video texture
Ref<render::Shader> shader = ...;  // Your material shader
shader->setTextureParameter("DiffuseMap", videoTexture);

// Render geometry with the material
// The video will play on the surface
```

**Example: TV screen in game world:**
1. Create a 3D model of a TV with a screen surface
2. Apply a material to the screen surface that uses a video texture
3. The video plays on the TV screen in your 3D world

### Video in UI

You can also display video in 2D UI:

```cpp
// In UI rendering code
render::Context context;
context.setTexture(0, videoTexture);  // Bind video texture
context.drawQuad(rect);  // Draw full-screen or UI element

// Video plays in the UI
```

### Controlling Playback

VideoTexture automatically starts playing when created and loops when it reaches the end. For more control, you'd need to access the underlying decoder:

```cpp
// Rewind video to beginning
// (Note: This requires accessing internal decoder, not exposed in public API)
```

Currently, VideoTexture doesn't expose play/pause/stop controls through its public API - it plays automatically. For interactive control, you would need to implement a wrapper or extend the VideoTexture class.

## Creating Video Assets in the Editor

To add video to your game:

### 1. Import Video File

1. Open the Traktor editor
2. Right-click in the Database panel where you want to add the video
3. Select **Import** > **Video**
4. Browse to your .ogv (Ogg Theora) video file
5. Click **OK**

The editor imports the video and creates a VideoAsset.

### 2. Configure Video Asset

1. Double-click the VideoAsset to open it
2. Configure properties:
   - **Source File:** Path to the .ogv video file
   - **Compression:** Quality/size tradeoff (if supported)
   - **Loop:** Whether video should loop (default: true)

3. Save the asset

### 3. Build Pipeline

The pipeline processes the VideoAsset into a VideoResource:
1. Reads the source video file
2. Optionally re-encodes for target platform
3. Creates VideoResource with embedded or linked video data
4. Stores in the output database

### 4. Use in Scene

**Option A: Material with video texture**
1. Create a material in the Material Editor
2. Add a texture parameter (e.g., "DiffuseMap")
3. Assign your video asset to the texture parameter
4. Apply material to 3D geometry in your scene

**Option B: UI with video background**
1. In your UI shader/script, bind the video texture
2. Render it to screen or UI element
3. Video plays as background

## Video Performance

Video decoding is CPU-intensive but runs in a background thread:

**Decoding thread:** Continuously decodes frames ahead of playback
**Main thread:** Only uploads decoded frames to GPU (fast)
**GPU:** Renders the texture like any other texture (very fast)

**Performance considerations:**

**Resolution matters:** 1920x1080 video requires 4x more decoding than 960x540
**Frame rate matters:** 60fps video requires 2x more decoding than 30fps
**Multiple videos:** Each video has its own decoder thread - be mindful of CPU usage
**Texture memory:** Video textures consume GPU memory based on resolution

**Optimization tips:**

**Use appropriate resolution:** Don't use 1080p video for a small TV screen in your game
**Limit simultaneous videos:** 3-4 playing videos is usually fine, 20+ may impact performance
**Consider frame rate:** 24-30fps is often sufficient for in-game videos
**Compress appropriately:** Balance quality vs file size based on use case

## Technical Details

### Frame Buffer

VideoTexture maintains a 4-frame ring buffer:
- **Current frame:** Displayed on screen now
- **Next 3 frames:** Pre-decoded and ready

This buffering ensures smooth playback even if decoding occasionally takes longer than one frame.

### Thread Safety

The decoder thread and main thread coordinate through:
- **Frame indices:** Track which frame is decoded vs uploaded
- **Buffer management:** Ring buffer prevents overwrite of in-use frames
- **Synchronization:** Minimal locking ensures both threads run efficiently

### Texture Upload

Each frame, VideoTexture:
1. Checks if next frame is ready in the buffer
2. If ready, locks the GPU texture
3. Copies frame data to GPU memory
4. Unlocks the texture
5. Advances to next frame

This happens quickly (typically <1ms) since the frame is already decoded.

### Timing

VideoTexture uses `Timer` to track playback time and calculate which frame should be displayed based on the video's frame rate. It automatically handles frame skipping if the game is running slow (to keep audio/video in sync if audio is added later).

## Extending the Video System

### Adding New Codecs

To support additional video formats, implement `IVideoDecoder`:

```cpp
class MyVideoDecoder : public IVideoDecoder
{
public:
    virtual bool create(IStream* stream) override
    {
        // Open video file stream
        // Initialize codec
        return true;
    }

    virtual void destroy() override
    {
        // Clean up codec resources
    }

    virtual bool getInformation(VideoDecoderInfo& outInfo) const override
    {
        // Fill in video width, height, frame rate
        outInfo.width = 1920;
        outInfo.height = 1080;
        outInfo.rate = 30.0f;
        return true;
    }

    virtual void rewind() override
    {
        // Seek to beginning of video
    }

    virtual bool decode(uint32_t frame, void* bits, uint32_t pitch) override
    {
        // Decode specific frame into bits buffer
        // pitch is row stride in bytes
        // Return true if successful
        return true;
    }
};
```

Register your decoder in the VideoFactory to make it available for video resources.

### Custom Playback Control

For interactive video (play/pause/seek), you can wrap VideoTexture:

```cpp
class InteractiveVideo : public Object
{
public:
    InteractiveVideo(VideoTexture* texture)
    :   m_texture(texture)
    ,   m_playing(true)
    {}

    void play() { m_playing = true; }
    void pause() { m_playing = false; }
    void seek(float time) { /* implement seeking */ }

    void update(float deltaTime)
    {
        if (m_playing)
        {
            // Update video texture
            // (Note: Current VideoTexture API doesn't support this directly)
        }
    }

private:
    Ref<VideoTexture> m_texture;
    bool m_playing;
};
```

This requires extending VideoTexture or accessing its internal decoder.

## Limitations

**No audio:** Currently, the Video module only decodes video frames, not audio. For videos with sound, decode audio separately using the Sound module and synchronize manually.

**No playback control:** VideoTexture plays automatically without exposed play/pause/seek controls. You'd need to extend the class or implement wrapper logic.

**Single loop mode:** Videos loop by default. No built-in support for play-once-and-stop (though you could detect loop and stop rendering).

**Limited codec support:** Only Ogg Theora currently supported. Additional codecs require implementing new IVideoDecoder subclasses.

These limitations reflect the current simple design focused on background/ambient video playback rather than interactive video players.

## Best Practices

**Choose appropriate resolution:** Match video resolution to display size. A 256x256 TV screen doesn't need 1080p video.

**Optimize frame rate:** 24-30fps is usually sufficient for in-game videos. Don't use 60fps unless necessary.

**Pre-compress videos:** Compress videos before importing to balance quality vs file size.

**Limit simultaneous playback:** Test performance with your target number of concurrent videos.

**Use for non-interactive content:** The current system works best for background/ambient videos, not interactive playback.

**Consider streaming:** For long videos, ensure the system can stream from disk rather than loading entirely into memory (check VideoDecoderTheora implementation details).

**Test on target hardware:** Video decoding performance varies widely across platforms. Test on lowest-spec target device.

## Troubleshooting

**Video doesn't play:**
- Verify video format is Ogg Theora (.ogv)
- Check video asset imported correctly in editor
- Ensure VideoResource bound successfully
- Check console for decoder errors

**Video stutters or drops frames:**
- Video resolution too high for target hardware
- Too many simultaneous videos
- CPU overloaded with other tasks
- Consider reducing resolution or frame rate

**Video quality poor:**
- Source video compression too aggressive
- Re-encode source with higher quality settings
- Check that pipeline isn't re-compressing unnecessarily

**Can't control playback:**
- Current API doesn't expose play/pause controls
- Consider extending VideoTexture for custom control
- Or use video only for passive background content

## Lua API

VideoTexture is registered in Lua but currently has no exposed methods or properties:

```lua
-- VideoTexture exists as a type but has no Lua-accessible interface
-- Use it through materials or shaders that accept textures
```

For Lua scripts, video textures are typically:
1. Assigned to materials in the editor
2. Materials applied to entities
3. Scripts don't directly control video playback

## See Also

- [Render System](render.md) - Textures and materials
- [Resource System](resources.md) - Loading video resources
- [Sound System](sound.md) - For audio playback (separate from video)

## References

- Source: `code/Video/VideoTexture.h`
- Source: `code/Video/IVideoDecoder.h`
- Source: `code/Video/VideoResource.h`
- Decoder: `code/Video/Decoders/VideoDecoderTheora.h`
- Factory: `code/Video/VideoFactory.h`
