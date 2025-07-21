# XPLMPlanes API Documentation

## Overview

The XPLMPlanes API provides comprehensive control over aircraft in X-Plane, including the user's aircraft and AI/multiplayer aircraft. This system enables plugins to change aircraft, position aircraft, and manage the multiplayer aircraft system with exclusive access control.

## Key Features

- **User Aircraft Control**: Load different aircraft and position them at airports or specific coordinates
- **Multiplayer Aircraft Management**: Control AI aircraft with exclusive access system
- **Aircraft Information Access**: Query loaded aircraft models and configurations
- **Position Management**: Place aircraft at airports or specific lat/lon coordinates
- **AI Control**: Disable X-Plane's AI for custom aircraft control

## Architecture

### Aircraft Indexing

Aircraft are indexed starting from 0:

- **Index 0**: Always the user's aircraft (XPLM_USER_AIRCRAFT)
- **Index 1+**: AI/multiplayer aircraft

### Exclusive Access System

The multiplayer aircraft system uses exclusive access - only one plugin can control AI aircraft at a time. This prevents conflicts between multiple plugins attempting to manage the same aircraft.

### Aircraft File Paths

**Important Note**: Unlike other X-Plane APIs, aircraft paths must be complete file system paths including drive letters on Windows and full directory paths on all platforms.

## Constants

### XPLM_USER_AIRCRAFT

```c
#define XPLM_USER_AIRCRAFT 0
```

Always refers to the user's aircraft (index 0).

## User Aircraft Functions

### XPLMSetUsersAircraft

```c
XPLM_API void XPLMSetUsersAircraft(const char * inAircraftPath);
```

Changes the user's aircraft to a different model.

**Parameters:**

- `inAircraftPath`: Complete file system path to .acf file

**Behavior:**

- Automatically repositions user at nearest airport's first runway
- Reloads flight model and aircraft systems
- Resets aircraft state (fuel, payload, etc.)

**Usage:**

```c
// Windows path example
XPLMSetUsersAircraft("C:\\X-Plane 12\\Aircraft\\Laminar Research\\Cessna 172SP\\Cessna_172SP.acf");

// macOS path example  
XPLMSetUsersAircraft("/Applications/X-Plane 12/Aircraft/Laminar Research/Cessna 172SP/Cessna_172SP.acf");

// Linux path example
XPLMSetUsersAircraft("/home/user/X-Plane 12/Aircraft/Laminar Research/Cessna 172SP/Cessna_172SP.acf");
```

**Implementation Guidelines:**

```c
void LoadAircraft(const char* aircraftName) {
    // Build complete path
    char systemPath[512];
    XPLMGetSystemPath(systemPath);
  
    char aircraftPath[1024];
    snprintf(aircraftPath, sizeof(aircraftPath), 
             "%sAircraft%sLaminar Research%s%s%s%s.acf",
             systemPath, XPLMGetDirectorySeparator(),
             XPLMGetDirectorySeparator(), aircraftName,
             XPLMGetDirectorySeparator(), aircraftName);
  
    XPLMSetUsersAircraft(aircraftPath);
  
    XPLMDebugString("Loaded aircraft: ");
    XPLMDebugString(aircraftPath);
    XPLMDebugString("\n");
}
```

### XPLMPlaceUserAtAirport

```c
XPLM_API void XPLMPlaceUserAtAirport(const char * inAirportCode);
```

Positions the user's aircraft at a specific airport.

**Parameters:**

- `inAirportCode`: ICAO airport identifier (e.g., "KBOS", "EGLL")

**Behavior:**

- Places aircraft on first available runway
- Sets appropriate heading for runway
- Preserves current aircraft and configuration
- Loads necessary scenery if not already loaded

**Examples:**

```c
// Place at Boston Logan
XPLMPlaceUserAtAirport("KBOS");

// Place at London Heathrow  
XPLMPlaceUserAtAirport("EGLL");

// Place at custom airport
XPLMPlaceUserAtAirport("XPSP");  // Custom scenery airport
```

