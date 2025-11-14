---
layout: default
permalink: /engine/sound/
title: Sound
parent: Engine

nav_order: 5
---

# Sound System - Low-Level Audio Architecture

Imagine the player firing a gun. The crack of the shot needs to come from exactly where the gun is in 3D space, with the right volume based on distance, filtered if there's a wall between the gun and the listener, and mixed with dozens of other sounds all playing at once. All of this needs to happen in real-time with zero noticeable latency. That's what the Sound system does.

Traktor's Sound system is the audio engine that handles everything from mixing virtual channels to driving platform-specific audio hardware. It provides both low-level audio architecture (AudioSystem, AudioChannel, drivers) and high-level game audio APIs (SoundPlayer, SoundHandle). Think of it as the complete audio solution - from the mixer thread that processes samples to the 3D positioning that makes sounds feel alive in your world.

This documentation covers the Sound module (`code/Sound/`), which provides the complete audio pipeline from low-level mixing to high-level game sound playback.

## Architecture Overview

The Sound system has three main layers:

**AudioSystem** - The central audio manager that creates and manages virtual audio channels. It runs a dedicated mixer thread that continuously mixes audio from all playing sounds and submits blocks of samples to the audio driver for playback. The AudioSystem handles global volume control, category volumes, and channel management.

**AudioChannel** - Individual virtual audio channels that play sounds. Each channel can play one sound at a time, with control over volume, pitch, and audio filters. The system manages a pool of channels (typically 32-64), allocating them to sounds as needed. When all channels are in use, lower-priority sounds get bumped.

**IAudioDriver** - Platform-specific audio output. Different drivers handle different platforms: XAudio2 (modern Windows), DirectSound (legacy Windows), OpenAL (cross-platform), OpenSL (Android), ALSA/PulseAudio (Linux), etc. The driver receives mixed audio blocks and pushes them to the hardware for playback.

This architecture separates platform-specific code (drivers) from mixing logic (AudioSystem/channels), making it easy to add new platform support or optimize mixing without touching driver code.

## Creating the Audio System

The AudioSystem is typically created by the runtime environment, but if you're building a custom application, here's how you create it:

```cpp
// Create audio driver for your platform
Ref<IAudioDriver> driver = new AudioDriverXAudio2();  // Windows
// Or: new AudioDriverOpenAL();  // Cross-platform
// Or: new AudioDriverOpenSL();  // Android

// Create audio system
Ref<AudioSystem> audioSystem = new AudioSystem(driver);

// Configure audio system
AudioSystemCreateDesc desc;
desc.channels = 32;  // Number of virtual channels
desc.driverDesc.sampleRate = 44100;  // Hz
desc.driverDesc.bitsPerSample = 16;  // 16-bit audio
desc.driverDesc.hwChannels = SbcMaxChannelCount;  // Speaker configuration
desc.driverDesc.frameSamples = 1024;  // Samples per block

// Create audio system
if (audioSystem->create(desc))
{
    // Audio system ready
}
```

**Channel count** determines how many sounds can play simultaneously. Set this based on your game's needs - 32 channels works for most games, 64 for more complex audio scenarios.

**Sample rate** is typically 44100 Hz (CD quality) or 48000 Hz (DVD quality). Higher sample rates produce better quality but consume more CPU and memory.

**Frame samples** is the size of each audio block submitted to the driver. Smaller blocks = lower latency but more CPU overhead. Larger blocks = higher latency but more efficient. 1024 samples at 44100 Hz = ~23ms latency, which is imperceptible.

**Speaker configuration** is defined by `SbcMaxChannelCount` from Types.h:
- 2.0 stereo: 2 channels (Left, Right)
- 5.1 surround: 6 channels (Left, Right, Center, LFE, RearLeft, RearRight)
- 7.1 surround: 8 channels (Left, Right, Center, LFE, RearLeft, RearRight, SideLeft, SideRight)

The system automatically adapts to your platform's default configuration.

## AudioSystem API

