# XPLMInstance.h - X-Plane 12 SDK Instance Drawing API Documentation

## Overview

The XPLMInstance API provides a modern, high-performance system for drawing 3D objects in X-Plane through an instancing approach. Unlike traditional per-frame drawing callbacks, the instancing API allows plugins to register objects once and then manipulate them efficiently through dataref-driven position and state updates. This approach significantly improves performance by consolidating dataref operations and reducing main thread bottlenecks.

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Core Concepts](#core-concepts)
- [Data Types and Structures](#data-types-and-structures)
- [Instance Management](#instance-management)
- [Position and State Control](#position-and-state-control)
- [Dataref Integration](#dataref-integration)
- [Implementation Guidelines](#implementation-guidelines)
- [Use Cases](#use-cases)
- [Performance Considerations](#performance-considerations)
- [Best Practices](#best-practices)
- [Migration from Legacy Drawing](#migration-from-legacy-drawing)
- [Advanced Techniques](#advanced-techniques)

## Architecture Overview

The Instance API fundamentally changes how 3D object drawing is handled in X-Plane plugins:

### Traditional Drawing Model (Deprecated)

- Per-frame drawing callbacks
- Scattered dataref access throughout rendering
- Main thread blocks on each dataref operation
- Manual object management and cleanup

### Instance Drawing Model (Modern)

- Register objects once with associated datarefs
- Batch dataref operations at controlled times
- X-Plane manages rendering optimization
- Automatic resource management

### Key Benefits

**Performance Gains**:

- Consolidates all dataref access to specific times/threads
- Reduces main thread blocking
- Enables better multi-threading in X-Plane
- Minimizes OpenGL state changes

**Simplicity**:

- No per-frame drawing callbacks needed
- Automatic object lifetime management
- Consistent coordinate system handling

**Future-Proof**:

- Compatible with modern graphics APIs (Vulkan, Metal)
- Supported by X-Plane's evolving rendering pipeline
- Designed for multi-threaded execution

## Core Concepts

### Instance Lifecycle

1. **Object Loading**: Load .obj file using XPLMScenery API
2. **Instance Creation**: Create instance with associated datarefs
3. **Position Updates**: Set position and dataref values as needed
4. **Automatic Rendering**: X-Plane handles rendering based on instance state
5. **Cleanup**: Destroy instance when no longer needed

### Dataref-Driven Animation

The instance system is built around the concept that all object animation and state changes are driven by datarefs:

```c
// Traditional approach - scattered dataref access
void DrawCallback() {
    float engine1_rpm = XPLMGetDataf(engine1_rpm_ref);  // Main thread blocks
    float engine2_rpm = XPLMGetDataf(engine2_rpm_ref);  // Main thread blocks
    // ... draw object with engine animations
}

// Instance approach - batched dataref access
const char* datarefs[] = {
    "sim/aircraft/engine/acf_spnrpm[0]",
    "sim/aircraft/engine/acf_spnrpm[1]", 
    NULL
};
XPLMInstanceRef instance = XPLMCreateInstance(objRef, datarefs);

// Later, update all values at once
float values[] = {2400.0f, 2350.0f};  // RPM values
XPLMInstanceSetPosition(instance, &position, values);
```

### Object vs Instance Relationship

- **XPLMObjectRef**: Represents the loaded .obj file (geometry, textures)
- **XPLMInstanceRef**: Represents one instance of that object in the world
- Multiple instances can share the same object reference
- Each instance has its own position and dataref values

## Data Types and Structures

### XPLMInstanceRef

```c
typedef void * XPLMInstanceRef;
```

**Purpose**: Opaque handle representing a single instance of a 3D object.

**Characteristics**:

- Created via `XPLMCreateInstance()`
- Unique per instance (not shared)
- Remains valid until `XPLMDestroyInstance()` is called
- Used for all instance manipulation operations

### Instance Creation Requirements

The instance system has specific requirements for object and dataref setup:

**Object Requirements**:

- Object must be fully loaded before creating instance
- Cannot pass NULL object reference
- Object reference cannot be changed after instance creation

**Dataref Requirements**:

- Must provide valid pointer to null-terminated string array
- Can be empty array (single NULL element) if no datarefs needed
- Custom datarefs must be registered before object loading
- Array is copied during creation (no need to maintain after call)

## Instance Management

### XPLMCreateInstance

```c
XPLM_API XPLMInstanceRef XPLMCreateInstance(
    XPLMObjectRef    obj,       // Fully loaded object reference
    const char **    datarefs   // Null-terminated array of dataref names
);
```

**Purpose**: Creates a new instance of a 3D object with associated datarefs.

**Parameters**:

- `obj`: Must be a valid, fully-loaded object reference from XPLMScenery API
- `datarefs`: Null-terminated array of dataref name strings

**Return Value**: XPLMInstanceRef handle, or NULL on failure

**Example Usage**:

```c
// Load object first
XPLMObjectRef myObject = XPLMLoadObject("Resources/plugins/MyPlugin/objects/aircraft.obj");

// Define datarefs for animation
const char* instanceDatarefs[] = {
    "sim/aircraft/engine/acf_spnrpm[0]",           // Engine 1 RPM
    "sim/aircraft/engine/acf_spnrpm[1]",           // Engine 2 RPM  
    "sim/aircraft/parts/acf_gear_deploy",          // Gear position
    "sim/cockpit/electrical/beacon_lights_on",     // Beacon lights
    NULL  // Null terminator required
};

// Create instance
XPLMInstanceRef myInstance = XPLMCreateInstance(myObject, instanceDatarefs);
if (myInstance == NULL) {
    XPLMDebugString("Failed to create object instance\n");
    return;
}

// Can release object reference if not needed elsewhere
// Instance maintains its own reference
XPLMUnloadObject(myObject);
```

### XPLMDestroyInstance

```c
XPLM_API void XPLMDestroyInstance(XPLMInstanceRef instance);
```

**Purpose**: Destroys an instance and deallocates associated resources.

**Important Notes**:

- Instance handle becomes invalid after this call
- Object reference remains valid (must be released separately if needed)
- Instance automatically stops rendering

**Example Usage**:

```c
// Clean up instance when plugin disabled
void XPluginDisable(void) {
    if (gMyInstance) {
        XPLMDestroyInstance(gMyInstance);
        gMyInstance = NULL;
    }
}
```

## Position and State Control

### XPLMInstanceSetPosition

```c
XPLM_API void XPLMInstanceSetPosition(
    XPLMInstanceRef          instance,      // Instance to update
    const XPLMDrawInfo_t *   new_position,  // New position/orientation
    const float *            data           // Dataref values array
);
```

**Purpose**: Updates both the position/orientation and all dataref values for an instance.

**Parameters**:

- `instance`: Instance handle to update
- `new_position`: Pointer to XPLMDrawInfo_t structure with position data
- `data`: Array of float values corresponding to registered datarefs

**Critical Notes**:

- **Never call from drawing callbacks** - defeats the purpose of instancing
- Call from flight loop callbacks or UI callbacks instead
- Data array must contain one float per registered dataref
- Array order must match dataref registration order
- Before X-Plane 11.50: Must pass valid data pointer even with no datarefs

### XPLMDrawInfo_t Structure

The position information uses the standard XPLMDrawInfo_t structure from the Scenery API:

```c
typedef struct {
    float x;        // X position in meters, OpenGL coordinates
    float y;        // Y position in meters, OpenGL coordinates  
    float z;        // Z position in meters, OpenGL coordinates
    float pitch;    // Pitch in degrees (positive = nose up)
    float heading;  // Heading in degrees (positive = right turn)
    float roll;     // Roll in degrees (positive = right roll)
} XPLMDrawInfo_t;
```

**Coordinate System**:

- Uses OpenGL coordinate system
- Positions in meters from arbitrary origin
- Rotations in degrees
- Use XPLMGraphics functions for coordinate conversions

### Example Position Updates

```c
// Flight loop callback for updating instances
float MyFlightLoopCallback(float inElapsedSinceLastCall, 
                          float inElapsedTimeSinceLastFlightLoop,
                          int inCounter, void *inRefcon) {
  
    // Get aircraft position
    float planeX = XPLMGetDataf(gPlaneXRef);
    float planeY = XPLMGetDataf(gPlaneYRef);
    float planeZ = XPLMGetDataf(gPlaneZRef);
    float planeHeading = XPLMGetDataf(gPlaneHeadingRef);
  
    // Position instance relative to aircraft
    XPLMDrawInfo_t position;
    position.x = planeX + 50.0f;  // 50m to the right
    position.y = planeY;
    position.z = planeZ;
    position.pitch = 0.0f;
    position.heading = planeHeading;
    position.roll = 0.0f;
  
    // Update dataref values
    float datarefValues[] = {
        XPLMGetDataf(gEngine1RPMRef),   // Engine 1 RPM
        XPLMGetDataf(gEngine2RPMRef),   // Engine 2 RPM
        XPLMGetDataf(gGearPositionRef), // Gear position
        XPLMGetDataf(gBeaconLightRef)   // Beacon state
    };
  
    // Update instance (consolidates all dataref operations)
    XPLMInstanceSetPosition(gMyInstance, &position, datarefValues);
  
    return -1.0f; // Call next frame
}
```

## Dataref Integration

### Dataref Array Management

The dataref array passed to `XPLMCreateInstance()` defines the "schema" for all future position updates:

```c
// Define dataref schema
const char* aircraftDatarefs[] = {
    "sim/aircraft/engine/acf_spnrpm[0]",        // Index 0
    "sim/aircraft/engine/acf_spnrpm[1]",        // Index 1  
    "sim/aircraft/parts/acf_gear_deploy",       // Index 2
    "sim/cockpit/electrical/beacon_lights_on",  // Index 3
    "sim/cockpit/electrical/nav_lights_on",     // Index 4
    NULL
};

// Later, values array MUST match this order:
float values[] = {
    2400.0f,  // Index 0: Engine 1 RPM
    2350.0f,  // Index 1: Engine 2 RPM
    1.0f,     // Index 2: Gear down
    1.0f,     // Index 3: Beacon on
    1.0f      // Index 4: Nav lights on
};

XPLMInstanceSetPosition(instance, &pos, values);
```

### Custom Datarefs

When using custom datarefs in your object, they must be registered before the object is loaded:

```c
// BAD: Register dataref after object loading
XPLMObjectRef obj = XPLMLoadObject("my_object.obj");
XPLMRegisterDataAccessor("myplane/custom_value", ...); // Too late!

// GOOD: Register dataref before object loading
XPLMRegisterDataAccessor("myplane/custom_value", 
                        xplmType_Float, 1,
                        NULL, NULL,
                        GetCustomValue, SetCustomValue,
                        NULL, NULL, NULL, NULL, NULL, NULL,
                        &gCustomValue);

XPLMObjectRef obj = XPLMLoadObject("my_object.obj");

// Use in instance
const char* datarefs[] = {
    "myplane/custom_value",
    NULL
};
XPLMInstanceRef instance = XPLMCreateInstance(obj, datarefs);
```

### Empty Dataref Arrays

For objects that don't need dataref-driven animation:

```c
// Empty dataref array (still need valid pointer)
const char* noDatarefs[] = { NULL };

XPLMInstanceRef staticInstance = XPLMCreateInstance(obj, noDatarefs);

// Position updates with no dataref values
float emptyValues[] = {};  // Empty array, but valid pointer
XPLMInstanceSetPosition(staticInstance, &position, emptyValues);
```

**Note**: Before X-Plane 11.50, you must pass a valid (non-NULL) pointer for the data parameter even when no datarefs are registered.

## Implementation Guidelines

### Basic Instance Setup

```c
// Global variables
static XPLMObjectRef gMyObject = NULL;
static XPLMInstanceRef gMyInstance = NULL;
static XPLMDataRef gPlaneXRef, gPlaneYRef, gPlaneZRef;

// Initialize instance system
int InitializeInstances() {
    // Load object
    gMyObject = XPLMLoadObject("Resources/plugins/MyPlugin/objects/example.obj");
    if (!gMyObject) {
        XPLMDebugString("Failed to load object\n");
        return 0;
    }
  
    // Get aircraft position datarefs
    gPlaneXRef = XPLMFindDataRef("sim/flightmodel/position/local_x");
    gPlaneYRef = XPLMFindDataRef("sim/flightmodel/position/local_y");
    gPlaneZRef = XPLMFindDataRef("sim/flightmodel/position/local_z");
  
    if (!gPlaneXRef || !gPlaneYRef || !gPlaneZRef) {
        XPLMDebugString("Failed to find position datarefs\n");
        return 0;
    }
  
    // Create instance
    const char* datarefs[] = { NULL };  // No animation datarefs
    gMyInstance = XPLMCreateInstance(gMyObject, datarefs);
    if (!gMyInstance) {
        XPLMDebugString("Failed to create instance\n");
        return 0;
    }
  
    // Can release object reference
    XPLMUnloadObject(gMyObject);
    gMyObject = NULL;
  
    return 1;
}

// Update instance position
void UpdateInstancePosition() {
    if (!gMyInstance) return;
  
    // Get current aircraft position
    float x = XPLMGetDataf(gPlaneXRef);
    float y = XPLMGetDataf(gPlaneYRef);
    float z = XPLMGetDataf(gPlaneZRef);
  
    // Position object 100m in front of aircraft
    XPLMDrawInfo_t position;
    position.x = x;
    position.y = y + 5.0f;    // 5m above
    position.z = z + 100.0f;  // 100m ahead
    position.pitch = 0.0f;
    position.heading = 0.0f;
    position.roll = 0.0f;
  
    // Update instance (no dataref values needed)
    XPLMInstanceSetPosition(gMyInstance, &position, NULL);
}

// Cleanup
void CleanupInstances() {
    if (gMyInstance) {
        XPLMDestroyInstance(gMyInstance);
        gMyInstance = NULL;
    }
}
```

### Multi-Instance Management

```c
// Managing multiple instances
#define MAX_INSTANCES 10

typedef struct {
    XPLMInstanceRef instance;
    float offsetX, offsetY, offsetZ;
    float rotationSpeed;
    float currentRotation;
} InstanceData_t;

static XPLMObjectRef gSharedObject = NULL;
static InstanceData_t gInstances[MAX_INSTANCES];
static int gInstanceCount = 0;

// Create multiple instances of same object
void CreateInstanceFormation() {
    // Load shared object
    gSharedObject = XPLMLoadObject("Resources/plugins/MyPlugin/objects/formation_aircraft.obj");
    if (!gSharedObject) return;
  
    // Create instances in formation
    const char* datarefs[] = {
        "sim/aircraft/engine/acf_spnrpm[0]",
        NULL
    };
  
    for (int i = 0; i < 5; i++) {
        InstanceData_t *inst = &gInstances[gInstanceCount];
      
        inst->instance = XPLMCreateInstance(gSharedObject, datarefs);
        if (inst->instance) {
            // Set formation position offsets
            inst->offsetX = (i - 2) * 50.0f;  // Spread across 200m
            inst->offsetY = 0.0f;
            inst->offsetZ = i * -25.0f;        // Staggered depth
            inst->rotationSpeed = 1.0f + (i * 0.1f);  // Slightly different speeds
            inst->currentRotation = 0.0f;
          
            gInstanceCount++;
        }
    }
  
    // Release shared object reference
    XPLMUnloadObject(gSharedObject);
    gSharedObject = NULL;
}

// Update all instances
void UpdateFormation() {
    // Get leader aircraft position
    float leaderX = XPLMGetDataf(gPlaneXRef);
    float leaderY = XPLMGetDataf(gPlaneYRef);
    float leaderZ = XPLMGetDataf(gPlaneZRef);
    float leaderHeading = XPLMGetDataf(gPlaneHeadingRef);
  
    for (int i = 0; i < gInstanceCount; i++) {
        InstanceData_t *inst = &gInstances[i];
        if (!inst->instance) continue;
      
        // Update rotation
        inst->currentRotation += inst->rotationSpeed;
        if (inst->currentRotation >= 360.0f) {
            inst->currentRotation -= 360.0f;
        }
      
        // Calculate formation position
        XPLMDrawInfo_t position;
        position.x = leaderX + inst->offsetX;
        position.y = leaderY + inst->offsetY;
        position.z = leaderZ + inst->offsetZ;
        position.pitch = 0.0f;
        position.heading = leaderHeading + inst->currentRotation;
        position.roll = 0.0f;
      
        // Update dataref values (engine RPM)
        float values[] = { 2400.0f + (i * 50.0f) };  // Slightly different RPMs
      
        XPLMInstanceSetPosition(inst->instance, &position, values);
    }
}
```

### Complex Animation System

```c
// Advanced animation with multiple datarefs
typedef struct {
    XPLMInstanceRef instance;
  
    // Animation state
    float propellerRotation;
    float gearPosition;
    float lightState;
  
    // Animation parameters
    float propellerSpeed;
    float gearTransitionTime;
    int gearTarget;  // 0 = up, 1 = down
} AnimatedAircraft_t;

static AnimatedAircraft_t gAnimatedAircraft;

void CreateAnimatedAircraft() {
    // Load object with animation datarefs
    XPLMObjectRef obj = XPLMLoadObject("Resources/plugins/MyPlugin/objects/animated_plane.obj");
    if (!obj) return;
  
    // Define animation datarefs
    const char* datarefs[] = {
        "sim/aircraft/prop/prop_rotation_angle_deg",  // Propeller rotation
        "sim/aircraft/parts/acf_gear_deploy",         // Gear position  
        "sim/cockpit/electrical/beacon_lights_on",    // Beacon light
        "sim/cockpit/electrical/nav_lights_on",       // Nav lights
        "sim/aircraft/engine/acf_spnrpm[0]",         // Engine RPM
        NULL
    };
  
    gAnimatedAircraft.instance = XPLMCreateInstance(obj, datarefs);
    XPLMUnloadObject(obj);
  
    // Initialize animation state
    gAnimatedAircraft.propellerRotation = 0.0f;
    gAnimatedAircraft.gearPosition = 0.0f;  // Start with gear up
    gAnimatedAircraft.lightState = 0.0f;
    gAnimatedAircraft.propellerSpeed = 720.0f;  // degrees per second
    gAnimatedAircraft.gearTransitionTime = 3.0f;  // 3 seconds to extend/retract
    gAnimatedAircraft.gearTarget = 0;
}

void UpdateAnimatedAircraft(float deltaTime) {
    if (!gAnimatedAircraft.instance) return;
  
    // Update propeller rotation
    gAnimatedAircraft.propellerRotation += gAnimatedAircraft.propellerSpeed * deltaTime;
    while (gAnimatedAircraft.propellerRotation >= 360.0f) {
        gAnimatedAircraft.propellerRotation -= 360.0f;
    }
  
    // Update gear position (smooth transition)
    float targetGear = (float)gAnimatedAircraft.gearTarget;
    float gearSpeed = 1.0f / gAnimatedAircraft.gearTransitionTime;
  
    if (gAnimatedAircraft.gearPosition < targetGear) {
        gAnimatedAircraft.gearPosition += gearSpeed * deltaTime;
        if (gAnimatedAircraft.gearPosition > targetGear) {
            gAnimatedAircraft.gearPosition = targetGear;
        }
    } else if (gAnimatedAircraft.gearPosition > targetGear) {
        gAnimatedAircraft.gearPosition -= gearSpeed * deltaTime;
        if (gAnimatedAircraft.gearPosition < targetGear) {
            gAnimatedAircraft.gearPosition = targetGear;
        }
    }
  
    // Update light state (blinking beacon)
    static float blinkTime = 0.0f;
    blinkTime += deltaTime;
    gAnimatedAircraft.lightState = (fmod(blinkTime, 1.0f) < 0.5f) ? 1.0f : 0.0f;
  
    // Set position
    XPLMDrawInfo_t position;
    GetAircraftPosition(&position);  // Your position calculation
  
    // Update all animation datarefs at once
    float values[] = {
        gAnimatedAircraft.propellerRotation,  // Propeller angle
        gAnimatedAircraft.gearPosition,       // Gear position (0-1)
        gAnimatedAircraft.lightState,         // Beacon on/off
        1.0f,                                 // Nav lights always on
        2400.0f                               // Engine RPM
    };
  
    XPLMInstanceSetPosition(gAnimatedAircraft.instance, &position, values);
}

// Control gear from command
void ToggleGear() {
    gAnimatedAircraft.gearTarget = 1 - gAnimatedAircraft.gearTarget;
}
```

## Use Cases

### 1. AI Traffic System

```c
// AI aircraft instance management
#define MAX_AI_AIRCRAFT 20

typedef struct {
    XPLMInstanceRef instance;
    float x, y, z;
    float heading, pitch, roll;
    float speed;
    float targetX, targetZ;  // Navigation target
    int isActive;
} AIAircraft_t;

static AIAircraft_t gAIFleet[MAX_AI_AIRCRAFT];
static XPLMObjectRef gAIObject = NULL;

void InitializeAISystem() {
    // Load shared AI aircraft object
    gAIObject = XPLMLoadObject("Resources/plugins/MyPlugin/objects/ai_cessna.obj");
    if (!gAIObject) return;
  
    // Create AI aircraft instances
    const char* datarefs[] = {
        "sim/aircraft/engine/acf_spnrpm[0]",
        "sim/aircraft/parts/acf_gear_deploy",
        NULL
    };
  
    for (int i = 0; i < MAX_AI_AIRCRAFT; i++) {
        gAIFleet[i].instance = XPLMCreateInstance(gAIObject, datarefs);
        gAIFleet[i].isActive = 0;
    }
  
    XPLMUnloadObject(gAIObject);
    gAIObject = NULL;
}

void SpawnAIAircraft(float x, float y, float z) {
    // Find available slot
    for (int i = 0; i < MAX_AI_AIRCRAFT; i++) {
        if (!gAIFleet[i].isActive) {
            gAIFleet[i].x = x;
            gAIFleet[i].y = y;
            gAIFleet[i].z = z;
            gAIFleet[i].heading = 0.0f;
            gAIFleet[i].pitch = 0.0f;
            gAIFleet[i].roll = 0.0f;
            gAIFleet[i].speed = 50.0f;  // m/s
            gAIFleet[i].isActive = 1;
          
            // Set initial target
            gAIFleet[i].targetX = x + 1000.0f;
            gAIFleet[i].targetZ = z;
            break;
        }
    }
}

void UpdateAISystem(float deltaTime) {
    for (int i = 0; i < MAX_AI_AIRCRAFT; i++) {
        AIAircraft_t *ai = &gAIFleet[i];
        if (!ai->isActive || !ai->instance) continue;
      
        // Simple navigation toward target
        float dx = ai->targetX - ai->x;
        float dz = ai->targetZ - ai->z;
        float distance = sqrt(dx*dx + dz*dz);
      
        if (distance > 10.0f) {
            // Move toward target
            ai->x += (dx / distance) * ai->speed * deltaTime;
            ai->z += (dz / distance) * ai->speed * deltaTime;
            ai->heading = atan2(dx, dz) * 180.0f / M_PI;
        } else {
            // Reached target, set new one
            ai->targetX = ai->x + (rand() % 2000) - 1000.0f;
            ai->targetZ = ai->z + (rand() % 2000) - 1000.0f;
        }
      
        // Update instance
        XPLMDrawInfo_t position;
        position.x = ai->x;
        position.y = ai->y;
        position.z = ai->z;
        position.pitch = ai->pitch;
        position.heading = ai->heading;
        position.roll = ai->roll;
      
        float values[] = {
            2200.0f,  // Engine RPM
            0.0f      // Gear up
        };
      
        XPLMInstanceSetPosition(ai->instance, &position, values);
    }
}
```

### 2. Ground Equipment System

```c
// Airport ground equipment
typedef struct {
    XPLMInstanceRef instance;
    float homeX, homeY, homeZ;  // Parking position
    float currentX, currentY, currentZ;
    int isMoving;
    float moveSpeed;
} GroundEquipment_t;

static GroundEquipment_t gBaggageCarts[10];
static GroundEquipment_t gFuelTrucks[5];

void CreateGroundEquipment() {
    // Load ground equipment objects
    XPLMObjectRef cartObj = XPLMLoadObject("Resources/plugins/MyPlugin/objects/baggage_cart.obj");
    XPLMObjectRef truckObj = XPLMLoadObject("Resources/plugins/MyPlugin/objects/fuel_truck.obj");
  
    const char* cartDatarefs[] = { NULL };  // Static objects
    const char* truckDatarefs[] = {
        "sim/aircraft/engine/acf_spnrpm[0]",  // Truck engine
        NULL
    };
  
    // Create baggage carts
    for (int i = 0; i < 10; i++) {
        gBaggageCarts[i].instance = XPLMCreateInstance(cartObj, cartDatarefs);
        // Position around terminal
        gBaggageCarts[i].homeX = 1000.0f + (i * 20.0f);
        gBaggageCarts[i].homeY = 100.0f;
        gBaggageCarts[i].homeZ = 2000.0f;
        gBaggageCarts[i].currentX = gBaggageCarts[i].homeX;
        gBaggageCarts[i].currentY = gBaggageCarts[i].homeY;
        gBaggageCarts[i].currentZ = gBaggageCarts[i].homeZ;
    }
  
    // Create fuel trucks
    for (int i = 0; i < 5; i++) {
        gFuelTrucks[i].instance = XPLMCreateInstance(truckObj, truckDatarefs);
        gFuelTrucks[i].homeX = 800.0f + (i * 30.0f);
        gFuelTrucks[i].homeY = 100.0f;
        gFuelTrucks[i].homeZ = 1800.0f;
        gFuelTrucks[i].moveSpeed = 10.0f;  // m/s
    }
  
    XPLMUnloadObject(cartObj);
    XPLMUnloadObject(truckObj);
}
```

### 3. Dynamic Weather Effects

```c
// Weather-responsive objects (wind socks, flags, etc.)
typedef struct {
    XPLMInstanceRef instance;
    float baseX, baseY, baseZ;
    float windResponse;  // How much object responds to wind
    XPLMDataRef windSpeedRef;
    XPLMDataRef windDirectionRef;
} WeatherObject_t;

static WeatherObject_t gWindSocks[5];

void CreateWeatherObjects() {
    XPLMObjectRef windsockObj = XPLMLoadObject("Resources/plugins/MyPlugin/objects/windsock.obj");
  
    const char* datarefs[] = {
        "sim/weather/wind_speed_kt[0]",     // Wind speed
        "sim/weather/wind_direction_degt[0]", // Wind direction
        NULL
    };
  
    for (int i = 0; i < 5; i++) {
        gWindSocks[i].instance = XPLMCreateInstance(windsockObj, datarefs);
        gWindSocks[i].baseX = 500.0f + (i * 200.0f);
        gWindSocks[i].baseY = 100.0f;
        gWindSocks[i].baseZ = 1500.0f;
        gWindSocks[i].windResponse = 1.0f;
        gWindSocks[i].windSpeedRef = XPLMFindDataRef("sim/weather/wind_speed_kt[0]");
        gWindSocks[i].windDirectionRef = XPLMFindDataRef("sim/weather/wind_direction_degt[0]");
    }
  
    XPLMUnloadObject(windsockObj);
}

void UpdateWeatherObjects() {
    for (int i = 0; i < 5; i++) {
        WeatherObject_t *ws = &gWindSocks[i];
        if (!ws->instance) continue;
      
        float windSpeed = XPLMGetDataf(ws->windSpeedRef);
        float windDirection = XPLMGetDataf(ws->windDirectionRef);
      
        // Calculate windsock deflection
        float deflection = (windSpeed / 30.0f) * 45.0f;  // Max 45° deflection at 30 knots
        if (deflection > 45.0f) deflection = 45.0f;
      
        XPLMDrawInfo_t position;
        position.x = ws->baseX;
        position.y = ws->baseY;
        position.z = ws->baseZ;
        position.pitch = -deflection;  // Negative pitch = upward deflection
        position.heading = windDirection;
        position.roll = 0.0f;
      
        float values[] = { windSpeed, windDirection };
        XPLMInstanceSetPosition(ws->instance, &position, values);
    }
}
```

## Performance Considerations

### Batch Updates

```c
// GOOD: Update all instances in one pass
void UpdateAllInstances() {
    // Read all datarefs once
    float windSpeed = XPLMGetDataf(gWindSpeedRef);
    float windDir = XPLMGetDataf(gWindDirRef);
    float planeX = XPLMGetDataf(gPlaneXRef);
    float planeY = XPLMGetDataf(gPlaneYRef);
    float planeZ = XPLMGetDataf(gPlaneZRef);
  
    // Update all instances with cached values
    for (int i = 0; i < gInstanceCount; i++) {
        UpdateSingleInstance(i, windSpeed, windDir, planeX, planeY, planeZ);
    }
}

// BAD: Scattered dataref reads
void UpdateInstancesBadly() {
    for (int i = 0; i < gInstanceCount; i++) {
        float windSpeed = XPLMGetDataf(gWindSpeedRef);  // Repeated reads
        float windDir = XPLMGetDataf(gWindDirRef);      // Repeated reads
        UpdateSingleInstance(i, windSpeed, windDir);
    }
}
```

### Selective Updates

```c
// Only update when necessary
typedef struct {
    XPLMInstanceRef instance;
    float lastUpdateTime;
    float updateInterval;
    int needsUpdate;
} ManagedInstance_t;

void UpdateManagedInstances() {
    float currentTime = XPLMGetElapsedTime();
  
    for (int i = 0; i < gManagedCount; i++) {
        ManagedInstance_t *mi = &gManagedInstances[i];
      
        // Check if update needed
        if (!mi->needsUpdate && 
            (currentTime - mi->lastUpdateTime) < mi->updateInterval) {
            continue;
        }
      
        // Update this instance
        UpdateSingleManagedInstance(mi);
        mi->lastUpdateTime = currentTime;
        mi->needsUpdate = 0;
    }
}

// Flag instances for update when state changes
void OnAircraftStateChange() {
    for (int i = 0; i < gManagedCount; i++) {
        gManagedInstances[i].needsUpdate = 1;
    }
}
```

## Best Practices

### Resource Management

```c
// Proper instance lifecycle management
typedef struct {
    XPLMObjectRef sharedObject;
    XPLMInstanceRef *instances;
    int instanceCount;
    int maxInstances;
} InstanceManager_t;

InstanceManager_t* CreateInstanceManager(const char* objectPath, 
                                        const char** datarefs, 
                                        int maxCount) {
    InstanceManager_t *mgr = malloc(sizeof(InstanceManager_t));
    if (!mgr) return NULL;
  
    // Load shared object
    mgr->sharedObject = XPLMLoadObject(objectPath);
    if (!mgr->sharedObject) {
        free(mgr);
        return NULL;
    }
  
    // Allocate instance array
    mgr->instances = calloc(maxCount, sizeof(XPLMInstanceRef));
    if (!mgr->instances) {
        XPLMUnloadObject(mgr->sharedObject);
        free(mgr);
        return NULL;
    }
  
    mgr->instanceCount = 0;
    mgr->maxInstances = maxCount;
  
    return mgr;
}

XPLMInstanceRef AddInstance(InstanceManager_t *mgr, const char** datarefs) {
    if (!mgr || mgr->instanceCount >= mgr->maxInstances) {
        return NULL;
    }
  
    XPLMInstanceRef instance = XPLMCreateInstance(mgr->sharedObject, datarefs);
    if (instance) {
        mgr->instances[mgr->instanceCount++] = instance;
    }
  
    return instance;
}

void DestroyInstanceManager(InstanceManager_t *mgr) {
    if (!mgr) return;
  
    // Destroy all instances
    for (int i = 0; i < mgr->instanceCount; i++) {
        if (mgr->instances[i]) {
            XPLMDestroyInstance(mgr->instances[i]);
        }
    }
  
    // Clean up resources
    free(mgr->instances);
    if (mgr->sharedObject) {
        XPLMUnloadObject(mgr->sharedObject);
    }
    free(mgr);
}
```

### Error Handling

```c
// Robust instance creation
XPLMInstanceRef CreateRobustInstance(const char* objectPath, const char** datarefs) {
    // Validate inputs
    if (!objectPath || !datarefs) {
        XPLMDebugString("ERROR: Invalid parameters to CreateRobustInstance\n");
        return NULL;
    }
  
    // Load object with error checking
    XPLMObjectRef obj = XPLMLoadObject(objectPath);
    if (!obj) {
        char msg[512];
        sprintf(msg, "ERROR: Failed to load object: %s\n", objectPath);
        XPLMDebugString(msg);
        return NULL;
    }
  
    // Validate datarefs exist
    for (int i = 0; datarefs[i] != NULL; i++) {
        XPLMDataRef testRef = XPLMFindDataRef(datarefs[i]);
        if (!testRef) {
            char msg[512];
            sprintf(msg, "WARNING: Dataref not found: %s\n", datarefs[i]);
            XPLMDebugString(msg);
        }
    }
  
    // Create instance
    XPLMInstanceRef instance = XPLMCreateInstance(obj, datarefs);
    if (!instance) {
        XPLMDebugString("ERROR: Failed to create instance\n");
        XPLMUnloadObject(obj);
        return NULL;
    }
  
    // Success - release object reference
    XPLMUnloadObject(obj);
    return instance;
}
```

## Migration from Legacy Drawing

### Before (Legacy Drawing Callbacks)

```c
// OLD: Drawing callback approach
static XPLMObjectRef gMyObject = NULL;

int MyDrawCallback(XPLMDrawingPhase inPhase, int inIsBefore, void *inRefcon) {
    if (inPhase != xplm_Phase_Objects || inIsBefore) return 1;
  
    // Expensive per-frame operations
    float rpm = XPLMGetDataf(XPLMFindDataRef("sim/aircraft/engine/acf_spnrpm[0]"));
    float x = XPLMGetDataf(XPLMFindDataRef("sim/flightmodel/position/local_x"));
    float y = XPLMGetDataf(XPLMFindDataRef("sim/flightmodel/position/local_y"));
    float z = XPLMGetDataf(XPLMFindDataRef("sim/flightmodel/position/local_z"));
  
    // Manual OpenGL state management
    glPushMatrix();
    glTranslatef(x + 50.0f, y, z);
  
    // Draw object manually
    if (gMyObject) {
        XPLMDrawObject(gMyObject);
    }
  
    glPopMatrix();
    return 1;
}

// Register callback
XPLMRegisterDrawCallback(MyDrawCallback, xplm_Phase_Objects, 0, NULL);
```

### After (Instance API)

```c
// NEW: Instance approach
static XPLMInstanceRef gMyInstance = NULL;
static XPLMDataRef gRPMRef = NULL, gPosXRef = NULL, gPosYRef = NULL, gPosZRef = NULL;

void InitializeInstance() {
    // Cache datarefs once
    gRPMRef = XPLMFindDataRef("sim/aircraft/engine/acf_spnrpm[0]");
    gPosXRef = XPLMFindDataRef("sim/flightmodel/position/local_x");
    gPosYRef = XPLMFindDataRef("sim/flightmodel/position/local_y");  
    gPosZRef = XPLMFindDataRef("sim/flightmodel/position/local_z");
  
    // Load object and create instance
    XPLMObjectRef obj = XPLMLoadObject("Resources/plugins/MyPlugin/objects/my_object.obj");
  
    const char* datarefs[] = {
        "sim/aircraft/engine/acf_spnrpm[0]",
        NULL
    };
  
    gMyInstance = XPLMCreateInstance(obj, datarefs);
    XPLMUnloadObject(obj);  // Instance holds reference
}

// Flight loop callback (called much less frequently)
float UpdateInstance(float elapsed, float elapsedSim, int counter, void *refcon) {
    if (!gMyInstance) return -1.0f;
  
    // Read datarefs once per update
    float rpm = XPLMGetDataf(gRPMRef);
    float x = XPLMGetDataf(gPosXRef);
    float y = XPLMGetDataf(gPosYRef);
    float z = XPLMGetDataf(gPosZRef);
  
    // Set position and state
    XPLMDrawInfo_t position;
    position.x = x + 50.0f;
    position.y = y;
    position.z = z;
    position.pitch = 0.0f;
    position.heading = 0.0f;
    position.roll = 0.0f;
  
    float values[] = { rpm };
    XPLMInstanceSetPosition(gMyInstance, &position, values);
  
    return 0.1f;  // Update every 0.1 seconds instead of every frame
}

// Register flight loop instead of draw callback
XPLMRegisterFlightLoopCallback(UpdateInstance, 0.1f, NULL);
```

## Advanced Techniques

### Dynamic Instance Creation

```c
// Create instances based on runtime conditions
void ManageAirportTraffic() {
    // Get current airport
    char currentAirport[8];
    XPLMGetDatab(XPLMFindDataRef("sim/airport/current_icao"), currentAirport, 0, 7);
  
    // Check if we need to spawn traffic for this airport
    if (strcmp(currentAirport, gLastAirport) != 0) {
        // Destroy old traffic
        DestroyAllTrafficInstances();
      
        // Create new traffic for current airport
        CreateTrafficForAirport(currentAirport);
      
        strcpy(gLastAirport, currentAirport);
    }
}

void CreateTrafficForAirport(const char* icao) {
    // Load airport-specific configuration
    TrafficConfig_t *config = LoadTrafficConfig(icao);
    if (!config) return;
  
    // Create appropriate number of instances for this airport size
    int trafficCount = CalculateTrafficDensity(config->airportSize);
  
    for (int i = 0; i < trafficCount; i++) {
        // Select random aircraft type
        const char* objectPath = SelectRandomAircraftType(config->aircraftTypes);
      
        // Create instance with appropriate datarefs
        const char* datarefs[] = {
            "sim/aircraft/engine/acf_spnrpm[0]",
            "sim/aircraft/parts/acf_gear_deploy",
            NULL
        };
      
        XPLMObjectRef obj = XPLMLoadObject(objectPath);
        if (obj) {
            XPLMInstanceRef instance = XPLMCreateInstance(obj, datarefs);
            XPLMUnloadObject(obj);
          
            if (instance) {
                AddTrafficInstance(instance, config);
            }
        }
    }
}
```

### Performance Monitoring

```c
// Monitor instance performance
typedef struct {
    int instanceCount;
    float lastUpdateTime;
    float totalUpdateTime;
    int updateCount;
    float averageUpdateTime;
} InstancePerformanceMetrics_t;

static InstancePerformanceMetrics_t gMetrics = {0};

void UpdateInstancesWithProfiling() {
    float startTime = XPLMGetElapsedTime();
  
    // Update all instances
    UpdateAllInstances();
  
    float endTime = XPLMGetElapsedTime();
    float updateTime = endTime - startTime;
  
    // Update metrics
    gMetrics.totalUpdateTime += updateTime;
    gMetrics.updateCount++;
    gMetrics.lastUpdateTime = updateTime;
    gMetrics.averageUpdateTime = gMetrics.totalUpdateTime / gMetrics.updateCount;
  
    // Log performance warnings
    if (updateTime > 0.016f) {  // More than one frame time at 60fps
        char msg[256];
        sprintf(msg, "WARNING: Instance update took %.3fms (%.1ffps equivalent)\n", 
                updateTime * 1000.0f, 1.0f / updateTime);
        XPLMDebugString(msg);
    }
}

void LogPerformanceMetrics() {
    char msg[512];
    sprintf(msg, "Instance Performance: Count=%d, Last=%.3fms, Avg=%.3fms\n",
            gMetrics.instanceCount, 
            gMetrics.lastUpdateTime * 1000.0f,
            gMetrics.averageUpdateTime * 1000.0f);
    XPLMDebugString(msg);
}
```