### XPLMPlaceUserAtLocation (XPLM300+)

```c
XPLM_API void XPLMPlaceUserAtLocation(
    double latitudeDegrees,      // Latitude in degrees
    double longitudeDegrees,     // Longitude in degrees  
    float  elevationMetersMSL,   // Elevation in meters MSL
    float  headingDegreesTrue,   // True heading in degrees
    float  speedMetersPerSecond  // Initial speed in m/s
);
```

Places the user's aircraft at a specific geographic location.

**Parameters:**

- `latitudeDegrees`: WGS84 latitude (-90 to +90)
- `longitudeDegrees`: WGS84 longitude (-180 to +180)
- `elevationMetersMSL`: Height above mean sea level in meters
- `headingDegreesTrue`: True heading (0-359 degrees, 0=North)
- `speedMetersPerSecond`: Initial airspeed in meters per second

**Important Notes:**

- Engines always start running (ignores user preferences)
- Automatically loads necessary scenery
- Can place aircraft in-air or on ground
- Use ground elevation + aircraft height for ground placement

**Usage Examples:**

```c
// Place aircraft over New York City at 3000 feet
XPLMPlaceUserAtLocation(
    40.7128,        // NYC latitude
    -74.0060,       // NYC longitude  
    914.4,          // 3000 feet in meters
    90.0,           // Heading east
    77.17           // 150 knots in m/s
);

// Place on ground (runway threshold)
float groundElevation = GetGroundElevation(lat, lon);
XPLMPlaceUserAtLocation(
    lat, lon,
    groundElevation + 3.0,  // Aircraft height above ground
    270.0,                  // Heading west
    0.0                     // Stationary
);
```

## Global Aircraft Access

### XPLMCountAircraft

```c
XPLM_API void XPLMCountAircraft(
    int *        outTotalAircraft,    // Maximum aircraft capacity
    int *        outActiveAircraft,   // Currently active aircraft  
    XPLMPluginID * outController      // Plugin controlling AI aircraft
);
```

Retrieves aircraft system status information.

**Parameters:**

- `outTotalAircraft`: Maximum number of aircraft X-Plane can handle
- `outActiveAircraft`: Number of aircraft currently active (includes user)
- `outController`: Plugin ID controlling AI aircraft (or XPLM_NO_PLUGIN_ID)

**Usage:**

```c
void QueryAircraftStatus() {
    int totalAircraft, activeAircraft;
    XPLMPluginID controller;
  
    XPLMCountAircraft(&totalAircraft, &activeAircraft, &controller);
  
    XPLMDebugString("Aircraft Status:\n");
    printf("  Total capacity: %d\n", totalAircraft);
    printf("  Active aircraft: %d\n", activeAircraft);
  
    if (controller == XPLM_NO_PLUGIN_ID) {
        XPLMDebugString("  No plugin controlling AI\n");
    } else if (controller == XPLMGetMyID()) {
        XPLMDebugString("  We control AI aircraft\n");
    } else {
        XPLMDebugString("  Another plugin controls AI\n");
    }
}
```

### XPLMGetNthAircraftModel

```c
XPLM_API void XPLMGetNthAircraftModel(
    int    inIndex,        // Aircraft index (0-based)
    char * outFileName,    // Aircraft file name (256+ chars)
    char * outPath         // Complete file path (512+ chars)
);
```

Retrieves the model information for a specific aircraft.

**Parameters:**

- `inIndex`: Aircraft index (0 = user, 1+ = AI aircraft)
- `outFileName`: Buffer for aircraft file name (recommend 256 chars)
- `outPath`: Buffer for complete file path (recommend 512 chars)

**Buffer Requirements:**

- `outFileName`: At least 256 characters
- `outPath`: At least 512 characters

**Example:**

