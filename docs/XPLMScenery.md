# XPLMScenery API Documentation

## Overview

The XPLMScenery API provides comprehensive access to X-Plane's scenery system, enabling plugins to interact with the 3D world environment. This API allows plugins to probe terrain elevation, access magnetic variation data, load and display 3D objects, and query the scenery library system.

## Key Features

- **Terrain Y-Testing**: Probe ground elevation and surface properties at any location
- **Magnetic Variation**: Access realistic magnetic declination data worldwide
- **3D Object Management**: Load, display, and manage X-Plane OBJ files
- **Scenery Library Access**: Query objects from X-Plane's virtual path system
- **Surface Properties**: Determine terrain type, slope, velocity, and wetness
- **Performance Optimized**: Efficient caching and probe reuse strategies

## Architecture

### Terrain System

X-Plane's terrain system provides:

1. **Elevation Data**: High-resolution terrain mesh with accurate elevation
2. **Surface Properties**: Material type, wetness, movement velocity
3. **Dynamic Objects**: Moving terrain (aircraft carriers, ships)
4. **Water Detection**: Identification of water vs. land surfaces
5. **Normal Vectors**: Surface slope information for physics

### Coordinate Systems

- **OpenGL Local Coordinates**: Used for all terrain probing results
- **Geographic Coordinates**: Latitude/longitude for magnetic variation
- **Earth-Relative Coordinates**: For object positioning and rotation

### Performance Design

- **Probe Objects**: Reusable handles that cache algorithm state
- **Spatial Locality**: Better performance when probing nearby points
- **Limited Range**: Probing restricted to loaded scenery area (~300×300 km)

## Data Types and Enumerations

### XPLMProbeType

```c
enum {
    xplm_ProbeY = 0
};
typedef int XPLMProbeType;
```

Defines terrain probe algorithms:

- **xplm_ProbeY**: Y-axis probe finding tallest physical scenery
  - Projects vertically through terrain
  - Returns highest surface intersection
  - Most common probe type for ground placement

### XPLMProbeResult

```c
enum {
    xplm_ProbeHitTerrain = 0,
    xplm_ProbeError     = 1,
    xplm_ProbeMissed    = 2
};
typedef int XPLMProbeResult;
```

Results from terrain probe operations:

- **xplm_ProbeHitTerrain**: Successful probe with valid terrain data
- **xplm_ProbeError**: API call error (invalid probe, bad parameters)
- **xplm_ProbeMissed**: Valid call but no terrain found (off planet edge)

### XPLMProbeRef

```c
typedef void * XPLMProbeRef;
```

Opaque handle to a terrain probe object. Used for:

- Caching probe algorithm state
- Improving performance through reuse
- Managing memory efficiently

### XPLMProbeInfo_t

```c
typedef struct {
    int   structSize;        // Size of structure for API compatibility
    float locationX;         // X coordinate of terrain hit (OpenGL local)
    float locationY;         // Y coordinate of terrain hit (OpenGL local) 
    float locationZ;         // Z coordinate of terrain hit (OpenGL local)
    float normalX;           // X component of surface normal vector
    float normalY;           // Y component of surface normal vector
    float normalZ;           // Z component of surface normal vector
    float velocityX;         // X velocity of terrain (m/s)
    float velocityY;         // Y velocity of terrain (m/s)
    float velocityZ;         // Z velocity of terrain (m/s)
    int   is_wet;           // 1 if water surface, 0 if land
} XPLMProbeInfo_t;
```

Complete terrain probe results containing:

- **Location**: Exact 3D point where probe hit terrain
- **Normal**: Surface slope vector (normalized)
- **Velocity**: Movement of terrain (for dynamic objects like carriers)
- **Surface Type**: Water vs. land classification

### XPLMObjectRef

```c
typedef void * XPLMObjectRef;
```

Handle to loaded 3D object (.obj file). Features:

- Reference counted for automatic memory management
- Shared between multiple users
- Platform-independent resource management

### XPLMDrawInfo_t

```c
typedef struct {
    int   structSize;     // Structure size for compatibility
    float x, y, z;        // Position in local coordinates
    float pitch;          // Rotation around X-axis (degrees)
    float heading;        // Rotation around Y-axis (degrees, clockwise)
    float roll;           // Rotation around Z-axis (degrees)
} XPLMDrawInfo_t;
```