The AudioSystem class manages the entire audio pipeline:

```cpp
class AudioSystem : public Object
{
public:
    // Create audio system with configuration
    bool create(const AudioSystemCreateDesc& desc);

    // Destroy audio system (stops all playback)
    void destroy();

    // Reset with a new audio driver
    bool reset(IAudioDriver* driver);

    // Suspend/resume playback (for app backgrounding)
    void suspend();
    void resume();

    // Global volume control (0-1)
    void setVolume(float volume);
    float getVolume() const;

    // Category volume control (0-1)
    void setVolume(handle_t category, float volume);
    float getVolume(handle_t category) const;

    // Combination matrix (advanced channel routing)
    void setCombineMatrix(float cm[SbcMaxChannelCount][SbcMaxChannelCount]);

    // Get virtual audio channel
    AudioChannel* getChannel(uint32_t channelId);

    // Get current mixer time (in seconds)
    double getTime() const;

    // Performance monitoring
    void getThreadPerformances(double& outMixerTime) const;
};
```

**Suspend/resume** is crucial for mobile platforms. When your app goes to the background, call `suspend()` to stop audio playback and free hardware resources. When returning to foreground, call `resume()` to restart playback.

**Category volumes** allow you to adjust volume for different types of sounds independently. For example, you might have categories for music, SFX, dialogue, and UI sounds. Players can adjust each category's volume in settings:

```cpp
// Define category handles
handle_t categoryMusic = getParameterHandle(L"Music");
handle_t categorySFX = getParameterHandle(L"SFX");
handle_t categoryDialogue = getParameterHandle(L"Dialogue");

// Set category volumes based on player settings
audioSystem->setVolume(categoryMusic, 0.7f);     // Music at 70%
audioSystem->setVolume(categorySFX, 1.0f);       // SFX at 100%
audioSystem->setVolume(categoryDialogue, 0.9f);  // Dialogue at 90%
```

**Combination matrix** is an advanced feature for custom channel routing. It maps virtual channels to hardware channels. The default is identity (each virtual channel maps to its corresponding hardware channel), but you can create custom mappings for special effects or unusual speaker configurations.

## Playing Sounds: AudioChannel

Each `AudioChannel` represents one playing sound. Channels are allocated from the AudioSystem's pool and reused:

```cpp
class AudioChannel : public Object
{
public:
    // Volume control (0-1)
    void setVolume(float volume);
    float getVolume() const;

    // Pitch control (1.0 = normal, 0.5 = half speed, 2.0 = double speed)
    void setPitch(float pitch);
    float getPitch() const;

    // Attach audio filter (for processing)
    void setFilter(const IAudioFilter* filter);

    // Set cursor parameter (for procedural audio)
    void setParameter(handle_t id, float parameter);

    // Disable repeat on current sound
    void disableRepeat();

    // Play sound buffer
    bool play(
        const IAudioBuffer* buffer,
        handle_t category,
        float gain,        // Gain in decibels
        bool repeat,
        uint32_t repeatFrom  // Sample offset for repeat
    );

    // Check if sound is playing
    bool isPlaying() const;

    // Stop playback
    void stop();

    // Get buffer cursor (for seeking, etc.)
    IAudioBufferCursor* getCursor();
};
```

**Pitch control** changes playback speed. Pitch 2.0 plays at double speed (one octave up), 0.5 plays at half speed (one octave down). This is useful for sound variation (randomize pitch slightly for each footstep), slow-motion effects, or procedural voice effects.

**Gain in decibels:** The `play()` method takes gain in decibels, not linear volume. Use the conversion functions:

```cpp
// Convert between linear and decibel
float db = linearToDecibel(0.5f);  // 0.5 linear = -6.02 dB
float lin = decibelToLinear(-6.0f); // -6 dB = 0.501 linear
```

Decibels are logarithmic, matching how humans perceive loudness. -6 dB is half volume, -12 dB is quarter volume, etc.

