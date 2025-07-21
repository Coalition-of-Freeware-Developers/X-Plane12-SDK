# XPLMSound API Documentation

## Overview

The XPLMSound API provides integration with X-Plane's FMOD-based audio system, enabling plugins to play custom sounds with full 3D spatial positioning, volume control, and audio bus routing. This API offers both simple PCM audio playback and advanced FMOD system access for complex audio applications.

## Key Features

- **3D Spatial Audio**: Position sounds in 3D space with distance attenuation
- **Audio Bus Routing**: Route sounds to appropriate audio channels (cockpit, exterior, radio, etc.)
- **FMOD Integration**: Full access to X-Plane's FMOD Studio system
- **Simple PCM Playback**: Easy audio playback without FMOD knowledge
- **Advanced Control**: Volume, pitch, fade distance, and directional cone control
- **Multiple Audio Devices**: Support for separate headset outputs for radio communications

## Architecture

### Audio Bus System

X-Plane routes audio through specialized buses that correspond to different parts of the simulation environment:

- **Radio Communications**: COM1, COM2, Pilot, Copilot channels
- **Aircraft Audio**: Interior and exterior aircraft sounds
- **Environment Audio**: External environmental sounds
- **User Interface**: Menu sounds and system notifications
- **Ground Communications**: ATC and ground control audio

### FMOD Integration Levels

The API supports two integration approaches:

1. **Simple Integration**: Use built-in PCM playback functions

   - No FMOD linking required
   - Limited to basic audio operations
   - Ideal for simple sound effects
2. **Advanced Integration**: Direct FMOD system access

   - Requires FMOD headers and linking
   - Full FMOD feature access
   - Advanced audio processing capabilities

### Audio Device Management

X-Plane may use separate audio devices:

- **Primary Output**: Main speakers/headphones for environment audio
- **Radio Output**: Dedicated headset for radio communications
- **Automatic Routing**: System handles device selection transparently

## Data Types and Enumerations

### XPLMAudioBus

```c
enum {
    xplm_AudioRadioCom1        = 0,  // COM1 radio channel
    xplm_AudioRadioCom2        = 1,  // COM2 radio channel  
    xplm_AudioRadioPilot       = 2,  // Pilot radio channel
    xplm_AudioRadioCopilot     = 3,  // Copilot radio channel
    xplm_AudioExteriorAircraft = 4,  // External aircraft sounds
    xplm_AudioExteriorEnvironment = 5, // Environmental sounds
    xplm_AudioExteriorUnprocessed = 6, // External unprocessed audio
    xplm_AudioInterior         = 7,  // Interior aircraft sounds
    xplm_AudioUI              = 8,  // User interface sounds
    xplm_AudioGround          = 9,  // Ground/ATC communications
    xplm_Master               = 10   // Master audio bus
};
typedef int XPLMAudioBus;
```

Audio bus routing determines:

- **Signal Processing**: Different effects applied per bus
- **Volume Controls**: Independent volume adjustment
- **Output Device**: Routing to main or headset outputs
- **Audio Context**: Spatial processing appropriate to source type

### XPLMBankID

```c
enum {
    xplm_MasterBank = 0,    // Main FMOD sound bank
    xplm_RadioBank  = 1     // Radio-specific sound bank
};
typedef int XPLMBankID;
```

FMOD banks represent different audio resource collections:

- **Master Bank**: Core aircraft and environment sounds
- **Radio Bank**: Communication-specific audio resources

### FMOD Data Types

When linking with FMOD, these types become available:

```c
typedef enum FMOD_RESULT {
    FMOD_OK = 0,  // Operation successful
    // Additional FMOD result codes available with full FMOD linking
} FMOD_RESULT;

typedef enum FMOD_SOUND_FORMAT {
    FMOD_SOUND_FORMAT_PCM16 = 2  // 16-bit PCM audio format
} FMOD_SOUND_FORMAT;

typedef struct FMOD_VECTOR {
    float x, y, z;  // 3D position/velocity vector
} FMOD_VECTOR;

typedef void FMOD_CHANNEL;  // Opaque FMOD channel handle
```

### XPLMPCMComplete_f

```c
typedef void (*XPLMPCMComplete_f)(
    void        *inRefcon,  // User reference data
    FMOD_RESULT status      // Completion status
);
```

Callback function called when PCM audio completes or is stopped:

- **inRefcon**: User data provided during playback start
- **status**: FMOD result code indicating success/failure

## Core Audio Functions (XPLM400+)

### XPLMPlayPCMOnBus

```c
XPLM_API FMOD_CHANNEL* XPLMPlayPCMOnBus(
    void                *audioBuffer,     // PCM audio data
    uint32_t            bufferSize,       // Buffer size in bytes
    FMOD_SOUND_FORMAT   soundFormat,      // Audio format
    int                 freqHz,           // Sample rate
    int                 numChannels,      // Channel count (1=mono, 2=stereo)
    int                 loop,             // Loop flag (0=once, 1=loop)
    XPLMAudioBus        audioType,        // Target audio bus
    XPLMPCMComplete_f   inCallback,       // Completion callback (optional)
    void                *inRefcon         // User reference data
);
```

Plays in-memory PCM audio data on specified audio bus.

**Parameters**:

- **audioBuffer**: Raw PCM audio data in memory
- **bufferSize**: Size of audio buffer in bytes
- **soundFormat**: Currently only `FMOD_SOUND_FORMAT_PCM16` supported
- **freqHz**: Sample rate (typically 22050, 44100, or 48000)
- **numChannels**: 1 for mono, 2 for stereo
- **loop**: 0 for one-shot playback, 1 for continuous loop
- **audioType**: Target audio bus for routing
- **inCallback**: Optional callback for completion notification
- **inRefcon**: User data passed to callback