3D object positioning and orientation data:

- **Position**: OpenGL local coordinate placement
- **Rotations**: Euler angles in degrees
- **Multiple Objects**: Array support for batch rendering

## Terrain Y-Testing API (XPLM200+)

### XPLMCreateProbe

```c
XPLM_API XPLMProbeRef XPLMCreateProbe(XPLMProbeType inProbeType);
```

Creates a new terrain probe object for repeated use.

**Parameters**:

- **inProbeType**: Type of probe algorithm (currently only `xplm_ProbeY`)

**Returns**: Probe handle for use with `XPLMProbeTerrainXYZ`, or NULL on error

**Performance Notes**:

- Create probes during plugin initialization
- Reuse same probe for nearby points
- Limit total probes to hundreds maximum

**Example**:

```c
static XPLMProbeRef gTerrainProbe = NULL;

PLUGIN_API int XPluginStart(char* name, char* sig, char* desc) {
    // Create probe during startup
    gTerrainProbe = XPLMCreateProbe(xplm_ProbeY);
    if (!gTerrainProbe) {
        XPLMDebugString("Failed to create terrain probe\n");
        return 0;
    }
    return 1;
}
```

### XPLMDestroyProbe

```c
XPLM_API void XPLMDestroyProbe(XPLMProbeRef inProbe);
```

Releases a terrain probe object and associated resources.

**Parameters**:

- **inProbe**: Probe handle from `XPLMCreateProbe`

**Important**: Always clean up probes during plugin shutdown to prevent memory leaks.

**Example**:

```c
PLUGIN_API void XPluginStop(void) {
    if (gTerrainProbe) {
        XPLMDestroyProbe(gTerrainProbe);
        gTerrainProbe = NULL;
    }
}
```

### XPLMProbeTerrainXYZ

```c
XPLM_API XPLMProbeResult XPLMProbeTerrainXYZ(
    XPLMProbeRef    inProbe,
    float           inX,
    float           inY,
    float           inZ,
    XPLMProbeInfo_t *outInfo
);
```

Performs terrain elevation probe at specified coordinates.

**Parameters**:

- **inProbe**: Probe object for the query
- **inX, inY, inZ**: Query point in OpenGL local coordinates
- **outInfo**: Result structure (must set `structSize` first)

**Returns**: Probe result indicating success or failure type

**Usage Pattern**:

```c
float GetTerrainElevation(float x, float z, float *outElevation) {
    XPLMProbeInfo_t info;
    info.structSize = sizeof(XPLMProbeInfo_t);
  
    XPLMProbeResult result = XPLMProbeTerrainXYZ(
        gTerrainProbe,
        x, 10000.0f, z,  // Probe from high altitude down
        &info
    );
  
    if (result == xplm_ProbeHitTerrain) {
        *outElevation = info.locationY;
        return 1; // Success
    }
  
    return 0; // Failed
}
```

**Advanced Usage - Surface Analysis**:

```c
typedef struct {
    float elevation;
    float slope;
    int isWater;
    int isMoving;
    float surfaceSpeed;
} TerrainInfo;

int AnalyzeTerrain(float x, float z, TerrainInfo *terrain) {
    XPLMProbeInfo_t info;
    info.structSize = sizeof(XPLMProbeInfo_t);
  
    XPLMProbeResult result = XPLMProbeTerrainXYZ(
        gTerrainProbe, x, 10000.0f, z, &info
    );
  
    if (result != xplm_ProbeHitTerrain) {
        return 0;
    }
  
    // Extract terrain properties
    terrain->elevation = info.locationY;
    terrain->isWater = info.is_wet;
  
    // Calculate slope from normal vector
    terrain->slope = acos(info.normalY) * 180.0f / M_PI; // Degrees from horizontal
  
    // Check if surface is moving (carrier deck, etc.)
    float speed = sqrt(info.velocityX * info.velocityX + 
                      info.velocityY * info.velocityY + 
                      info.velocityZ * info.velocityZ);
    terrain->isMoving = (speed > 0.1f); // Moving faster than 0.1 m/s
    terrain->surfaceSpeed = speed;
  
    return 1;
}
```