```c
void ListAllAircraft() {
    int totalAircraft, activeAircraft;
    XPLMPluginID controller;
  
    XPLMCountAircraft(&totalAircraft, &activeAircraft, &controller);
  
    for (int i = 0; i < activeAircraft; i++) {
        char fileName[256];
        char filePath[512];
      
        XPLMGetNthAircraftModel(i, fileName, filePath);
      
        printf("Aircraft %d: %s\n", i, fileName);
        printf("  Path: %s\n", filePath);
      
        if (i == 0) {
            printf("  (User Aircraft)\n");
        }
    }
}
```

## Exclusive Aircraft Access

### Access Control System

The exclusive access system ensures only one plugin controls AI aircraft at a time, preventing conflicts between traffic generators, online networks, and AI management plugins.

### XPLMPlanesAvailable_f

```c
typedef void (* XPLMPlanesAvailable_f)(void * inRefcon);
```

Callback function type called when aircraft access becomes available.

### XPLMAcquirePlanes

```c
XPLM_API int XPLMAcquirePlanes(
    char **               inAircraft,    // Array of aircraft paths (NULL-terminated)
    XPLMPlanesAvailable_f inCallback,    // Callback for when access available
    void *                inRefcon       // Callback reference data
);
```

Requests exclusive access to control AI aircraft.

**Parameters:**

- `inAircraft`: Array of aircraft path strings, NULL-terminated (can be NULL)
- `inCallback`: Called when access becomes available (if not granted immediately)
- `inRefcon`: User data passed to callback

**Returns:**

- `1`: Access granted immediately
- `0`: Access denied, callback will be called when available

**Aircraft Array Format:**

- Each element is a complete path to .acf file
- Empty string ("") for aircraft slots you don't want loaded
- NULL pointer terminates the array
- Pass NULL if you don't want to specify aircraft

**Usage Patterns:**

```c
// Global callback function
void PlanesAvailableCallback(void* refcon) {
    XPLMDebugString("Aircraft access now available!\n");
    InitializeAIAircraft();
}

// Request access with specific aircraft
int RequestAircraftAccess() {
    char* aircraftList[] = {
        "",  // Don't load aircraft at index 0 (user aircraft)
        "C:\\X-Plane 12\\Aircraft\\Laminar Research\\Boeing B737-800\\b738.acf",
        "C:\\X-Plane 12\\Aircraft\\Laminar Research\\Airbus A330\\A330.acf",
        "",  // Empty slot
        NULL // Terminator
    };
  
    int granted = XPLMAcquirePlanes(aircraftList, PlanesAvailableCallback, NULL);
  
    if (granted) {
        XPLMDebugString("Aircraft access granted immediately\n");
        InitializeAIAircraft();
        return 1;
    } else {
        XPLMDebugString("Aircraft access pending...\n");
        return 0;
    }
}

// Request access without specifying aircraft
int RequestSimpleAccess() {
    return XPLMAcquirePlanes(NULL, PlanesAvailableCallback, NULL);
}
```

### XPLMReleasePlanes

```c
XPLM_API void XPLMReleasePlanes(void);
```

Releases exclusive access to AI aircraft, allowing other plugins to take control.

**Important Notes:**

- Automatically called when plugin is disabled
- Must reacquire access after being disabled/enabled
- Good practice to release when not actively controlling aircraft

**Usage:**

```c
// Release access when done
void CleanupAIControl() {
    XPLMDebugString("Releasing aircraft control\n");
    XPLMReleasePlanes();
}

// In plugin disable function
PLUGIN_API void XPluginDisable(void) {
    if (gHaveAircraftControl) {
        XPLMReleasePlanes();
        gHaveAircraftControl = 0;
    }
}
```

## Aircraft Management Functions

### XPLMSetActiveAircraftCount

```c
XPLM_API void XPLMSetActiveAircraftCount(int inCount);
```

Sets the number of active aircraft in the simulation.

