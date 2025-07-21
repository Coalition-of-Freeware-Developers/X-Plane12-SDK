# XPLMCamera.h - X-Plane 12 SDK Camera Control API Documentation

## Overview

The XPLMCamera API provides comprehensive camera control functionality for X-Plane plugins, enabling developers to create custom views, implement dynamic camera systems, and use X-Plane as a rendering platform. This API allows plugins to programmatically control camera position, orientation, and behavior in real-time.

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Core Concepts](#core-concepts)
- [Data Types and Structures](#data-types-and-structures)
- [Camera Control Functions](#camera-control-functions)
- [Camera Management](#camera-management)
- [Implementation Guidelines](#implementation-guidelines)
- [Use Cases](#use-cases)
- [Performance Considerations](#performance-considerations)
- [Best Practices](#best-practices)
- [Common Patterns](#common-patterns)
- [Error Handling](#error-handling)
- [Advanced Features](#advanced-features)

## Architecture Overview

The Camera Control API operates on a callback-based system where plugins register camera control functions that are called each frame to determine camera positioning. The system is designed around these principles:

- **Frame-by-Frame Control**: Camera position is updated every frame via callbacks
- **Coordinate System**: Uses OpenGL coordinates with consistent orientation rules
- **Duration-Based Control**: Plugins can request temporary or permanent camera control
- **Non-Destructive**: Multiple plugins can potentially control camera sequentially
- **Real-Time**: Guarantees smooth camera motion through per-frame updates

### Camera Coordinate System

The camera system uses OpenGL coordinates with the following conventions:

- **Position**: (X, Y, Z) coordinates in OpenGL space
- **Initial Orientation**: Camera faces level with ground, directly up the negative-Z axis (approximately north)
- **Rotation Order**: Yaw → Pitch → Roll (applied in that order)
- **Coordinate Conversions**: Use XPLMGraphics routines for world-to-local coordinate conversion

## Core Concepts

### Camera Control Lifecycle

1. **Registration**: Plugin registers a camera control callback with desired duration
2. **Active Control**: Callback is called each frame to provide camera position
3. **Termination**: Control ends when duration expires, user changes view, or plugin surrenders control
4. **Cleanup**: Plugin receives notification when losing control (if applicable)

### Control Duration Types

- **Temporary Control**: Until user selects new view (`xplm_ControlCameraUntilViewChanges`)
- **Permanent Control**: Until plugin disabled or forcibly overridden (`xplm_ControlCameraForever`)

### Camera vs. Pilot Head Position

**Important Distinction**: The Camera API controls the entire camera system, not just cockpit pilot head position. For cockpit view adjustments, use pilot head datarefs instead of this API.

## Data Types and Structures

### XPLMCameraControlDuration

```c
typedef int XPLMCameraControlDuration;

enum {
    xplm_ControlCameraUntilViewChanges = 1,  // Temporary control
    xplm_ControlCameraForever         = 2,   // Permanent control
};
```

**Purpose**: Specifies how long the plugin wants to retain camera control.

**Values**:

- `xplm_ControlCameraUntilViewChanges`: Control ends when user picks a new view
- `xplm_ControlCameraForever`: Control continues until plugin disabled or another plugin takes control

### XPLMCameraPosition_t

```c
typedef struct {
    float x;        // X position in OpenGL coordinates
    float y;        // Y position in OpenGL coordinates  
    float z;        // Z position in OpenGL coordinates
    float pitch;    // Pitch rotation in degrees (positive = nose up)
    float heading;  // Heading rotation in degrees (positive = yaw right)
    float roll;     // Roll rotation in degrees (positive = roll right)
    float zoom;     // Zoom factor (1.0 = normal, 2.0 = 2x magnification)
} XPLMCameraPosition_t;
```

**Purpose**: Complete specification of camera position and orientation.

**Field Details**:

- **Position Fields (x, y, z)**: OpenGL coordinates for camera location
- **Orientation Fields (pitch, heading, roll)**: Rotation angles in degrees
- **Zoom Field**: Magnification factor (1.0 = normal view, 2.0 = objects appear twice as large)

**Coordinate System Details**:

- **Positive Pitch**: Camera tilts up (nose up attitude)
- **Positive Heading**: Camera rotates right (clockwise when viewed from above)
- **Positive Roll**: Camera rolls right (clockwise when looking forward)

### XPLMCameraControl_f

```c
typedef int (* XPLMCameraControl_f)(
    XPLMCameraPosition_t *outCameraPosition,  // Camera position to set (can be NULL)
    int                   inIsLosingControl,  // 1 if losing control, 0 otherwise
    void                 *inRefcon            // Plugin reference data
);
```

**Purpose**: Callback function for continuous camera control.

**Parameters**:

- `outCameraPosition`: Pointer to structure to fill with new camera position
- `inIsLosingControl`: Flag indicating whether X-Plane is taking control away
- `inRefcon`: Reference constant passed during registration

**Return Value**:

- **1**: Continue controlling camera (apply new position)
- **0**: Surrender camera control

**Important Notes**:

- When `inIsLosingControl` is 1, `outCameraPosition` will be NULL
- Contents of `outCameraPosition` are undefined when callback is called
- Must always fill all fields of the position structure

## Camera Control Functions

### XPLMControlCamera

```c
XPLM_API void XPLMControlCamera(
    XPLMCameraControlDuration inHowLong,
    XPLMCameraControl_f       inControlFunc,
    void                     *inRefcon
);
```

**Purpose**: Initiates camera control with specified callback and duration.

**Parameters**:

- `inHowLong`: Duration type for camera control
- `inControlFunc`: Callback function for camera positioning (must not be NULL)
- `inRefcon`: Reference data passed to callback

**Usage Example**:

```c
static int MyCameraControlCallback(XPLMCameraPosition_t *outPos, int inIsLosingControl, void *inRefcon) {
    if (inIsLosingControl) {
        // Cleanup when losing control
        return 1;  // Acknowledge loss of control
    }
  
    // Set camera position
    outPos->x = 1000.0f;
    outPos->y = 500.0f;
    outPos->z = 2000.0f;
    outPos->pitch = 10.0f;
    outPos->heading = 45.0f;
    outPos->roll = 0.0f;
    outPos->zoom = 1.0f;
  
    return 1;  // Continue controlling
}

// Register for camera control
XPLMControlCamera(xplm_ControlCameraUntilViewChanges, MyCameraControlCallback, NULL);
```

### XPLMDontControlCamera

```c
XPLM_API void XPLMDontControlCamera(void);
```

**Purpose**: Stops plugin from controlling the camera voluntarily.

**Important Notes**:

- Only call when plugin currently has camera control
- Camera control function will NOT be called with `inIsLosingControl` flag
- X-Plane resumes camera control on next cycle
- Use for graceful surrender of camera control

**Usage Example**:

```c
// Stop controlling camera when done
if (cameraControlFinished) {
    XPLMDontControlCamera();
}
```

## Camera Management

### XPLMIsCameraBeingControlled

```c
XPLM_API int XPLMIsCameraBeingControlled(
    XPLMCameraControlDuration *outCameraControlDuration  // Can be NULL
);
```

**Purpose**: Checks if camera is currently under plugin control.

**Parameters**:

- `outCameraControlDuration`: Optional pointer to receive control duration type

**Return Value**:

- **1**: Camera is being controlled by a plugin
- **0**: Camera is not under plugin control

**Usage Example**:

```c
XPLMCameraControlDuration duration;
if (XPLMIsCameraBeingControlled(&duration)) {
    if (duration == xplm_ControlCameraForever) {
        // Camera is under permanent control
    }
}
```

### XPLMReadCameraPosition

```c
XPLM_API void XPLMReadCameraPosition(
    XPLMCameraPosition_t *outCameraPosition
);
```

**Purpose**: Retrieves current camera position and orientation.

**Parameters**:

- `outCameraPosition`: Pointer to structure to receive current camera state

**Usage Example**:

```c
XPLMCameraPosition_t currentPos;
XPLMReadCameraPosition(&currentPos);

// Use current position for calculations
float currentAltitude = currentPos.y;
float currentHeading = currentPos.heading;
```

## Implementation Guidelines

### Basic Camera Control Setup

```c
#include "XPLMCamera.h"
#include "XPLMDataAccess.h"

// Global variables
static int gHasCameraControl = 0;
static XPLMDataRef gPlaneXRef, gPlaneYRef, gPlaneZRef;

// Camera control callback
static int CameraControlCallback(XPLMCameraPosition_t *outPos, int inIsLosingControl, void *inRefcon) {
    if (inIsLosingControl) {
        gHasCameraControl = 0;
        return 1;
    }
  
    // Get aircraft position
    float planeX = XPLMGetDataf(gPlaneXRef);
    float planeY = XPLMGetDataf(gPlaneYRef);
    float planeZ = XPLMGetDataf(gPlaneZRef);
  
    // Position camera relative to aircraft
    outPos->x = planeX + 50.0f;  // 50 meters behind
    outPos->y = planeY + 10.0f;  // 10 meters above
    outPos->z = planeZ;
    outPos->pitch = -5.0f;       // Look slightly down
    outPos->heading = 0.0f;
    outPos->roll = 0.0f;
    outPos->zoom = 1.0f;
  
    return 1;  // Continue controlling
}

// Initialize camera system
int InitCameraSystem() {
    // Look up aircraft position datarefs
    gPlaneXRef = XPLMFindDataRef("sim/flightmodel/position/local_x");
    gPlaneYRef = XPLMFindDataRef("sim/flightmodel/position/local_y");  
    gPlaneZRef = XPLMFindDataRef("sim/flightmodel/position/local_z");
  
    if (!gPlaneXRef || !gPlaneYRef || !gPlaneZRef) {
        return 0;  // Failed to find datarefs
    }
  
    return 1;
}

// Start camera control
void StartCameraControl() {
    if (!gHasCameraControl) {
        XPLMControlCamera(xplm_ControlCameraUntilViewChanges, CameraControlCallback, NULL);
        gHasCameraControl = 1;
    }
}
```

### Advanced Camera Control with Smooth Transitions

```c
// Smooth camera transition system
typedef struct {
    XPLMCameraPosition_t startPos;
    XPLMCameraPosition_t targetPos;
    float transitionTime;
    float currentTime;
    int isTransitioning;
} CameraTransition_t;

static CameraTransition_t gTransition;

// Interpolate between camera positions
static void InterpolateCameraPosition(XPLMCameraPosition_t *outPos, 
                                     const XPLMCameraPosition_t *startPos,
                                     const XPLMCameraPosition_t *endPos, 
                                     float t) {
    // Clamp t to [0, 1]
    if (t < 0.0f) t = 0.0f;
    if (t > 1.0f) t = 1.0f;
  
    // Apply smooth easing function
    float smoothT = t * t * (3.0f - 2.0f * t);  // Smoothstep
  
    // Interpolate position
    outPos->x = startPos->x + (endPos->x - startPos->x) * smoothT;
    outPos->y = startPos->y + (endPos->y - startPos->y) * smoothT;
    outPos->z = startPos->z + (endPos->z - startPos->z) * smoothT;
  
    // Interpolate rotation (handle wrapping for heading)
    outPos->pitch = startPos->pitch + (endPos->pitch - startPos->pitch) * smoothT;
    outPos->roll = startPos->roll + (endPos->roll - startPos->roll) * smoothT;
  
    // Handle heading wrap-around
    float headingDiff = endPos->heading - startPos->heading;
    if (headingDiff > 180.0f) headingDiff -= 360.0f;
    if (headingDiff < -180.0f) headingDiff += 360.0f;
    outPos->heading = startPos->heading + headingDiff * smoothT;
  
    // Interpolate zoom
    outPos->zoom = startPos->zoom + (endPos->zoom - startPos->zoom) * smoothT;
}

// Smooth transition camera callback
static int SmoothCameraCallback(XPLMCameraPosition_t *outPos, int inIsLosingControl, void *inRefcon) {
    if (inIsLosingControl) {
        gTransition.isTransitioning = 0;
        return 1;
    }
  
    if (gTransition.isTransitioning) {
        gTransition.currentTime += 0.016f;  // Assume ~60 FPS
        float t = gTransition.currentTime / gTransition.transitionTime;
      
        if (t >= 1.0f) {
            // Transition complete
            *outPos = gTransition.targetPos;
            gTransition.isTransitioning = 0;
        } else {
            // Interpolate position
            InterpolateCameraPosition(outPos, &gTransition.startPos, &gTransition.targetPos, t);
        }
    } else {
        // Normal camera positioning logic here
        // ...
    }
  
    return 1;
}

// Start smooth transition to new position
void StartCameraTransition(const XPLMCameraPosition_t *targetPos, float transitionTime) {
    XPLMReadCameraPosition(&gTransition.startPos);
    gTransition.targetPos = *targetPos;
    gTransition.transitionTime = transitionTime;
    gTransition.currentTime = 0.0f;
    gTransition.isTransitioning = 1;
}
```

## Use Cases

### 1. External Aircraft Views

Create custom external views that follow the aircraft from specific angles:

```c
// Chase camera implementation
static int ChaseCamera(XPLMCameraPosition_t *outPos, int inIsLosingControl, void *inRefcon) {
    if (inIsLosingControl) return 1;
  
    // Get aircraft data
    float planeX = XPLMGetDataf(gPlaneXRef);
    float planeY = XPLMGetDataf(gPlaneYRef);
    float planeZ = XPLMGetDataf(gPlaneZRef);
    float planeHeading = XPLMGetDataf(gPlaneHeadingRef);
  
    // Position camera behind aircraft
    float chaseDistance = 100.0f;  // meters
    float chaseHeight = 20.0f;     // meters above aircraft
  
    float headingRad = planeHeading * M_PI / 180.0f;
    outPos->x = planeX - sin(headingRad) * chaseDistance;
    outPos->z = planeZ - cos(headingRad) * chaseDistance;
    outPos->y = planeY + chaseHeight;
  
    // Look at aircraft
    outPos->heading = planeHeading;
    outPos->pitch = -10.0f;  // Look slightly down
    outPos->roll = 0.0f;
    outPos->zoom = 1.0f;
  
    return 1;
}
```

### 2. Cinematic Camera System

Create scripted camera movements for presentations or recordings:

```c
// Keyframe-based cinematic camera
typedef struct {
    XPLMCameraPosition_t position;
    float time;
} CameraKeyframe_t;

static CameraKeyframe_t gKeyframes[] = {
    {{1000, 500, 2000, 0, 0, 0, 1.0f}, 0.0f},      // Start position
    {{1500, 600, 2500, -10, 45, 0, 1.5f}, 5.0f},   // Move and zoom
    {{2000, 400, 3000, 15, 90, 5, 1.0f}, 10.0f},   // Final position
};

static float gCinematicTime = 0.0f;
static int gCinematicActive = 0;

static int CinematicCamera(XPLMCameraPosition_t *outPos, int inIsLosingControl, void *inRefcon) {
    if (inIsLosingControl || !gCinematicActive) return 1;
  
    gCinematicTime += 0.016f;  // Increment time
  
    // Find appropriate keyframe pair
    int numKeyframes = sizeof(gKeyframes) / sizeof(gKeyframes[0]);
    for (int i = 0; i < numKeyframes - 1; i++) {
        if (gCinematicTime >= gKeyframes[i].time && gCinematicTime <= gKeyframes[i + 1].time) {
            // Interpolate between keyframes
            float t = (gCinematicTime - gKeyframes[i].time) / 
                     (gKeyframes[i + 1].time - gKeyframes[i].time);
          
            InterpolateCameraPosition(outPos, &gKeyframes[i].position, 
                                     &gKeyframes[i + 1].position, t);
            return 1;
        }
    }
  
    // End of cinematic
    gCinematicActive = 0;
    return 0;  // Surrender control
}
```

### 3. VR/AR Camera Integration

Interface with VR/AR systems for immersive experiences:

```c
// VR headset tracking integration
static int VRCamera(XPLMCameraPosition_t *outPos, int inIsLosingControl, void *inRefcon) {
    if (inIsLosingControl) return 1;
  
    // Get VR headset position and orientation
    // (This would interface with your VR SDK)
    VRPose_t vrPose = GetVRHeadsetPose();
  
    // Convert VR coordinates to X-Plane coordinates
    outPos->x = vrPose.position.x;
    outPos->y = vrPose.position.y;
    outPos->z = vrPose.position.z;
    outPos->pitch = vrPose.rotation.pitch;
    outPos->heading = vrPose.rotation.yaw;
    outPos->roll = vrPose.rotation.roll;
    outPos->zoom = 1.0f;
  
    return 1;
}
```

### 4. Recording and Replay System

Record camera movements and play them back:

```c
// Camera recording system
typedef struct {
    XPLMCameraPosition_t *positions;
    float *timestamps;
    int count;
    int capacity;
    int isRecording;
    int isReplaying;
    float startTime;
} CameraRecorder_t;

static CameraRecorder_t gRecorder = {0};

static int ReplayCamera(XPLMCameraPosition_t *outPos, int inIsLosingControl, void *inRefcon) {
    if (inIsLosingControl || !gRecorder.isReplaying) return 1;
  
    float currentTime = XPLMGetElapsedTime() - gRecorder.startTime;
  
    // Find appropriate frame
    for (int i = 0; i < gRecorder.count - 1; i++) {
        if (currentTime >= gRecorder.timestamps[i] && currentTime <= gRecorder.timestamps[i + 1]) {
            float t = (currentTime - gRecorder.timestamps[i]) / 
                     (gRecorder.timestamps[i + 1] - gRecorder.timestamps[i]);
          
            InterpolateCameraPosition(outPos, &gRecorder.positions[i], 
                                     &gRecorder.positions[i + 1], t);
            return 1;
        }
    }
  
    // End of replay
    gRecorder.isReplaying = 0;
    return 0;
}
```

## Performance Considerations

### 1. Callback Efficiency

Camera callbacks are called every frame (~60 FPS), so efficiency is critical:

```c
// BAD: Expensive operations in callback
static int SlowCamera(XPLMCameraPosition_t *outPos, int inIsLosingControl, void *inRefcon) {
    if (inIsLosingControl) return 1;
  
    // DON'T DO THIS - expensive file I/O in callback
    FILE *f = fopen("config.txt", "r");
    // ... read configuration each frame
  
    return 1;
}

// GOOD: Pre-compute expensive operations
static XPLMCameraPosition_t gPrecomputedPos;
static int gNeedsUpdate = 1;

static int FastCamera(XPLMCameraPosition_t *outPos, int inIsLosingControl, void *inRefcon) {
    if (inIsLosingControl) return 1;
  
    if (gNeedsUpdate) {
        // Update position only when needed
        ComputeNewCameraPosition(&gPrecomputedPos);
        gNeedsUpdate = 0;
    }
  
    *outPos = gPrecomputedPos;
    return 1;
}
```

### 2. Dataref Access Optimization

Cache dataref handles and minimize dataref reads:

```c
// Cache frequently used datarefs
static XPLMDataRef gPlaneXRef = NULL;
static XPLMDataRef gPlaneYRef = NULL;
static XPLMDataRef gPlaneZRef = NULL;

// Initialize once during plugin startup
void InitializeDatarefs() {
    gPlaneXRef = XPLMFindDataRef("sim/flightmodel/position/local_x");
    gPlaneYRef = XPLMFindDataRef("sim/flightmodel/position/local_y");
    gPlaneZRef = XPLMFindDataRef("sim/flightmodel/position/local_z");
}

// Read datarefs efficiently in callback
static int OptimizedCamera(XPLMCameraPosition_t *outPos, int inIsLosingControl, void *inRefcon) {
    if (inIsLosingControl) return 1;
  
    // Read multiple datarefs in one batch when possible
    float planePos[3];
    planePos[0] = XPLMGetDataf(gPlaneXRef);
    planePos[1] = XPLMGetDataf(gPlaneYRef);
    planePos[2] = XPLMGetDataf(gPlaneZRef);
  
    // Use cached values for calculations
    outPos->x = planePos[0] + 50.0f;
    outPos->y = planePos[1] + 10.0f;
    outPos->z = planePos[2];
  
    return 1;
}
```

## Best Practices

### 1. Control Duration Selection

Choose appropriate control duration based on use case:

```c
// For user-initiated views (temporary)
void ActivateCustomView() {
    XPLMControlCamera(xplm_ControlCameraUntilViewChanges, MyViewCallback, NULL);
}

// For automated systems (permanent until disabled)
void StartAutomaticCinematic() {
    XPLMControlCamera(xplm_ControlCameraForever, CinematicCallback, NULL);
}
```

### 2. Graceful Control Handling

Always handle loss of control gracefully:

```c
static int RobustCamera(XPLMCameraPosition_t *outPos, int inIsLosingControl, void *inRefcon) {
    if (inIsLosingControl) {
        // Clean up resources
        CleanupCameraResources();
      
        // Update plugin state
        gCameraControlActive = 0;
      
        // Log the event
        XPLMDebugString("Camera control lost\n");
      
        return 1;  // Acknowledge loss of control
    }
  
    // Normal camera positioning
    CalculateCameraPosition(outPos);
    return 1;
}
```

### 3. External View Preparation

Set appropriate view mode for external cameras:

```c
// Prepare for external camera views
void PrepareExternalView() {
    // Switch to external view first for correct behavior
    XPLMCommandOnce(XPLMFindCommand("sim/view/outside"));
  
    // Small delay to let view change take effect
    // Then start camera control
    ScheduleDelayedCameraActivation();
}
```

### 4. Coordinate System Awareness

Always use appropriate coordinate systems:

```c
// Convert between coordinate systems properly
static void ConvertWorldToLocal(double lat, double lon, double alt, 
                               float *outX, float *outY, float *outZ) {
    // Use XPLMGraphics functions for proper conversion
    XPLMWorldToLocal(lat, lon, alt, outX, outY, outZ);
}

static int GeographicCamera(XPLMCameraPosition_t *outPos, int inIsLosingControl, void *inRefcon) {
    if (inIsLosingControl) return 1;
  
    // Geographic coordinates for camera position
    double cameraLat = 37.7749;  // San Francisco
    double cameraLon = -122.4194;
    double cameraAlt = 1000.0;   // 1000 feet
  
    // Convert to local coordinates
    ConvertWorldToLocal(cameraLat, cameraLon, cameraAlt, 
                       &outPos->x, &outPos->y, &outPos->z);
  
    outPos->pitch = 0.0f;
    outPos->heading = 0.0f;
    outPos->roll = 0.0f;
    outPos->zoom = 1.0f;
  
    return 1;
}
```

## Common Patterns

### Camera Following Pattern

```c
// Generic camera following system
typedef struct {
    XPLMDataRef targetXRef;
    XPLMDataRef targetYRef;
    XPLMDataRef targetZRef;
    float offsetX, offsetY, offsetZ;
    float pitch, heading, roll;
} CameraFollowTarget_t;

static int FollowCamera(XPLMCameraPosition_t *outPos, int inIsLosingControl, void *inRefcon) {
    CameraFollowTarget_t *target = (CameraFollowTarget_t *)inRefcon;
  
    if (inIsLosingControl) return 1;
  
    // Get target position
    float targetX = XPLMGetDataf(target->targetXRef);
    float targetY = XPLMGetDataf(target->targetYRef);
    float targetZ = XPLMGetDataf(target->targetZRef);
  
    // Apply offset
    outPos->x = targetX + target->offsetX;
    outPos->y = targetY + target->offsetY;
    outPos->z = targetZ + target->offsetZ;
  
    // Apply orientation
    outPos->pitch = target->pitch;
    outPos->heading = target->heading;
    outPos->roll = target->roll;
    outPos->zoom = 1.0f;
  
    return 1;
}
```

### State Machine Camera

```c
// State-based camera system
typedef enum {
    CAMERA_STATE_IDLE,
    CAMERA_STATE_TAKEOFF,
    CAMERA_STATE_CRUISE,
    CAMERA_STATE_LANDING
} CameraState_t;

static CameraState_t gCameraState = CAMERA_STATE_IDLE;

static int StateMachineCamera(XPLMCameraPosition_t *outPos, int inIsLosingControl, void *inRefcon) {
    if (inIsLosingControl) return 1;
  
    switch (gCameraState) {
        case CAMERA_STATE_TAKEOFF:
            PositionTakeoffCamera(outPos);
            break;
        case CAMERA_STATE_CRUISE:
            PositionCruiseCamera(outPos);
            break;
        case CAMERA_STATE_LANDING:
            PositionLandingCamera(outPos);
            break;
        default:
            PositionDefaultCamera(outPos);
            break;
    }
  
    return 1;
}
```

## Error Handling

### Defensive Programming

```c
static int SafeCamera(XPLMCameraPosition_t *outPos, int inIsLosingControl, void *inRefcon) {
    // Always check for null pointers
    if (!outPos && !inIsLosingControl) {
        XPLMDebugString("ERROR: NULL camera position pointer\n");
        return 0;
    }
  
    if (inIsLosingControl) {
        // Safe cleanup
        SafeCleanup();
        return 1;
    }
  
    // Validate dataref handles
    if (!gPlaneXRef) {
        XPLMDebugString("ERROR: Invalid dataref handle\n");
        return 0;  // Surrender control
    }
  
    // Range checking
    float x = XPLMGetDataf(gPlaneXRef);
    if (isnan(x) || isinf(x)) {
        XPLMDebugString("ERROR: Invalid position data\n");
        return 0;
    }
  
    // Set safe default values
    outPos->x = x;
    outPos->y = 1000.0f;  // Safe altitude
    outPos->z = 0.0f;
    outPos->pitch = 0.0f;
    outPos->heading = 0.0f;
    outPos->roll = 0.0f;
    outPos->zoom = 1.0f;
  
    return 1;
}
```

## Advanced Features

### Multi-Plugin Camera Coordination

```c
// Coordinate with other plugins
static int CooperativeCamera(XPLMCameraPosition_t *outPos, int inIsLosingControl, void *inRefcon) {
    if (inIsLosingControl) return 1;
  
    // Check if higher priority plugin wants control
    if (CheckHigherPriorityRequest()) {
        return 0;  // Voluntarily surrender control
    }
  
    // Normal camera operation
    CalculatePosition(outPos);
    return 1;
}
```

### Dynamic Zoom Based on Speed

```c
static int DynamicZoomCamera(XPLMCameraPosition_t *outPos, int inIsLosingControl, void *inRefcon) {
    if (inIsLosingControl) return 1;
  
    // Get aircraft speed
    float speed = XPLMGetDataf(gGroundSpeedRef);  // meters/second
  
    // Calculate zoom based on speed (zoom out when faster)
    float baseZoom = 1.0f;
    float zoomFactor = 1.0f + (speed / 100.0f);  // Zoom out more at higher speeds
    outPos->zoom = baseZoom / zoomFactor;
  
    // Clamp zoom to reasonable range
    if (outPos->zoom < 0.1f) outPos->zoom = 0.1f;
    if (outPos->zoom > 5.0f) outPos->zoom = 5.0f;
  
    // Set other position parameters
    SetBasicCameraPosition(outPos);
  
    return 1;
}
```