**Repeat and repeatFrom:** When repeat is true, the sound loops. The `repeatFrom` parameter specifies which sample to loop back to. For music with an intro, you might set repeatFrom to the sample where the main loop starts, so the intro plays once and then the loop section repeats.

## High-Level Sound Player

For most game code, you don't use AudioChannel directly. Instead, use `SoundPlayer`, which provides a high-level API with 3D positioning, priority management, and automatic channel allocation:

```cpp
class SoundPlayer : public Object
{
public:
    // Create sound player
    bool create(AudioSystem* audioSystem, SurroundEnvironment* surroundEnvironment);

    // Destroy sound player
    void destroy();

    // Play 2D sound (no spatial positioning)
    Ref<SoundHandle> play(const Sound* sound, uint32_t priority);

    // Play 3D sound (positioned in space)
    Ref<SoundHandle> play(
        const Sound* sound,
        const Vector4& position,
        uint32_t priority,
        bool autoStopFar  // Automatically stop when too far from listener
    );

    // Add/remove listeners (for 3D audio)
    void addListener(const SoundListener* listener);
    void removeListener(const SoundListener* listener);

    // Update (call every frame)
    void update(float dT);
};
```

**Priority** determines which sounds get channels when all channels are in use. Higher priority sounds bump lower priority sounds. Use priorities like:
- 255: Critical UI sounds, always play
- 200: Player gunshots, footsteps
- 150: Enemy sounds nearby
- 100: Environmental sounds
- 50: Distant ambient sounds

**SoundHandle** is returned from `play()` and lets you control the playing sound:

```cpp
class SoundHandle : public Object
{
public:
    // Stop sound immediately
    void stop();

    // Fade off (smooth volume fade to zero, then stop)
    void fadeOff();

    // Check if sound is still playing
    bool isPlaying();

    // Adjust volume while playing
    void setVolume(float volume);

    // Adjust pitch while playing
    void setPitch(float pitch);

    // Update position for 3D sounds
    void setPosition(const Vector4& position);

    // Set procedural audio parameter
    void setParameter(int32_t id, float parameter);

    // Get audio buffer cursor (advanced)
    IAudioBufferCursor* getCursor();
};
```

**Example: Playing a 3D gunshot sound:**

```cpp
// Play gunshot at gun's position
Ref<SoundHandle> gunshotSound = soundPlayer->play(
    gunshotResource,         // Sound resource
    gunPosition,             // 3D position in world
    200,                     // High priority
    true                     // Auto-stop when far away
);

// Randomize pitch slightly for variation
gunshotSound->setPitch(1.0f + (rand() / (float)RAND_MAX * 0.2f - 0.1f));  // 0.9 to 1.1
```

## Sound Resources

The `Sound` class wraps an audio buffer with metadata:

```cpp
class Sound : public Object
{
public:
    Sound(
        IAudioBuffer* buffer,  // Audio data
        handle_t category,     // Category (Music, SFX, etc.)
        float gain,            // Gain in dB
        float range            // 3D audio range
    );

    IAudioBuffer* getBuffer() const;
    uint32_t getCategory() const;
    float getGain() const;
    float getRange() const;
};
```

**IAudioBuffer** is the actual audio data. It can be a simple PCM buffer, a streamed audio file (MP3, OGG, FLAC), or a procedural audio generator (Resound bank):

```cpp
class IAudioBuffer : public Object
{
public:
    // Create cursor for playback
    virtual Ref<IAudioBufferCursor> createCursor() const = 0;

    // Get audio block from cursor
    virtual bool getBlock(
        IAudioBufferCursor* cursor,
        const IAudioMixer* mixer,
        AudioBlock& outBlock
    ) const = 0;
};
```

The buffer/cursor pattern allows multiple sounds to play from the same buffer simultaneously (each gets its own cursor tracking playback position).

## 3D Audio Listener