**Parameters:**

- `inCount`: Desired number of active aircraft (including user aircraft)

**Behavior:**

- Limited by total aircraft capacity
- Excess aircraft beyond capacity are ignored
- Requires exclusive aircraft access

**Usage:**

```c
void ConfigureAircraftCount() {
    int totalCapacity, currentActive;
    XPLMPluginID controller;
  
    XPLMCountAircraft(&totalCapacity, &currentActive, &controller);
  
    // Set to 75% of capacity
    int desiredCount = (totalCapacity * 3) / 4;
    XPLMSetActiveAircraftCount(desiredCount);
  
    printf("Set active aircraft count to %d (capacity: %d)\n", 
           desiredCount, totalCapacity);
}
```

### XPLMSetAircraftModel

```c
XPLM_API void XPLMSetAircraftModel(
    int          inIndex,         // Aircraft index (1+, not 0)
    const char * inAircraftPath   // Complete path to .acf file
);
```

Loads a specific aircraft model at the given index.

**Parameters:**

- `inIndex`: Aircraft index (must be > 0, cannot change user aircraft)
- `inAircraftPath`: Complete file system path to .acf file

**Restrictions:**

- Requires exclusive aircraft access
- Cannot modify user aircraft (index 0) - use XPLMSetUsersAircraft instead
- Index must be valid (within active aircraft count)

**Usage:**

```c
void LoadAIAircraft() {
    if (!gHaveAircraftControl) {
        XPLMDebugString("No aircraft access - cannot load AI aircraft\n");
        return;
    }
  
    // Load commercial aircraft in slots 1-3
    const char* commercialAircraft[] = {
        "C:\\X-Plane 12\\Aircraft\\Laminar Research\\Boeing B737-800\\b738.acf",
        "C:\\X-Plane 12\\Aircraft\\Laminar Research\\Airbus A330\\A330.acf", 
        "C:\\X-Plane 12\\Aircraft\\Laminar Research\\Boeing B747-400\\b744.acf"
    };
  
    for (int i = 0; i < 3; i++) {
        XPLMSetAircraftModel(i + 1, commercialAircraft[i]);
        printf("Loaded aircraft at index %d: %s\n", i + 1, commercialAircraft[i]);
    }
}
```

### XPLMDisableAIForPlane

```c
XPLM_API void XPLMDisableAIForPlane(int inPlaneIndex);
```

Disables X-Plane's built-in AI for a specific aircraft.

**Parameters:**

- `inPlaneIndex`: Aircraft index to disable AI for

**Behavior:**

- Aircraft continues to be rendered and exists in simulation
- Aircraft will not move or update on its own
- Plugin assumes full control of aircraft behavior
- Useful for custom AI implementations

**Use Cases:**

```c
void SetupCustomAI() {
    // Disable AI for aircraft 1-5 to implement custom behavior
    for (int i = 1; i <= 5; i++) {
        XPLMDisableAIForPlane(i);
    }
  
    // Now we control these aircraft completely
    RegisterCustomAIUpdate();
}

// Custom AI update function
void UpdateCustomAI() {
    for (int i = 1; i <= 5; i++) {
        UpdateAircraftPosition(i);
        UpdateAircraftSystems(i);
        UpdateAircraftBehavior(i);
    }
}
```

## Deprecated Functions

### XPLMDrawAircraft (DEPRECATED)

```c
XPLM_API void XPLMDrawAircraft(
    int                   inPlaneIndex,
    float                 inX, inY, inZ,
    float                 inPitch, inRoll, inYaw,
    int                   inFullDraw,
    XPLMPlaneDrawState_t * inDrawStateInfo
);
```

**WARNING**: This function is deprecated and will not work in future X-Plane versions. Use XPLMInstance API for 3D aircraft rendering instead.

### XPLMReinitUsersPlane (DEPRECATED)

```c
XPLM_API void XPLMReinitUsersPlane(void);
```