## Magnetic Variation API (XPLM300+)

### XPLMGetMagneticVariation

```c
XPLM_API float XPLMGetMagneticVariation(
    double latitude,
    double longitude
);
```

Returns magnetic declination (variation) at specified location.

**Parameters**:

- **latitude**: Geographic latitude in degrees
- **longitude**: Geographic longitude in degrees

**Returns**: Magnetic variation in degrees (positive = magnetic north east of true north)

**Usage**:

```c
float GetMagneticHeading(double lat, double lon, float trueHeading) {
    float variation = XPLMGetMagneticVariation(lat, lon);
    return trueHeading - variation;
}

// Example: Convert GPS track to magnetic heading
void UpdateCompass(void) {
    double lat = XPLMGetDataf(latitudeRef);
    double lon = XPLMGetDataf(longitudeRef);
    float trueTrack = XPLMGetDataf(trackRef);
  
    float magneticTrack = GetMagneticHeading(lat, lon, trueTrack);
    XPLMSetDataf(compassHeadingRef, magneticTrack);
}
```

### XPLMDegTrueToDegMagnetic

```c
XPLM_API float XPLMDegTrueToDegMagnetic(float headingDegreesTrue);
```

Converts true heading to magnetic heading at current aircraft location.

**Parameters**:

- **headingDegreesTrue**: Heading relative to true north (degrees)

**Returns**: Heading relative to magnetic north (degrees)

**Example**:

```c
void SetAutopilotHeading(float trueHeading) {
    float magneticHeading = XPLMDegTrueToDegMagnetic(trueHeading);
    XPLMSetDataf(autopilotHeadingRef, magneticHeading);
}
```

### XPLMDegMagneticToDegTrue

```c
XPLM_API float XPLMDegMagneticToDegTrue(float headingDegreesMagnetic);
```

Converts magnetic heading to true heading at current aircraft location.

**Parameters**:

- **headingDegreesMagnetic**: Heading relative to magnetic north (degrees)

**Returns**: Heading relative to true north (degrees)

**Example**:

```c
void ConvertCompassReading(void) {
    float compassHeading = XPLMGetDataf(compassRef);
    float trueHeading = XPLMDegMagneticToDegTrue(compassHeading);
  
    // Use true heading for navigation calculations
    CalculateNavigationTo(targetLat, targetLon, trueHeading);
}
```

## Object Drawing API (XPLM200+)

### XPLMLoadObject

```c
XPLM_API XPLMObjectRef XPLMLoadObject(const char *inPath);
```

Loads a 3D object (.obj) file for rendering.

**Parameters**:

- **inPath**: Path relative to X-Plane root folder

**Returns**: Object handle, or NULL if loading failed

**Important Notes**:

- Objects are reference counted - same file returns same handle
- Must call `XPLMUnloadObject` for each `XPLMLoadObject` call
- Datarefs used by object must be registered before loading
- Loading may be deferred until first use

**Path Guidelines**:

- Use paths relative to X-System folder
- Prefix with `./` if in root directory (not recommended)
- Place plugin objects in plugin subdirectories
- Use forward slashes on all platforms

**Example**:

```c
static XPLMObjectRef gCustomAircraft = NULL;

int LoadPluginObjects(void) {
    // Load custom aircraft model
    gCustomAircraft = XPLMLoadObject("Resources/plugins/MyPlugin/objects/aircraft.obj");
    if (!gCustomAircraft) {
        XPLMDebugString("Failed to load custom aircraft object\n");
        return 0;
    }
  
    return 1;
}
```

### XPLMLoadObjectAsync (XPLM210+)

```c
XPLM_API void XPLMLoadObjectAsync(
    const char         *inPath,
    XPLMObjectLoaded_f inCallback,
    void               *inRefcon
);
```

Loads object asynchronously without blocking the simulation.

**Parameters**:

- **inPath**: Object file path (same rules as `XPLMLoadObject`)
- **inCallback**: Function called when loading completes
- **inRefcon**: User data passed to callback

**Callback Signature**:

```c
typedef void (*XPLMObjectLoaded_f)(
    XPLMObjectRef inObject,  // Loaded object or NULL if failed
    void          *inRefcon  // Your reference data
);
```

**Advantages**:

- No simulation pause during loading
- Better user experience for large objects
- Suitable for runtime object loading

**Example**:

```c
void ObjectLoadedCallback(XPLMObjectRef object, void *refcon) {
    const char *objectName = (const char*)refcon;
  
    if (object) {
        XPLMDebugString("Successfully loaded: ");
        XPLMDebugString(objectName);
        XPLMDebugString("\n");
      
        // Store object reference for use
        StoreLoadedObject(objectName, object);
    } else {
        XPLMDebugString("Failed to load: ");
        XPLMDebugString(objectName);
        XPLMDebugString("\n");
    }
}

void LoadObjectsAsync(void) {
    XPLMLoadObjectAsync(
        "Resources/plugins/MyPlugin/objects/building.obj",
        ObjectLoadedCallback,
        "building"
    );
  
    XPLMLoadObjectAsync(
        "Resources/plugins/MyPlugin/objects/vehicle.obj", 
        ObjectLoadedCallback,
        "vehicle"
    );
}
```

### XPLMUnloadObject

```c
XPLM_API void XPLMUnloadObject(XPLMObjectRef inObject);
```

Decrements object reference count, potentially unloading it from memory.

**Parameters**:

- **inObject**: Object handle from load function

**Reference Counting**:

- Each `XPLMLoadObject` call increments reference count
- Each `XPLMUnloadObject` call decrements reference count
- Object purged from memory when count reaches zero
- Always balance load/unload calls

**Example**:

```c
void CleanupObjects(void) {
    if (gCustomAircraft) {
        XPLMUnloadObject(gCustomAircraft);
        gCustomAircraft = NULL;
    }
}
```

## Library Access API (XPLM200+)

### XPLMLibraryEnumerator_f

```c
typedef void (*XPLMLibraryEnumerator_f)(
    const char *inFilePath,
    void       *inRef
);
```

Callback function for processing library search results.

**Parameters**:

- **inFilePath**: Path to found object (relative to X-System folder)
- **inRef**: User reference data from search call

### XPLMLookupObjects

```c
XPLM_API int XPLMLookupObjects(
    const char                 *inPath,
    float                      inLatitude,
    float                      inLongitude,  
    XPLMLibraryEnumerator_f    enumerator,
    void                       *ref
);
```

Searches X-Plane's virtual library system for objects matching a path.

**Parameters**:

- **inPath**: Virtual path pattern to search for
- **inLatitude**: Geographic latitude for location-specific results
- **inLongitude**: Geographic longitude for location-specific results
- **enumerator**: Callback function for each found object
- **ref**: User data passed to callback

**Returns**: Number of matching objects found

**Virtual Paths**: X-Plane uses virtual paths for scenery objects:

- `lib/aircraft/...` - Aircraft models
- `lib/buildings/...` - Building objects
- `lib/vehicles/...` - Vehicle models
- `lib/airport/...` - Airport infrastructure

**Example - Finding Regional Aircraft**:

```c
typedef struct {
    char paths[10][256];
    int count;
} ObjectList;

void CollectObjects(const char *path, void *ref) {
    ObjectList *list = (ObjectList*)ref;
  
    if (list->count < 10) {
        strcpy(list->paths[list->count], path);
        list->count++;
    }
}

void FindRegionalAircraft(double lat, double lon) {
    ObjectList aircraft;
    aircraft.count = 0;
  
    int found = XPLMLookupObjects(
        "lib/aircraft/general_aviation/Cessna_172.obj",
        lat, lon,
        CollectObjects,
        &aircraft
    );
  
    printf("Found %d aircraft variants:\n", found);
    for (int i = 0; i < aircraft.count; i++) {
        printf("  %s\n", aircraft.paths[i]);
    }
}
```

## Implementation Guidelines

### Best Practices

#### 1. Terrain Probing Strategy