**Returns**: FMOD channel handle for further control, or NULL on failure

**Important Notes**:

- Audio starts on next frame, not immediately
- Keep audio buffer valid until completion callback
- Set initial position before first frame if needed for 3D audio

**Example - Simple Sound Effect**:

```c
// Static audio data (embedded or loaded)
static int16_t beepSound[1000]; // 1000 samples of beep
static int soundInitialized = 0;

void PlayBeepSound(void) {
    if (!soundInitialized) {
        // Generate simple beep sound
        for (int i = 0; i < 1000; i++) {
            beepSound[i] = (int16_t)(sin(i * 2.0 * M_PI * 440.0 / 22050.0) * 16000);
        }
        soundInitialized = 1;
    }
  
    FMOD_CHANNEL* channel = XPLMPlayPCMOnBus(
        beepSound,                    // Audio buffer
        sizeof(beepSound),            // Buffer size
        FMOD_SOUND_FORMAT_PCM16,      // 16-bit PCM
        22050,                        // 22kHz sample rate
        1,                            // Mono
        0,                            // No loop
        xplm_AudioUI,                 // UI audio bus
        NULL,                         // No callback
        NULL                          // No reference data
    );
  
    if (!channel) {
        XPLMDebugString("Failed to play beep sound\n");
    }
}
```

**Example - 3D Positioned Audio**:

```c
typedef struct {
    int16_t *audioData;
    int dataSize;
    char name[64];
} SoundEffect;

void SoundCompletionCallback(void *refcon, FMOD_RESULT status) {
    SoundEffect *sound = (SoundEffect*)refcon;
  
    if (status == FMOD_OK) {
        XPLMDebugString("Sound completed: ");
        XPLMDebugString(sound->name);
        XPLMDebugString("\n");
    } else {
        XPLMDebugString("Sound failed: ");
        XPLMDebugString(sound->name);
        XPLMDebugString("\n");
    }
  
    // Clean up if dynamically allocated
    free(sound->audioData);
    free(sound);
}

FMOD_CHANNEL* PlayEngineSound(float x, float y, float z) {
    // Load or generate engine sound data
    SoundEffect *sound = malloc(sizeof(SoundEffect));
    sound->dataSize = LoadEngineAudioData(&sound->audioData);
    strcpy(sound->name, "Engine");
  
    FMOD_CHANNEL* channel = XPLMPlayPCMOnBus(
        sound->audioData,
        sound->dataSize,
        FMOD_SOUND_FORMAT_PCM16,
        44100,                        // High quality for engine
        1,                            // Mono for easier 3D positioning
        1,                            // Loop continuously
        xplm_AudioExteriorAircraft,   // External aircraft bus
        SoundCompletionCallback,
        sound
    );
  
    if (channel) {
        // Position the sound in 3D space
        FMOD_VECTOR position = {x, y, z};
        FMOD_VECTOR velocity = {0, 0, 0};
        XPLMSetAudioPosition(channel, &position, &velocity);
      
        // Set appropriate fade distances for engine sound
        XPLMSetAudioFadeDistance(channel, 10.0f, 1000.0f);
    }
  
    return channel;
}
```

### XPLMStopAudio

```c
XPLM_API FMOD_RESULT XPLMStopAudio(FMOD_CHANNEL* fmod_channel);
```

Stops playing audio channel immediately.

**Parameters**:

- **fmod_channel**: Channel handle from `XPLMPlayPCMOnBus`

**Returns**: FMOD result code indicating success/failure

**Effects**:

- Audio stops immediately
- Completion callback called if registered
- Channel handle becomes invalid after this call

**Example**:

```c
static FMOD_CHANNEL* gEngineChannel = NULL;

void StopEngineSound(void) {
    if (gEngineChannel) {
        FMOD_RESULT result = XPLMStopAudio(gEngineChannel);
        if (result == FMOD_OK) {
            XPLMDebugString("Engine sound stopped successfully\n");
        } else {
            XPLMDebugString("Failed to stop engine sound\n");
        }
        gEngineChannel = NULL; // Handle is now invalid
    }
}
```

## 3D Audio Control Functions

### XPLMSetAudioPosition

```c
XPLM_API FMOD_RESULT XPLMSetAudioPosition(
    FMOD_CHANNEL *fmod_channel,  // Audio channel to position
    FMOD_VECTOR  *position,      // 3D position in local coordinates
    FMOD_VECTOR  *velocity       // 3D velocity for Doppler effect
);
```

Positions audio source in 3D space with velocity for Doppler effect.

**Parameters**:

- **fmod_channel**: Channel to position
- **position**: 3D location in OpenGL local coordinates
- **velocity**: 3D velocity vector in meters/second

**Returns**: FMOD result code

**Effects**:

- Enables 3D audio processing automatically
- Distance attenuation applied
- Doppler shift based on velocity
- Stereo/surround positioning

**Example - Moving Vehicle Audio**:

```c
typedef struct {
    FMOD_CHANNEL *channel;
    float x, y, z;           // Current position
    float vx, vy, vz;        // Velocity
    float lastUpdateTime;
} MovingAudioSource;

void UpdateMovingAudio(MovingAudioSource *source, float deltaTime) {
    if (!source->channel) return;
  
    // Update position based on velocity
    source->x += source->vx * deltaTime;
    source->y += source->vy * deltaTime; 
    source->z += source->vz * deltaTime;
  
    // Set new audio position
    FMOD_VECTOR position = {source->x, source->y, source->z};
    FMOD_VECTOR velocity = {source->vx, source->vy, source->vz};
  
    FMOD_RESULT result = XPLMSetAudioPosition(source->channel, &position, &velocity);
  
    if (result != FMOD_OK) {
        XPLMDebugString("Failed to update audio position\n");
    }
}

// Usage in flight loop
float AudioUpdateCallback(float elapsed, float total, int counter, void *ref) {
    MovingAudioSource *sources = (MovingAudioSource*)ref;
  
    for (int i = 0; i < sourceCount; i++) {
        UpdateMovingAudio(&sources[i], elapsed);
    }
  
    return 0.05f; // Update every 50ms for smooth audio
}
```

### XPLMSetAudioFadeDistance

```c
XPLM_API FMOD_RESULT XPLMSetAudioFadeDistance(
    FMOD_CHANNEL *fmod_channel,      // Audio channel to configure
    float        min_fade_distance,  // Distance where volume starts decreasing
    float        max_fade_distance   // Distance where volume reaches zero
);
```

Controls 3D distance attenuation for audio source.

**Parameters**:

- **fmod_channel**: Channel to configure
- **min_fade_distance**: Distance where attenuation begins (meters)
- **max_fade_distance**: Distance where sound becomes inaudible (meters)

**Returns**: FMOD result code

**Special Values**:

- Negative values for both parameters: Convert 3D sound back to 2D
- Zero values: Use FMOD defaults (not recommended)

**Example - Different Sound Types**:

```c
void SetupAudioDistances(FMOD_CHANNEL *channel, const char *soundType) {
    if (strcmp(soundType, "engine") == 0) {
        // Engine sounds - audible from far away
        XPLMSetAudioFadeDistance(channel, 50.0f, 2000.0f);
    }
    else if (strcmp(soundType, "gear") == 0) {
        // Landing gear - close range mechanical sound
        XPLMSetAudioFadeDistance(channel, 5.0f, 100.0f);
    }
    else if (strcmp(soundType, "wind") == 0) {
        // Wind effects - medium range environmental sound
        XPLMSetAudioFadeDistance(channel, 10.0f, 500.0f);
    }
    else if (strcmp(soundType, "cockpit") == 0) {
        // Cockpit sounds - close range, not 3D
        XPLMSetAudioFadeDistance(channel, -1.0f, -1.0f); // 2D audio
    }
}
```

### XPLMSetAudioVolume

```c
XPLM_API FMOD_RESULT XPLMSetAudioVolume(
    FMOD_CHANNEL *fmod_channel,  // Audio channel to adjust
    float        source_volume   // Volume level (0.0 to 1.0+)
);
```

Sets the source volume level for audio channel.

**Parameters**:

- **fmod_channel**: Channel to adjust
- **source_volume**: Volume multiplier (0.0 = silent, 1.0 = full, >1.0 = amplified)

**Returns**: FMOD result code

**Usage Notes**:

- Use for source volume changes, not distance fading
- Values >1.0 can amplify quiet sounds but may cause distortion
- Combines with distance attenuation and bus volume controls

**Example - Dynamic Volume Control**:

```c
void UpdateEngineVolume(FMOD_CHANNEL *engineChannel) {
    // Get engine power from X-Plane
    float enginePower = XPLMGetDataf(enginePowerRef);    // 0.0 to 1.0
    float engineRPM = XPLMGetDataf(engineRPMRef);        // RPM value
  
    // Calculate volume based on engine state
    float volume = 0.1f + (enginePower * 0.9f); // Base volume + power
  
    // Reduce volume at very low RPM
    if (engineRPM < 800.0f) {
        volume *= (engineRPM / 800.0f);
    }
  
    XPLMSetAudioVolume(engineChannel, volume);
}

// Smooth volume transitions
void FadeAudioVolume(FMOD_CHANNEL *channel, float targetVolume, float fadeTime) {
    static float currentVolume = 1.0f;
    static float lastFadeTime = 0.0f;
  
    float currentTime = XPLMGetElapsedTime();
    float deltaTime = currentTime - lastFadeTime;
    lastFadeTime = currentTime;
  
    if (deltaTime > 0.0f) {
        float fadeRate = 1.0f / fadeTime; // Volume units per second
        float change = fadeRate * deltaTime;
      
        if (currentVolume < targetVolume) {
            currentVolume = fmin(currentVolume + change, targetVolume);
        } else if (currentVolume > targetVolume) {
            currentVolume = fmax(currentVolume - change, targetVolume);
        }
      
        XPLMSetAudioVolume(channel, currentVolume);
    }
}
```

### XPLMSetAudioPitch

```c
XPLM_API FMOD_RESULT XPLMSetAudioPitch(
    FMOD_CHANNEL *fmod_channel,  // Audio channel to adjust
    float        audio_pitch_hz  // Pitch adjustment in Hz
);
```

Changes the playback pitch/frequency of audio channel.

**Parameters**:

- **fmod_channel**: Channel to adjust
- **audio_pitch_hz**: Pitch multiplier (1.0 = normal, 2.0 = octave up, 0.5 = octave down)

**Returns**: FMOD result code

**Effects**:

- Changes both pitch and playback speed
- Values >1.0 increase pitch and speed
- Values <1.0 decrease pitch and speed

**Example - Engine Pitch Control**:

```c
void UpdateEnginePitch(FMOD_CHANNEL *engineChannel) {
    // Get engine RPM from X-Plane
    float engineRPM = XPLMGetDataf(engineRPMRef);
    float idleRPM = 800.0f;
    float maxRPM = 2700.0f;
  
    // Calculate pitch based on RPM (simulate engine pitch change)
    float rpmRatio = (engineRPM - idleRPM) / (maxRPM - idleRPM);
    rpmRatio = fmax(0.0f, fmin(1.0f, rpmRatio)); // Clamp to 0-1
  
    // Pitch range: 0.8 (low RPM) to 1.5 (high RPM) 
    float pitch = 0.8f + (rpmRatio * 0.7f);
  
    XPLMSetAudioPitch(engineChannel, pitch);
}

// Doppler effect simulation (alternative to automatic velocity-based Doppler)
void ApplyDopplerEffect(FMOD_CHANNEL *channel, float relativeVelocity) {
    float soundSpeed = 343.0f; // Speed of sound in m/s
  
    // Calculate Doppler shift
    float dopplerRatio = soundSpeed / (soundSpeed - relativeVelocity);
  
    // Clamp to reasonable range
    dopplerRatio = fmax(0.5f, fmin(2.0f, dopplerRatio));
  
    XPLMSetAudioPitch(channel, dopplerRatio);
}
```

### XPLMSetAudioCone

```c
XPLM_API FMOD_RESULT XPLMSetAudioCone(
    FMOD_CHANNEL *fmod_channel,        // Audio channel to configure
    float        inside_cone_angle,    // Inner cone angle in degrees
    float        outside_cone_angle,   // Outer cone angle in degrees  
    float        outside_volume,       // Volume outside outer cone
    FMOD_VECTOR  *orientation         // Cone direction vector
);
```

Creates directional audio cone for realistic sound projection.

**Parameters**:

- **fmod_channel**: Channel to make directional
- **inside_cone_angle**: Full volume cone angle (degrees, 0-360)
- **outside_cone_angle**: Outer cone boundary (degrees, larger than inside)
- **outside_volume**: Volume multiplier outside outer cone (0.0-1.0)
- **orientation**: Direction vector in local coordinates

**Returns**: FMOD result code

**Behavior**:

- Full volume inside inner cone
- Linear fade between inner and outer cone
- Constant reduced volume outside outer cone
- Automatically enables 3D processing

**Example - Directional Engine Sound**:

```c
void CreateDirectionalEngineSound(float x, float y, float z, float heading) {
    FMOD_CHANNEL* channel = XPLMPlayPCMOnBus(
        engineSoundData, engineSoundSize,
        FMOD_SOUND_FORMAT_PCM16, 44100, 1, 1,
        xplm_AudioExteriorAircraft, NULL, NULL
    );
  
    if (channel) {
        // Position the engine
        FMOD_VECTOR position = {x, y, z};
        FMOD_VECTOR velocity = {0, 0, 0};
        XPLMSetAudioPosition(channel, &position, &velocity);
      
        // Create directional cone pointing forward from aircraft
        float headingRad = heading * M_PI / 180.0f;
        FMOD_VECTOR orientation = {
            sin(headingRad),  // X component
            0.0f,             // Y component (level)
            -cos(headingRad)  // Z component
        };
      
        XPLMSetAudioCone(
            channel,
            45.0f,      // 45-degree inner cone (full volume forward)
            120.0f,     // 120-degree outer cone
            0.3f,       // 30% volume behind aircraft
            &orientation
        );
      
        XPLMSetAudioFadeDistance(channel, 20.0f, 1000.0f);
    }
}

// Exhaust pipe directional sound
void CreateExhaustSound(float x, float y, float z) {
    FMOD_CHANNEL* channel = PlayExhaustAudio();
  
    if (channel) {
        // Exhaust points backward and down
        FMOD_VECTOR exhaustDirection = {0.0f, -0.3f, 1.0f}; // Backward and down
      
        XPLMSetAudioCone(
            channel,
            30.0f,      // Narrow forward projection
            90.0f,      // Wider outer cone
            0.1f,       // Very quiet from front
            &exhaustDirection
        );
    }
}
```

## Advanced FMOD Integration (XPLM400+)

### XPLMGetFMODStudio

```c
XPLM_API FMOD_STUDIO_SYSTEM* XPLMGetFMODStudio(void);
```

**Requires**: `#include <fmod.h>` or `#include <fmod.hpp>` before XPLMSound.h

Returns direct access to X-Plane's FMOD Studio system for advanced audio operations.

**Use Cases**:

- Custom audio effects processing
- Advanced mixing and routing
- Complex event-based audio systems
- Professional audio plugin development

**Example - Advanced FMOD Usage**:

```c
#include <fmod_studio.h>
#include "XPLMSound.h"

void SetupAdvancedAudio(void) {
    FMOD_STUDIO_SYSTEM* studioSystem = XPLMGetFMODStudio();
  
    if (studioSystem) {
        // Get core FMOD system
        FMOD_SYSTEM* coreSystem;
        FMOD_Studio_System_GetCoreSystem(studioSystem, &coreSystem);
      
        // Create custom DSP effects
        FMOD_DSP* reverbDSP;
        FMOD_System_CreateDSPByType(coreSystem, FMOD_DSP_TYPE_SFXREVERB, &reverbDSP);
      
        // Configure reverb parameters
        FMOD_DSP_SetParameterFloat(reverbDSP, FMOD_DSP_SFXREVERB_DECAYTIME, 2.0f);
        FMOD_DSP_SetParameterFloat(reverbDSP, FMOD_DSP_SFXREVERB_WETLEVEL, 0.3f);
      
        // Apply to specific channel group
        FMOD_CHANNELGROUP* interiorGroup = XPLMGetFMODChannelGroup(xplm_AudioInterior);
        FMOD_ChannelGroup_AddDSP(interiorGroup, FMOD_CHANNELCONTROL_DSP_TAIL, reverbDSP);
    }
}
```