**WARNING**: Do not use. Use XPLMPlaceUserAtAirport or XPLMPlaceUserAtLocation instead.

## Implementation Patterns

### Traffic Management System

```c
#include "XPLMPlanes.h"
#include "XPLMDataAccess.h"

// Traffic manager state
typedef struct {
    int aircraftIndex;
    float latitude;
    float longitude; 
    float altitude;
    float heading;
    float speed;
    char callsign[16];
} TrafficAircraft;

static int gHaveAircraftControl = 0;
static TrafficAircraft gTrafficAircraft[20];
static int gTrafficCount = 0;

// Initialize traffic system
int InitializeTrafficSystem() {
    int result = XPLMAcquirePlanes(NULL, TrafficAvailableCallback, NULL);
  
    if (result) {
        XPLMDebugString("Traffic system: Acquired aircraft control\n");
        SetupTrafficAircraft();
        gHaveAircraftControl = 1;
    } else {
        XPLMDebugString("Traffic system: Waiting for aircraft access\n");
    }
  
    return result;
}

void TrafficAvailableCallback(void* refcon) {
    XPLMDebugString("Traffic system: Aircraft now available\n");
    SetupTrafficAircraft();
    gHaveAircraftControl = 1;
}

void SetupTrafficAircraft() {
    // Configure for 10 AI aircraft + user
    XPLMSetActiveAircraftCount(11);
  
    // Load commercial aircraft for traffic
    const char* aircraftPath = "C:\\X-Plane 12\\Aircraft\\Laminar Research\\Boeing B737-800\\b738.acf";
  
    for (int i = 1; i <= 10; i++) {
        XPLMSetAircraftModel(i, aircraftPath);
        XPLMDisableAIForPlane(i);
      
        // Initialize traffic data
        gTrafficAircraft[i-1].aircraftIndex = i;
        snprintf(gTrafficAircraft[i-1].callsign, sizeof(gTrafficAircraft[i-1].callsign), 
                 "AI%03d", i);
    }
  
    gTrafficCount = 10;
    XPLMDebugString("Traffic system: Configured 10 AI aircraft\n");
}
```

### Aircraft Spawning System

```c
typedef struct {
    char modelPath[512];
    char name[64];
    int category;  // 0=GA, 1=Commercial, 2=Military
} AircraftTemplate;

static AircraftTemplate gAircraftTemplates[] = {
    {"Aircraft\\Laminar Research\\Cessna 172SP\\Cessna_172SP.acf", "Cessna 172", 0},
    {"Aircraft\\Laminar Research\\Boeing B737-800\\b738.acf", "Boeing 737", 1},
    {"Aircraft\\Laminar Research\\Airbus A330\\A330.acf", "Airbus A330", 1},
    {"Aircraft\\Laminar Research\\F-4 Phantom II\\F-4.acf", "F-4 Phantom", 2}
};

int SpawnAircraft(int templateIndex, float lat, float lon, float altitude, float heading) {
    if (!gHaveAircraftControl) {
        XPLMDebugString("Cannot spawn aircraft - no control access\n");
        return -1;
    }
  
    if (templateIndex >= sizeof(gAircraftTemplates) / sizeof(gAircraftTemplates[0])) {
        return -1;
    }
  
    // Find available aircraft slot
    int aircraftSlot = FindAvailableAircraftSlot();
    if (aircraftSlot == -1) {
        XPLMDebugString("No available aircraft slots\n");
        return -1;
    }
  
    // Build complete path
    char systemPath[512];
    XPLMGetSystemPath(systemPath);
  
    char fullPath[1024];
    snprintf(fullPath, sizeof(fullPath), "%s%s", 
             systemPath, gAircraftTemplates[templateIndex].modelPath);
  
    // Load aircraft model
    XPLMSetAircraftModel(aircraftSlot, fullPath);
    XPLMDisableAIForPlane(aircraftSlot);
  
    // Set initial position via datarefs
    SetAircraftPosition(aircraftSlot, lat, lon, altitude, heading);
  
    printf("Spawned %s at slot %d\n", 
           gAircraftTemplates[templateIndex].name, aircraftSlot);
  
    return aircraftSlot;
}
```