```c
typedef struct {
    XPLMProbeRef probe;
    float lastX, lastZ;
    float cachedElevation;
    int cacheValid;
} TerrainCache;

float GetElevationCached(TerrainCache *cache, float x, float z) {
    // Use cache if point hasn't moved much
    float dx = x - cache->lastX;
    float dz = z - cache->lastZ;
    float distance = sqrt(dx*dx + dz*dz);
  
    if (cache->cacheValid && distance < 1.0f) { // Within 1 meter
        return cache->cachedElevation;
    }
  
    // Probe terrain
    XPLMProbeInfo_t info;
    info.structSize = sizeof(info);
  
    XPLMProbeResult result = XPLMProbeTerrainXYZ(
        cache->probe, x, 10000.0f, z, &info
    );
  
    if (result == xplm_ProbeHitTerrain) {
        cache->cachedElevation = info.locationY;
        cache->lastX = x;
        cache->lastZ = z;
        cache->cacheValid = 1;
        return info.locationY;
    }
  
    return 0.0f; // Sea level default
}
```

#### 2. Object Management System

```c
typedef struct {
    XPLMObjectRef object;
    int refCount;
    char path[256];
} ManagedObject;

static ManagedObject gObjects[100];
static int gObjectCount = 0;

XPLMObjectRef LoadManagedObject(const char *path) {
    // Check if already loaded
    for (int i = 0; i < gObjectCount; i++) {
        if (strcmp(gObjects[i].path, path) == 0) {
            gObjects[i].refCount++;
            return gObjects[i].object;
        }
    }
  
    // Load new object
    if (gObjectCount < 100) {
        XPLMObjectRef obj = XPLMLoadObject(path);
        if (obj) {
            strcpy(gObjects[gObjectCount].path, path);
            gObjects[gObjectCount].object = obj;
            gObjects[gObjectCount].refCount = 1;
            gObjectCount++;
            return obj;
        }
    }
  
    return NULL;
}

void UnloadManagedObject(const char *path) {
    for (int i = 0; i < gObjectCount; i++) {
        if (strcmp(gObjects[i].path, path) == 0) {
            gObjects[i].refCount--;
            if (gObjects[i].refCount <= 0) {
                XPLMUnloadObject(gObjects[i].object);
                // Remove from array (shift others down)
                memmove(&gObjects[i], &gObjects[i+1], 
                       (gObjectCount-i-1) * sizeof(ManagedObject));
                gObjectCount--;
            }
            break;
        }
    }
}
```

#### 3. Magnetic Variation Helper

```c
typedef struct {
    double lastLat, lastLon;
    float cachedVariation;
    float cacheTime;
} MagVarCache;

static MagVarCache gMagVarCache = {0};

float GetMagneticVariationCached(double lat, double lon) {
    float currentTime = XPLMGetElapsedTime();
  
    // Cache is valid for 60 seconds and within 0.1 degrees
    float dt = currentTime - gMagVarCache.cacheTime;
    float dlat = fabs(lat - gMagVarCache.lastLat);
    float dlon = fabs(lon - gMagVarCache.lastLon);
  
    if (dt < 60.0f && dlat < 0.1 && dlon < 0.1) {
        return gMagVarCache.cachedVariation;
    }
  
    // Update cache
    gMagVarCache.cachedVariation = XPLMGetMagneticVariation(lat, lon);
    gMagVarCache.lastLat = lat;
    gMagVarCache.lastLon = lon;
    gMagVarCache.cacheTime = currentTime;
  
    return gMagVarCache.cachedVariation;
}
```

### Common Use Cases

#### 1. Ground-Following Objects

```c
void UpdateGroundVehicles(void) {
    static XPLMProbeRef probe = NULL;
    if (!probe) {
        probe = XPLMCreateProbe(xplm_ProbeY);
    }
  
    for (int i = 0; i < vehicleCount; i++) {
        Vehicle *v = &vehicles[i];
      
        // Probe terrain at vehicle position
        XPLMProbeInfo_t info;
        info.structSize = sizeof(info);
      
        XPLMProbeResult result = XPLMProbeTerrainXYZ(
            probe, v->x, v->y + 100.0f, v->z, &info
        );
      
        if (result == xplm_ProbeHitTerrain) {
            // Place vehicle on ground
            v->y = info.locationY + v->groundClearance;
          
            // Align with terrain slope
            v->pitch = atan2(info.normalZ, info.normalY) * 180.0f / M_PI;
            v->roll = atan2(info.normalX, info.normalY) * 180.0f / M_PI;
          
            // Handle water surfaces
            if (info.is_wet && !v->canSwim) {
                v->isActive = 0; // Disable land vehicle in water
            }
        }
    }
}
```