For 3D audio to work, the system needs to know where the listener is (typically the player's camera or character):

```cpp
class SoundListener : public Object
{
public:
    // Set listener transform (position and orientation)
    void setTransform(const Transform& transform);

    const Transform& getTransform() const;
};
```

**Usage:**

```cpp
// Create listener
Ref<SoundListener> listener = new SoundListener();

// Update listener each frame to match camera
listener->setTransform(camera->getTransform());
soundPlayer->addListener(listener);
```

The listener's position and orientation determine how 3D sounds are spatialized. Sounds in front use the front speakers, sounds behind use the rear speakers, sounds to the left are louder in the left ear, etc.

## Audio Filters

Filters process audio in real-time. They're attached to channels to apply effects like surround sound spatialization, low-pass filtering, reverb, and more.

### Surround Filter

The `SurroundFilter` positions 3D sounds in the surround field based on the listener's position:

```cpp
class SurroundFilter : public IAudioFilter
{
public:
    SurroundFilter(
        SurroundEnvironment* environment,
        const Vector4& position,    // Sound source position
        float maxDistance           // Distance where sound is inaudible
    );

    // Update speaker position (for moving sounds)
    void setSpeakerPosition(const Vector4& position);

    // Update max distance
    void setMaxDistance(float maxDistance);
};
```

**SurroundEnvironment** defines global 3D audio settings:

```cpp
class SurroundEnvironment : public Object
{
public:
    // Distance where attenuation starts
    void setMaxDistance(float maxDistance);
    float getMaxDistance() const;

    // Inner radius (full volume sphere)
    void setInnerRadius(float innerRadius);
    float getInnerRadius() const;

    // Enable full surround (vs stereo-only)
    void setFullSurround(bool fullSurround);
    bool getFullSurround() const;
};
```

**How 3D audio attenuation works:**
- Within `innerRadius`: Sound plays at full volume, no attenuation
- Between `innerRadius` and `maxDistance`: Volume smoothly fades based on distance
- Beyond `maxDistance`: Sound is silent

### Low-Pass Filter

The `LowPassFilter` attenuates high frequencies, creating a muffled effect:

```cpp
class LowPassFilter : public IAudioFilter
{
public:
    LowPassFilter();
    explicit LowPassFilter(float cutOff);  // Cutoff frequency in Hz

    // Adjust cutoff frequency
    void setCutOff(float cutOff);
    float getCutOff() const;
};
```

**Use cases:**
- Underwater sounds: Low cutoff (~300-500 Hz) makes everything sound muffled
- Sounds through walls: Medium cutoff (~2000-4000 Hz) reduces clarity
- Distance: Far sounds naturally lose high frequencies

**Example: Muffled underwater sound:**

```cpp
// Create low-pass filter for underwater effect
Ref<LowPassFilter> underwaterFilter = new LowPassFilter(400.0f);  // 400 Hz cutoff

// Attach to channel
channel->setFilter(underwaterFilter);
```

### Other Filters

Traktor provides several other audio filters:

**CombFilter** - Creates comb filtering effects (metallic, robotic sounds)

**DitherFilter** - Adds dithering to reduce quantization noise

**EqualizerFilter** - Frequency equalization for tone shaping

**FFTFilter** - Frequency-domain processing base class

**GroupFilter** - Chains multiple filters together

**NormalizationFilter** - Automatic gain control/normalization

**RingModulationFilter** - Ring modulation effects (tremolo, vibrato, weird tones)

All filters inherit from `IAudioFilter`:

```cpp
class IAudioFilter : public Object
{
    // Filter implementations create instances and process audio
};
```

Filters are attached to channels with `channel->setFilter()`. Only one filter per channel, but use `GroupFilter` to chain multiple filters.

## Resound: Procedural Audio Banks

Traktor includes a unique system called **Resound** for procedural audio. Instead of pre-recorded audio files, Resound uses "grains" (building blocks) to generate audio procedurally. This allows variations, randomization, blending, and compact representation.

### Bank Buffer

A `BankBuffer` is an `IAudioBuffer` built from grains:

```cpp
class BankBuffer : public IAudioBuffer
{
public:
    BankBuffer(const RefArray<IGrain>& grains);
};
```

### Grain Types

Grains are small audio operations that can be combined:

**PlayGrain** - Plays a simple audio clip

**RandomGrain** - Randomly selects one of several child grains

**SequenceGrain** - Plays child grains in sequence

**SimultaneousGrain** - Plays multiple child grains simultaneously

**BlendGrain** - Blends between child grains based on a parameter

**TriggerGrain** - Waits for a trigger before playing

**RepeatGrain** - Repeats a child grain N times

**MuteGrain** - Silence for a duration

**EnvelopeGrain** - Applies volume envelope (attack/decay/sustain/release)

**InLoopOutGrain** - Plays intro, loops middle section, then plays outro

**Example use case:** Footsteps with variation:
```
RandomGrain (selects one):
  - PlayGrain(footstep_grass_01.wav)
  - PlayGrain(footstep_grass_02.wav)
  - PlayGrain(footstep_grass_03.wav)
```

Every time you play this bank, it randomly selects one of the three footstep sounds, giving natural variation without pre-creating dozens of variations.

**Example use case:** Looping music with intro:
```
SequenceGrain:
  - PlayGrain(music_intro.wav)
  - InLoopOutGrain:
      - In: silence
      - Loop: PlayGrain(music_loop.wav) [repeats]
      - Out: PlayGrain(music_outro.wav) [when stopped]
```

This plays the intro once, loops the main section, and plays the outro when you stop the music.

**All grain types have matching data classes** (`PlayGrainData`, `RandomGrainData`, etc.) that define the grain's configuration in the editor. At runtime, the data is compiled into grain instances that generate audio.

## Audio Blocks: The Pipeline Data

The entire audio system works with `AudioBlock` structures - chunks of multi-channel audio samples:

```cpp
struct AudioBlock
{
    float* samples[SbcMaxChannelCount];  // Pointers to each channel's samples
    uint32_t samplesCount;               // Number of samples per channel
    uint32_t sampleRate;                 // Sample rate (e.g., 44100 Hz)
    uint32_t maxChannel;                 // Highest channel with audio data
    handle_t category;                   // Sound category
};
```

**How the pipeline works:**
1. AudioChannel requests audio block from IAudioBuffer via cursor
2. IAudioBuffer fills the block with samples (PCM data, decoded stream, or procedurally generated)
3. If filter is attached, filter processes the block
4. AudioSystem mixes all channel blocks into final multi-channel output
5. Final mixed block is submitted to IAudioDriver
6. Driver pushes block to hardware playback

This happens continuously in the mixer thread, producing blocks at exactly the rate needed to prevent audio glitches.

## Platform-Specific Drivers

Each platform has its own audio driver implementation:

**AudioDriverXAudio2** - Modern Windows (Windows 7+), low latency, high quality

**AudioDriverDs8** - DirectSound 8 (legacy Windows), broader compatibility

**AudioDriverWinMM** - WinMM (oldest Windows API), fallback for ancient systems

**AudioDriverOpenAL** - Cross-platform (Windows, Linux, macOS), good compatibility

**AudioDriverOpenSL** - Android and embedded platforms

**AudioDriverAlsa** - Linux ALSA (low-level Linux audio)

**AudioDriverPulseAudio** - Linux PulseAudio (high-level Linux audio)

**AudioDriverNull** - Silent driver (for testing or headless servers)

All drivers implement the same `IAudioDriver` interface:

```cpp
class IAudioDriver : public Object
{
public:
    // Create driver with description and optional custom mixer
    virtual bool create(
        const SystemApplication& sysapp,
        const AudioDriverCreateDesc& desc,
        Ref<IAudioMixer>& outMixer
    ) = 0;

    // Destroy driver (stop playback, release hardware)
    virtual void destroy() = 0;

    // Wait until driver is ready for next block
    virtual void wait() = 0;

    // Submit mixed audio block for playback
    virtual void submit(const AudioBlock& block) = 0;
};
```

The AudioSystem calls `submit()` with each mixed block, then calls `wait()` to synchronize with playback. The driver must ensure smooth, glitch-free playback by managing internal buffers appropriately.

## Mixer Thread

The AudioSystem runs a dedicated **mixer thread** that continuously:
1. Mixes all active channels into output blocks
2. Waits for driver to be ready (`driver->wait()`)
3. Submits mixed block to driver (`driver->submit(block)`)
4. Repeats

This thread runs at high priority to ensure audio never glitches, even if the game thread stutters. The mixer thread is completely separate from the game thread, so game logic slowdowns don't affect audio.

**Performance monitoring:**

```cpp
double mixerTime;
audioSystem->getThreadPerformances(mixerTime);

if (mixerTime > 0.010)  // 10ms
{
    log::warning << "Mixer thread taking too long: " << mixerTime * 1000.0 << " ms" << Endl;
}
```

If mixer thread time is consistently high, you're either mixing too many channels, using expensive filters, or the CPU is overloaded.

## Best Practices

**Use the high-level SoundPlayer when possible.** It handles channel allocation, priorities, 3D positioning, and other details automatically. Only drop to AudioChannel level when you need fine control.

**Limit active channels.** More channels = more mixing work. 32-64 channels is plenty for most games. Use priorities to ensure important sounds always play.

**Cache sound resources.** Load Sound resources once and reuse them. Don't load the same wav file multiple times.

**Stream large audio files.** Music and long dialogue should stream from disk, not load entirely into memory. Small sound effects should be fully loaded for instant playback.

**Use categories for volume control.** Let players adjust music, SFX, and dialogue volumes independently.

**Monitor mixer thread performance.** If mixer time exceeds frame budget, reduce channel count or simplify filters.

**Test on target hardware.** Audio performance varies wildly across platforms. Test on low-end devices to ensure smooth playback.

**Handle suspend/resume on mobile.** Always suspend when backgrounded, resume when foregrounded.

## Lua API

From Lua scripts, you have access to some Sound classes via the AudioClassFactory registration:

```lua
-- Access audio system (typically provided by context)
local audioSystem = context.audioSystem

-- Get channel by ID
local channel = audioSystem:getChannel(0)

-- Set channel volume and pitch
channel.volume = 0.8
channel.pitch = 1.2

-- Check if playing
if channel.playing then
    channel:stop()
end

-- Play sound through channel
channel:play(soundBuffer, category, gain, repeat, repeatFrom)

-- Audio filters from Lua
local lowPass = LowPassFilter(800.0)  -- 800 Hz cutoff
channel:setFilter(lowPass)

-- Sound handle (from SoundPlayer)
local handle = soundPlayer:play(sound, priority)
handle:setVolume(0.5)
handle:setPitch(1.1)
handle:setPosition(Vector4(10, 0, 0))
handle:stop()

-- Surround environment
local surroundEnv = SurroundEnvironment()
surroundEnv.maxDistance = 50.0
surroundEnv.innerRadius = 5.0
surroundEnv.fullSurround = true
```

Most game code uses the higher-level audio system rather than directly manipulating channels, but the Lua API is there when you need it.

## See Also

- [Scripting](scripting.md) - Using audio from Lua scripts
- [Resource System](resources.md) - Loading sound resources
- [Effects](effects.md) - Sound effects integrated with visual effects

## References

- Source: `code/Sound/AudioSystem.h`
- Source: `code/Sound/AudioChannel.h`
- Source: `code/Sound/Player/SoundPlayer.h`
- Source: `code/Sound/Player/SoundHandle.h`
- Source: `code/Sound/Player/SoundListener.h`
- Source: `code/Sound/Sound.h`
- Source: `code/Sound/IAudioBuffer.h`
- Source: `code/Sound/IAudioDriver.h`
- Source: `code/Sound/IAudioFilter.h`
- Source: `code/Sound/Filters/`
- Source: `code/Sound/Resound/`