### Aircraft Position Management

```c
void SetAircraftPosition(int aircraftIndex, float lat, float lon, float alt, float heading) {
    // Create dataref names for specific aircraft
    char latDataref[128], lonDataref[128], altDataref[128], headingDataref[128];
  
    snprintf(latDataref, sizeof(latDataref), 
             "sim/multiplayer/position/plane%d_lat", aircraftIndex);
    snprintf(lonDataref, sizeof(lonDataref), 
             "sim/multiplayer/position/plane%d_lon", aircraftIndex);
    snprintf(altDataref, sizeof(altDataref), 
             "sim/multiplayer/position/plane%d_el", aircraftIndex);
    snprintf(headingDataref, sizeof(headingDataref), 
             "sim/multiplayer/position/plane%d_psi", aircraftIndex);
  
    // Set position
    XPLMSetDataf(XPLMFindDataRef(latDataref), lat);
    XPLMSetDataf(XPLMFindDataRef(lonDataref), lon);
    XPLMSetDataf(XPLMFindDataRef(altDataref), alt);
    XPLMSetDataf(XPLMFindDataRef(headingDataref), heading);
}

void UpdateAircraftTrajectory(int aircraftIndex, float deltaTime) {
    // Get current position
    float lat = GetAircraftLatitude(aircraftIndex);
    float lon = GetAircraftLongitude(aircraftIndex);
    float alt = GetAircraftAltitude(aircraftIndex);
    float heading = GetAircraftHeading(aircraftIndex);
    float speed = GetAircraftSpeed(aircraftIndex);
  
    // Calculate movement
    float distance = speed * deltaTime;
    float deltaLat, deltaLon;
  
    CalculateMovement(distance, heading, &deltaLat, &deltaLon);
  
    // Update position
    SetAircraftPosition(aircraftIndex, lat + deltaLat, lon + deltaLon, alt, heading);
}
```

## Best Practices

### Access Management

1. **Request Early**: Acquire aircraft access during plugin enable
2. **Handle Callbacks**: Always provide callback for delayed access
3. **Release Cleanly**: Release access when shutting down
4. **Check Status**: Verify access before aircraft operations

```c
static int gPendingAircraftAccess = 0;

PLUGIN_API int XPluginEnable(void) {
    if (!XPLMAcquirePlanes(NULL, DelayedAircraftSetup, NULL)) {
        XPLMDebugString("Aircraft access pending...\n");
        gPendingAircraftAccess = 1;
    }
    return 1;
}

PLUGIN_API void XPluginDisable(void) {
    if (gHaveAircraftControl) {
        XPLMReleasePlanes();
        gHaveAircraftControl = 0;
    }
}
```

### Path Construction

```c
char* BuildAircraftPath(const char* relativePath) {
    static char fullPath[1024];
    char systemPath[512];
  
    XPLMGetSystemPath(systemPath);
    snprintf(fullPath, sizeof(fullPath), "%s%s", systemPath, relativePath);
  
    return fullPath;
}

// Usage
XPLMSetUsersAircraft(BuildAircraftPath("Aircraft/Laminar Research/Cessna 172SP/Cessna_172SP.acf"));
```

### Error Handling

```c
int SafeAircraftOperation(int aircraftIndex) {
    // Check access
    int total, active;
    XPLMPluginID controller;
    XPLMCountAircraft(&total, &active, &controller);
  
    if (controller != XPLMGetMyID()) {
        XPLMDebugString("No aircraft control access\n");
        return 0;
    }
  
    // Check valid index
    if (aircraftIndex < 0 || aircraftIndex >= active) {
        XPLMDebugString("Invalid aircraft index\n");
        return 0;
    }
  
    // Perform operation
    return 1;
}
```