### XPLMGetFMODChannelGroup

```c
XPLM_API FMOD_CHANNELGROUP* XPLMGetFMODChannelGroup(XPLMAudioBus audioType);
```

**Requires**: FMOD headers

Returns FMOD channel group for specific audio bus, enabling advanced mixing control.

**Parameters**:

- **audioType**: Audio bus identifier

**Returns**: FMOD channel group handle or NULL

**Use Cases**:

- Bus-wide effects processing
- Advanced volume automation
- Custom mixing scenarios
- Audio analysis and visualization

**Example - Bus Effects Processing**:

```c
void SetupBusEffects(void) {
    // Add compression to exterior aircraft sounds
    FMOD_CHANNELGROUP* exteriorGroup = XPLMGetFMODChannelGroup(xplm_AudioExteriorAircraft);
  
    if (exteriorGroup) {
        FMOD_DSP* compressor;
        FMOD_SYSTEM* system;
      
        // Get system from channel group
        FMOD_ChannelGroup_GetSystemObject(exteriorGroup, &system);
        FMOD_System_CreateDSPByType(system, FMOD_DSP_TYPE_COMPRESSOR, &compressor);
      
        // Configure compressor
        FMOD_DSP_SetParameterFloat(compressor, FMOD_DSP_COMPRESSOR_THRESHOLD, -20.0f);
        FMOD_DSP_SetParameterFloat(compressor, FMOD_DSP_COMPRESSOR_RATIO, 4.0f);
      
        // Add to channel group
        FMOD_ChannelGroup_AddDSP(exteriorGroup, FMOD_CHANNELCONTROL_DSP_TAIL, compressor);
    }
  
    // Add radio static to COM channels
    FMOD_CHANNELGROUP* com1Group = XPLMGetFMODChannelGroup(xplm_AudioRadioCom1);
    if (com1Group) {
        ApplyRadioStaticEffect(com1Group);
    }
}
```

## Implementation Guidelines

### Audio Resource Management

```c
typedef struct {
    FMOD_CHANNEL *channel;
    char name[64];
    float x, y, z;
    int isActive;
    float volume;
    void *audioData;
    size_t dataSize;
} AudioSource;

static AudioSource gAudioSources[MAX_AUDIO_SOURCES];
static int gSourceCount = 0;

int CreateAudioSource(const char *name, void *data, size_t size, 
                      XPLMAudioBus bus) {
    if (gSourceCount >= MAX_AUDIO_SOURCES) return -1;
  
    AudioSource *source = &gAudioSources[gSourceCount];
    strcpy(source->name, name);
    source->audioData = malloc(size);
    memcpy(source->audioData, data, size);
    source->dataSize = size;
    source->isActive = 0;
    source->volume = 1.0f;
  
    return gSourceCount++;
}

void PlayAudioSource(int sourceID, float x, float y, float z) {
    if (sourceID < 0 || sourceID >= gSourceCount) return;
  
    AudioSource *source = &gAudioSources[sourceID];
  
    source->channel = XPLMPlayPCMOnBus(
        source->audioData, source->dataSize,
        FMOD_SOUND_FORMAT_PCM16, 44100, 1, 0,
        xplm_AudioExteriorAircraft, 
        AudioCompletionCallback, source
    );
  
    if (source->channel) {
        FMOD_VECTOR pos = {x, y, z};
        FMOD_VECTOR vel = {0, 0, 0};
      
        XPLMSetAudioPosition(source->channel, &pos, &vel);
        XPLMSetAudioVolume(source->channel, source->volume);
      
        source->x = x; source->y = y; source->z = z;
        source->isActive = 1;
    }
}

void AudioCompletionCallback(void *refcon, FMOD_RESULT status) {
    AudioSource *source = (AudioSource*)refcon;
    source->isActive = 0;
    source->channel = NULL;
}

void CleanupAudioSources(void) {
    for (int i = 0; i < gSourceCount; i++) {
        if (gAudioSources[i].channel) {
            XPLMStopAudio(gAudioSources[i].channel);
        }
        free(gAudioSources[i].audioData);
    }
    gSourceCount = 0;
}
```

### Dynamic Audio Loading

```c
typedef struct {
    char filename[256];
    void *data;
    size_t size;
    int loaded;
} AudioFile;

static AudioFile gAudioFiles[50];
static int gFileCount = 0;

int LoadAudioFile(const char *filename) {
    if (gFileCount >= 50) return -1;
  
    AudioFile *file = &gAudioFiles[gFileCount];
    strcpy(file->filename, filename);
  
    // Load file in background thread (pseudo-code)
    FILE *f = fopen(filename, "rb");
    if (!f) return -1;
  
    fseek(f, 0, SEEK_END);
    file->size = ftell(f);
    fseek(f, 0, SEEK_SET);
  
    file->data = malloc(file->size);
    fread(file->data, 1, file->size, f);
    fclose(f);
  
    file->loaded = 1;
  
    return gFileCount++;
}

FMOD_CHANNEL* PlayAudioFile(int fileID, XPLMAudioBus bus) {
    if (fileID < 0 || fileID >= gFileCount) return NULL;
  
    AudioFile *file = &gAudioFiles[fileID];
    if (!file->loaded) return NULL;
  
    return XPLMPlayPCMOnBus(
        file->data, file->size,
        FMOD_SOUND_FORMAT_PCM16, 44100, 2, 0, // Assume stereo
        bus, NULL, NULL
    );
}
```