#### 2. Runway Surface Detection

```c
int IsOnRunway(float x, float z, RunwayInfo *runway) {
    static XPLMProbeRef probe = NULL;
    if (!probe) {
        probe = XPLMCreateProbe(xplm_ProbeY);
    }
  
    XPLMProbeInfo_t info;
    info.structSize = sizeof(info);
  
    XPLMProbeResult result = XPLMProbeTerrainXYZ(
        probe, x, 1000.0f, z, &info
    );
  
    if (result == xplm_ProbeHitTerrain) {
        // Check if surface is relatively flat (runway-like)
        float slope = acos(info.normalY) * 180.0f / M_PI;
      
        if (slope < 5.0f && !info.is_wet) { // Flat, dry surface
            runway->elevation = info.locationY;
            runway->slope = slope;
            return 1;
        }
    }
  
    return 0;
}
```

#### 3. Dynamic Object Library

```c
typedef struct {
    char virtualPath[256];
    XPLMObjectRef *objects;
    int objectCount;
    double lat, lon;
} LibrarySet;

void LibraryEnumeratorCallback(const char *path, void *ref) {
    LibrarySet *set = (LibrarySet*)ref;
  
    // Load found object
    XPLMObjectRef obj = XPLMLoadObject(path);
    if (obj) {
        set->objects = realloc(set->objects, 
            (set->objectCount + 1) * sizeof(XPLMObjectRef));
        set->objects[set->objectCount] = obj;
        set->objectCount++;
    }
}

int LoadLibrarySet(const char *virtualPath, double lat, double lon, 
                   LibrarySet *set) {
    strcpy(set->virtualPath, virtualPath);
    set->lat = lat;
    set->lon = lon;
    set->objects = NULL;
    set->objectCount = 0;
  
    int found = XPLMLookupObjects(virtualPath, lat, lon, 
                                 LibraryEnumeratorCallback, set);
  
    return found;
}

void FreeLibrarySet(LibrarySet *set) {
    for (int i = 0; i < set->objectCount; i++) {
        XPLMUnloadObject(set->objects[i]);
    }
    free(set->objects);
    set->objects = NULL;
    set->objectCount = 0;
}
```

## Performance Considerations

### Terrain Probing Optimization

1. **Probe Reuse**: Use same probe for multiple nearby queries
2. **Spatial Caching**: Cache results for small movements
3. **Batch Processing**: Group probes by location
4. **LOD System**: Reduce probe frequency for distant objects

### Object Management

1. **Reference Counting**: Proper load/unload balance prevents memory leaks
2. **Async Loading**: Use async loading for large objects
3. **Culling**: Don't load objects outside view range
4. **Shared Resources**: Reuse objects when possible

### Memory Management

```c
// Good - Single probe for multiple queries
XPLMProbeRef probe = XPLMCreateProbe(xplm_ProbeY);
for (int i = 0; i < pointCount; i++) {
    ProbePoint(probe, points[i].x, points[i].z);
}
XPLMDestroyProbe(probe);

// Bad - Creating probe for each query  
for (int i = 0; i < pointCount; i++) {
    XPLMProbeRef probe = XPLMCreateProbe(xplm_ProbeY);
    ProbePoint(probe, points[i].x, points[i].z);
    XPLMDestroyProbe(probe);
}
```

## Integration with Other APIs

### Flight Loop Integration

```c
float TerrainUpdateCallback(float elapsed, float total, int counter, void *ref) {
    // Get current aircraft position
    float x = XPLMGetDataf(localXRef);
    float z = XPLMGetDataf(localZRef);
  
    // Update ground elevation for landing gear
    float groundElevation;
    if (GetTerrainElevation(x, z, &groundElevation)) {
        float aircraftY = XPLMGetDataf(localYRef);
        float agl = aircraftY - groundElevation;
      
        // Update AGL dataref for other plugins
        XPLMSetDataf(heightAGLRef, agl);
    }
  
    return 0.1f; // Update every 0.1 seconds
}
```