## Integration with Other Systems

### Message Handling

```c
PLUGIN_API void XPluginReceiveMessage(XPLMPluginID from, int msg, void* param) {
    switch (msg) {
        case XPLM_MSG_PLANE_LOADED:
            OnAircraftLoaded((int)(intptr_t)param);
            break;
          
        case XPLM_MSG_PLANE_UNLOADED:
            OnAircraftUnloaded((int)(intptr_t)param);
            break;
          
        case XPLM_MSG_AIRPLANE_COUNT_CHANGED:
            OnAircraftCountChanged();
            break;
          
        case XPLM_MSG_RELEASE_PLANES:
            OnPlanesReleaseRequested(from);
            break;
    }
}

void OnPlanesReleaseRequested(XPLMPluginID requestingPlugin) {
    char pluginName[256];
    XPLMGetPluginInfo(requestingPlugin, pluginName, NULL, NULL, NULL);
  
    printf("Plugin %s requesting aircraft control\n", pluginName);
  
    // Yield to online networks, hold against synthetic traffic
    if (IsOnlineNetworkPlugin(requestingPlugin)) {
        XPLMDebugString("Yielding to online network\n");
        XPLMReleasePlanes();
        gHaveAircraftControl = 0;
    }
}
```

### Dataref Integration

```c
void MonitorUserAircraft() {
    static XPLMDataRef latRef = NULL, lonRef = NULL, altRef = NULL;
  
    if (!latRef) {
        latRef = XPLMFindDataRef("sim/flightmodel/position/latitude");
        lonRef = XPLMFindDataRef("sim/flightmodel/position/longitude");
        altRef = XPLMFindDataRef("sim/flightmodel/position/elevation");
    }
  
    float lat = XPLMGetDataf(latRef);
    float lon = XPLMGetDataf(lonRef);
    float alt = XPLMGetDataf(altRef);
  
    printf("User aircraft position: %.6f, %.6f at %.1f ft\n", lat, lon, alt * 3.28084);
}
```

## Version Compatibility

- **Base Aircraft Control**: All SDK versions
- **XPLMPlaceUserAtLocation**: XPLM300+ (X-Plane 11+)
- **Deprecated Drawing Functions**: Removed in modern X-Plane versions

## Common Use Cases

### 1. Aircraft Selection System

```c
void PresentAircraftMenu() {
    // Build list of available aircraft
    char searchPath[512];
    XPLMGetSystemPath(searchPath);
    strcat(searchPath, "Aircraft");
  
    // Scan for .acf files
    // Present menu to user
    // Load selected aircraft
}
```

### 2. Formation Flying

```c
void MaintainFormation(int leaderIndex, int wingmanIndex) {
    // Get leader position
    float leaderLat = GetAircraftLatitude(leaderIndex);
    float leaderLon = GetAircraftLongitude(leaderIndex);
    float leaderHeading = GetAircraftHeading(leaderIndex);
  
    // Calculate wingman position
    float wingmanLat, wingmanLon;
    CalculateFormationPosition(leaderLat, leaderLon, leaderHeading, 
                              &wingmanLat, &wingmanLon);
  
    // Set wingman position
    SetAircraftPosition(wingmanIndex, wingmanLat, wingmanLon, 
                       GetAircraftAltitude(leaderIndex), leaderHeading);
}
```

### 3. Emergency Scenarios

```c
void TriggerEmergencyScenario() {
    // Place user at emergency location
    XPLMPlaceUserAtLocation(
        40.6892,   // Emergency airport lat
        -74.1745,  // Emergency airport lon  
        10.0,      // Low altitude
        270.0,     // Emergency heading
        50.0       // Emergency speed
    );
  
    XPLMDebugString("Emergency scenario activated!\n");
}
```