### Audio System Integration

```c
typedef struct {
    float masterVolume;
    float engineVolume;  
    float environmentVolume;
    float radioVolume;
    int muteAll;
} AudioSettings;

static AudioSettings gAudioSettings = {1.0f, 1.0f, 1.0f, 1.0f, 0};

void ApplyAudioSettings(void) {
    if (gAudioSettings.muteAll) {
        // Mute all plugin audio
        for (int i = 0; i < gSourceCount; i++) {
            if (gAudioSources[i].isActive) {
                XPLMSetAudioVolume(gAudioSources[i].channel, 0.0f);
            }
        }
        return;
    }
  
    // Apply volume settings by category
    for (int i = 0; i < gSourceCount; i++) {
        if (!gAudioSources[i].isActive) continue;
      
        float volume = gAudioSources[i].volume * gAudioSettings.masterVolume;
      
        // Apply category-specific volume
        if (strstr(gAudioSources[i].name, "engine")) {
            volume *= gAudioSettings.engineVolume;
        }
        else if (strstr(gAudioSources[i].name, "environment")) {
            volume *= gAudioSettings.environmentVolume;
        }
        else if (strstr(gAudioSources[i].name, "radio")) {
            volume *= gAudioSettings.radioVolume;
        }
      
        XPLMSetAudioVolume(gAudioSources[i].channel, volume);
    }
}
```

## Common Use Cases

### 1. Aircraft Engine Sound System

```c
typedef struct {
    FMOD_CHANNEL *idleChannel;
    FMOD_CHANNEL *runupChannel;
    FMOD_CHANNEL *cruiseChannel;
    float currentRPM;
    float targetRPM;
    float x, y, z;  // Engine position
} EngineAudio;

void InitEngineAudio(EngineAudio *engine, float x, float y, float z) {
    engine->x = x; engine->y = y; engine->z = z;
    engine->currentRPM = 0.0f;
    engine->targetRPM = 0.0f;
  
    // Load different engine sound layers
    engine->idleChannel = XPLMPlayPCMOnBus(
        idleSound, idleSoundSize,
        FMOD_SOUND_FORMAT_PCM16, 44100, 1, 1, // Looping
        xplm_AudioExteriorAircraft, NULL, NULL
    );
  
    engine->runupChannel = XPLMPlayPCMOnBus(
        runupSound, runupSoundSize,
        FMOD_SOUND_FORMAT_PCM16, 44100, 1, 1, // Looping
        xplm_AudioExteriorAircraft, NULL, NULL
    );
  
    engine->cruiseChannel = XPLMPlayPCMOnBus(
        cruiseSound, cruiseSoundSize,
        FMOD_SOUND_FORMAT_PCM16, 44100, 1, 1, // Looping
        xplm_AudioExteriorAircraft, NULL, NULL
    );
  
    // Position all channels
    FMOD_VECTOR pos = {x, y, z};
    FMOD_VECTOR vel = {0, 0, 0};
  
    XPLMSetAudioPosition(engine->idleChannel, &pos, &vel);
    XPLMSetAudioPosition(engine->runupChannel, &pos, &vel);
    XPLMSetAudioPosition(engine->cruiseChannel, &pos, &vel);
  
    // Set fade distances
    XPLMSetAudioFadeDistance(engine->idleChannel, 20.0f, 800.0f);
    XPLMSetAudioFadeDistance(engine->runupChannel, 30.0f, 1200.0f);
    XPLMSetAudioFadeDistance(engine->cruiseChannel, 50.0f, 2000.0f);
}

void UpdateEngineAudio(EngineAudio *engine, float deltaTime) {
    float enginePower = XPLMGetDataf(enginePowerRef);
    engine->targetRPM = 800.0f + (enginePower * 2000.0f); // 800-2800 RPM
  
    // Smooth RPM changes
    float rpmChange = 500.0f * deltaTime; // Max 500 RPM/second change
    if (engine->currentRPM < engine->targetRPM) {
        engine->currentRPM = fmin(engine->currentRPM + rpmChange, engine->targetRPM);
    } else {
        engine->currentRPM = fmax(engine->currentRPM - rpmChange, engine->targetRPM);
    }
  
    // Mix audio layers based on RPM
    float idleVolume = 1.0f;
    float runupVolume = 0.0f;
    float cruiseVolume = 0.0f;
  
    if (engine->currentRPM > 1200.0f) {
        float runupRatio = (engine->currentRPM - 1200.0f) / 800.0f; // 1200-2000 RPM
        runupRatio = fmin(1.0f, runupRatio);
        runupVolume = runupRatio;
        idleVolume = 1.0f - (runupRatio * 0.7f);
    }
  
    if (engine->currentRPM > 2000.0f) {
        float cruiseRatio = (engine->currentRPM - 2000.0f) / 800.0f; // 2000-2800 RPM
        cruiseRatio = fmin(1.0f, cruiseRatio);
        cruiseVolume = cruiseRatio;
        runupVolume = 1.0f - (cruiseRatio * 0.5f);
    }
  
    // Apply volumes
    XPLMSetAudioVolume(engine->idleChannel, idleVolume);
    XPLMSetAudioVolume(engine->runupChannel, runupVolume);
    XPLMSetAudioVolume(engine->cruiseChannel, cruiseVolume);
  
    // Adjust pitch based on RPM
    float pitchRatio = engine->currentRPM / 1800.0f; // Baseline RPM
    XPLMSetAudioPitch(engine->idleChannel, 0.8f + (pitchRatio * 0.4f));
    XPLMSetAudioPitch(engine->runupChannel, 0.9f + (pitchRatio * 0.3f));
    XPLMSetAudioPitch(engine->cruiseChannel, 1.0f + (pitchRatio * 0.2f));
}
```