### Instance API Integration

```c
void UpdateInstancedObjects(void) {
    // Use terrain probing to place instanced objects on ground
    for (int i = 0; i < instanceCount; i++) {
        XPLMProbeInfo_t info;
        info.structSize = sizeof(info);
      
        XPLMProbeResult result = XPLMProbeTerrainXYZ(
            gProbe, instances[i].x, 1000.0f, instances[i].z, &info
        );
      
        if (result == xplm_ProbeHitTerrain) {
            XPLMDrawInfo_t drawInfo;
            drawInfo.structSize = sizeof(drawInfo);
            drawInfo.x = instances[i].x;
            drawInfo.y = info.locationY;
            drawInfo.z = instances[i].z;
            drawInfo.pitch = 0.0f;
            drawInfo.heading = instances[i].heading;
            drawInfo.roll = 0.0f;
          
            // Update instance position
            float datarefValues[1] = { instances[i].animationState };
            XPLMInstanceSetPosition(instances[i].instance, &drawInfo, datarefValues);
        }
    }
}
```

## Error Handling and Debugging

### Probe Error Handling

```c
const char* ProbeResultToString(XPLMProbeResult result) {
    switch (result) {
        case xplm_ProbeHitTerrain: return "Hit Terrain";
        case xplm_ProbeError: return "API Error"; 
        case xplm_ProbeMissed: return "Missed Terrain";
        default: return "Unknown Result";
    }
}

int SafeProbeRequest(float x, float y, float z, XPLMProbeInfo_t *info) {
    if (!gTerrainProbe) {
        XPLMDebugString("Terrain probe not initialized\n");
        return 0;
    }
  
    XPLMProbeResult result = XPLMProbeTerrainXYZ(gTerrainProbe, x, y, z, info);
  
    if (result != xplm_ProbeHitTerrain) {
        char msg[256];
        sprintf(msg, "Terrain probe failed at (%.1f, %.1f, %.1f): %s\n",
                x, y, z, ProbeResultToString(result));
        XPLMDebugString(msg);
        return 0;
    }
  
    return 1;
}
```

### Object Loading Validation

```c
XPLMObjectRef SafeLoadObject(const char *path) {
    if (!path || strlen(path) == 0) {
        XPLMDebugString("Invalid object path\n");
        return NULL;
    }
  
    XPLMObjectRef obj = XPLMLoadObject(path);
  
    if (!obj) {
        char msg[512];
        sprintf(msg, "Failed to load object: %s\n", path);
        XPLMDebugString(msg);
    } else {
        char msg[512];
        sprintf(msg, "Successfully loaded object: %s\n", path);
        XPLMDebugString(msg);
    }
  
    return obj;
}
```

## Version Compatibility

- **XPLM200+**: Core terrain probing, object loading, library access
- **XPLM210+**: Asynchronous object loading
- **XPLM300+**: Magnetic variation API

Version-specific code:

```c
#if defined(XPLM300)
    // Use modern magnetic variation API
    float variation = XPLMGetMagneticVariation(lat, lon);
#else
    // Implement fallback or disable feature
    float variation = 0.0f; // Assume no variation
#endif

#if defined(XPLM210)
    // Use asynchronous loading for better performance
    XPLMLoadObjectAsync(path, callback, refcon);
#else
    // Fall back to synchronous loading
    XPLMObjectRef obj = XPLMLoadObject(path);
    callback(obj, refcon);
#endif
```

## Summary

The XPLMScenery API provides tools for creating ground-aware plugins:

1. **Terrain System**: Accurate elevation and surface property queries
2. **Magnetic Realism**: Proper magnetic variation for navigation accuracy
3. **3D Integration**: Seamless object loading and management
4. **Performance Focus**: Optimized for real-time simulation requirements
5. **Comprehensive Coverage**: From simple elevation checks to complex object libraries

Key design principles:

- **Probe Reuse**: Create probes once, use many times
- **Reference Counting**: Balance object load/unload calls
- **Caching Strategy**: Cache expensive operations appropriately
- **Error Handling**: Graceful degradation when operations fail
- **Performance Awareness**: Consider frame rate impact in all operations