### 2. Radio Communication System

```c
typedef struct {
    FMOD_CHANNEL *staticChannel;
    FMOD_CHANNEL *voiceChannel;
    float signalStrength;
    int isTransmitting;
    float frequency;
} RadioAudio;

void InitRadioAudio(RadioAudio *radio) {
    // Continuous static background
    radio->staticChannel = XPLMPlayPCMOnBus(
        radioStaticSound, radioStaticSize,
        FMOD_SOUND_FORMAT_PCM16, 22050, 1, 1, // Loop static
        xplm_AudioRadioCom1, NULL, NULL
    );
  
    XPLMSetAudioVolume(radio->staticChannel, 0.1f); // Low static
    radio->signalStrength = 1.0f;
    radio->isTransmitting = 0;
}

void PlayRadioTransmission(RadioAudio *radio, void *voiceData, size_t dataSize) {
    // Stop previous transmission
    if (radio->voiceChannel) {
        XPLMStopAudio(radio->voiceChannel);
    }
  
    // Play voice transmission
    radio->voiceChannel = XPLMPlayPCMOnBus(
        voiceData, dataSize,
        FMOD_SOUND_FORMAT_PCM16, 22050, 1, 0, // No loop
        xplm_AudioRadioCom1,
        RadioTransmissionComplete, radio
    );
  
    if (radio->voiceChannel) {
        // Adjust static level during transmission
        XPLMSetAudioVolume(radio->staticChannel, 0.3f * (1.0f - radio->signalStrength));
      
        // Apply radio filtering effect (simulate compression/distortion)
        XPLMSetAudioPitch(radio->voiceChannel, 1.0f);
      
        radio->isTransmitting = 1;
    }
}

void RadioTransmissionComplete(void *refcon, FMOD_RESULT status) {
    RadioAudio *radio = (RadioAudio*)refcon;
    radio->isTransmitting = 0;
    radio->voiceChannel = NULL;
  
    // Return static to normal level
    XPLMSetAudioVolume(radio->staticChannel, 0.1f);
}
```

### 3. Environmental Audio System

```c
typedef struct {
    FMOD_CHANNEL *windChannel;
    FMOD_CHANNEL *rainChannel;
    float windSpeed;
    float precipitation;
    float altitude;
} WeatherAudio;

void UpdateWeatherAudio(WeatherAudio *weather) {
    // Get weather data from X-Plane
    weather->windSpeed = XPLMGetDataf(windSpeedRef);
    weather->precipitation = XPLMGetDataf(precipitationRef);
    weather->altitude = XPLMGetDataf(altitudeRef);
  
    // Wind sound intensity based on airspeed and wind
    float airspeed = XPLMGetDataf(airspeedRef);
    float windIntensity = fmin(1.0f, (airspeed + weather->windSpeed) / 50.0f);
  
    if (windIntensity > 0.1f) {
        if (!weather->windChannel) {
            weather->windChannel = XPLMPlayPCMOnBus(
                windSound, windSoundSize,
                FMOD_SOUND_FORMAT_PCM16, 44100, 2, 1,
                xplm_AudioExteriorEnvironment, NULL, NULL
            );
        }
      
        XPLMSetAudioVolume(weather->windChannel, windIntensity);
        XPLMSetAudioPitch(weather->windChannel, 0.8f + (windIntensity * 0.4f));
    } else if (weather->windChannel) {
        XPLMStopAudio(weather->windChannel);
        weather->windChannel = NULL;
    }
  
    // Rain sound based on precipitation
    if (weather->precipitation > 0.1f) {
        if (!weather->rainChannel) {
            weather->rainChannel = XPLMPlayPCMOnBus(
                rainSound, rainSoundSize,
                FMOD_SOUND_FORMAT_PCM16, 44100, 2, 1,
                xplm_AudioExteriorEnvironment, NULL, NULL
            );
        }
      
        XPLMSetAudioVolume(weather->rainChannel, weather->precipitation);
    } else if (weather->rainChannel) {
        XPLMStopAudio(weather->rainChannel);
        weather->rainChannel = NULL;
    }
}
```

## Performance Optimization

### Audio Channel Management

- **Channel Limits**: FMOD has finite channel limits - manage active channels carefully
- **Distance Culling**: Stop distant audio sources to free channels
- **Priority Systems**: Use priority levels for important vs. ambient sounds
- **Channel Pooling**: Reuse channels when possible

### Memory Management

```c
// Good - Pre-load frequently used sounds
void PreloadCommonSounds(void) {
    LoadAudioFile("sounds/engine_idle.wav");
    LoadAudioFile("sounds/gear_retract.wav");
    LoadAudioFile("sounds/flap_move.wav");
}

// Bad - Loading sounds during gameplay
void BadSoundTrigger(void) {
    void *data = LoadSoundFileNow("sounds/emergency.wav"); // Blocks sim!
    XPLMPlayPCMOnBus(data, size, format, freq, channels, loop, bus, NULL, NULL);
}
```

### CPU Usage Optimization

- **Update Frequency**: Don't update audio parameters every frame
- **Spatial Optimization**: Group nearby audio sources
- **LOD Systems**: Reduce audio quality for distant sources
- **Conditional Processing**: Skip updates for inactive audio

## Integration with X-Plane Systems

### Message Integration

```c
PLUGIN_API void XPluginReceiveMessage(XPLMPluginID from, int msg, void *param) {
    switch (msg) {
        case XPLM_MSG_FMOD_BANK_LOADED:
            if ((XPLMBankID)(intptr_t)param == xplm_MasterBank) {
                InitializeAudioSystem();
            }
            break;
          
        case XPLM_MSG_FMOD_BANK_UNLOADING:
            if ((XPLMBankID)(intptr_t)param == xplm_MasterBank) {
                ShutdownAudioSystem();
            }
            break;
          
        case XPLM_MSG_PLANE_LOADED:
            LoadAircraftSpecificSounds();
            break;
    }
}
```

### Dataref Integration

```c
float AudioFlightLoopCallback(float elapsed, float total, int counter, void *ref) {
    // Update engine audio based on dataref values
    float n1 = XPLMGetDataf(engine1N1Ref);
    float n2 = XPLMGetDataf(engine2N1Ref);
  
    UpdateEngineChannel(engine1Channel, n1);
    UpdateEngineChannel(engine2Channel, n2);
  
    // Update gear audio
    float gearPosition = XPLMGetDataf(gearPositionRef);
    if (gearPosition != lastGearPosition) {
        if (gearPosition > lastGearPosition) {
            PlayGearExtensionSound();
        } else {
            PlayGearRetractionSound();
        }
        lastGearPosition = gearPosition;
    }
  
    return 0.05f; // 20Hz update rate
}
```

## Error Handling and Debugging

### FMOD Error Checking

```c
const char* FMODResultToString(FMOD_RESULT result) {
    switch (result) {
        case FMOD_OK: return "No errors";
        case FMOD_ERR_BADCOMMAND: return "Tried to call a function on a data type that does not allow this type of functionality";
        case FMOD_ERR_CHANNEL_ALLOC: return "Error trying to allocate a channel";
        case FMOD_ERR_CHANNEL_STOLEN: return "The specified channel has been reused to play another sound";
        case FMOD_ERR_DMA: return "DMA Failure";
        case FMOD_ERR_DSP_CONNECTION: return "DSP connection error";
        case FMOD_ERR_DSP_DONTPROCESS: return "DSP return code from a DSP process query callback";
        case FMOD_ERR_DSP_FORMAT: return "DSP Format error";
        case FMOD_ERR_DSP_INUSE: return "DSP is already in the mixer's DSP network";
        case FMOD_ERR_DSP_NOTFOUND: return "DSP connection error";
        case FMOD_ERR_DSP_RESERVED: return "DSP reserved error";
        case FMOD_ERR_DSP_SILENCE: return "DSP return code from a DSP process query callback";
        case FMOD_ERR_DSP_TYPE: return "DSP Type error";
        default: return "Unknown FMOD error";
    }
}

void CheckFMODResult(FMOD_RESULT result, const char *operation) {
    if (result != FMOD_OK) {
        char message[512];
        sprintf(message, "FMOD Error in %s: %s\n", operation, FMODResultToString(result));
        XPLMDebugString(message);
    }
}

// Usage
FMOD_RESULT result = XPLMSetAudioVolume(channel, 0.5f);
CheckFMODResult(result, "SetAudioVolume");
```

### Debug Audio Monitoring

```c
void LogAudioStatus(void) {
    char message[1024];
    int activeChannels = 0;
  
    for (int i = 0; i < gSourceCount; i++) {
        if (gAudioSources[i].isActive) {
            activeChannels++;
        }
    }
  
    sprintf(message, "Audio Status: %d active channels, %d total sources\n", 
            activeChannels, gSourceCount);
    XPLMDebugString(message);
  
    // Log memory usage
    size_t totalMemory = 0;
    for (int i = 0; i < gFileCount; i++) {
        totalMemory += gAudioFiles[i].size;
    }
  
    sprintf(message, "Audio Memory: %zu bytes in %d files\n", 
            totalMemory, gFileCount);
    XPLMDebugString(message);
}
```

## Version Compatibility and Feature Detection

```c
int InitializeAudioSystem(void) {
#if defined(XPLM400)
    // Check if FMOD audio system is available
    FMOD_STUDIO_SYSTEM* studio = XPLMGetFMODStudio();
    if (!studio) {
        XPLMDebugString("FMOD Studio not available\n");
        return 0;
    }
  
    XPLMDebugString("FMOD Studio audio system initialized\n");
    return 1;
#else
    XPLMDebugString("Audio system requires XPLM400+\n");
    return 0;
#endif
}
```

## Summary

The XPLMSound API provides comprehensive audio capabilities for X-Plane plugins:

1. **Simple Audio Playback**: Easy PCM audio playback with minimal setup
2. **3D Spatial Audio**: Full 3D positioning with distance attenuation and Doppler effects
3. **Advanced Control**: Pitch, volume, and directional audio cone control
4. **Professional Integration**: Direct FMOD access for complex audio applications
5. **Performance Optimized**: Designed for real-time simulation requirements

Key principles for successful audio integration:

- **Resource Management**: Pre-load audio data and manage channel allocation
- **Performance Awareness**: Consider CPU and memory impact of audio processing
- **User Experience**: Provide realistic, immersive audio that enhances simulation
- **Error Handling**: Graceful degradation when audio operations fail
- **Integration**: Coordinate with X-Plane's existing audio systems and user preferences
