# XPLMDataAccess.h - X-Plane 12 SDK Data Access API Documentation

## Overview

The XPLMDataAccess API provides a powerful, flexible, and high-performance data access system for reading and writing simulator data in X-Plane 12. This API is the primary interface for accessing aircraft systems, simulator state, environmental data, and custom plugin data.

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Core Concepts](#core-concepts)
- [Data Types and Structures](#data-types-and-structures)
- [Data Reference Operations](#data-reference-operations)
- [Data Accessors](#data-accessors)
- [Publishing Custom Data](#publishing-custom-data)
- [Shared Data References](#shared-data-references)
- [Implementation Guidelines](#implementation-guidelines)
- [Performance Considerations](#performance-considerations)
- [Best Practices](#best-practices)
- [Common Use Cases](#common-use-cases)
- [Error Handling](#error-handling)

## Architecture Overview

The Data Access API operates on opaque data references (datarefs) that represent specific pieces of simulator data. The system is designed around the following principles:

- **Opaque Handles**: Data references are opaque pointers that hide implementation details
- **String-Based Lookup**: Datarefs are identified by hierarchical string names (e.g., "sim/cockpit/radios/nav1_freq_hz")
- **Type Safety**: Each dataref has specific data types and access permissions
- **Performance**: Reading/writing operations are optimized for real-time performance
- **Plugin Integration**: Plugins can both consume and publish data references

## Core Concepts

### Dataref Workflow

1. **Lookup Phase**: Find datarefs by string path during plugin startup
2. **Caching**: Store the opaque dataref handle for the plugin's lifetime
3. **Access Phase**: Read/write data using typed functions during runtime
4. **Type Validation**: Verify dataref types match expected usage

### Data Reference Lifecycle

- **Discovery**: `XPLMFindDataRef()` locates existing datarefs
- **Active**: Normal read/write operations
- **Orphaned**: Plugin providing the dataref is unloaded (read returns 0)
- **Re-registered**: Original plugin reloads and dataref becomes active again

## Data Types and Structures

### XPLMDataRef

```c
typedef void * XPLMDataRef;
```

Opaque handle to a data reference. Never hardcode these values - always obtain through `XPLMFindDataRef()`.

### XPLMDataTypeID

```c
enum {
    xplmType_Unknown = 0,        // Unknown data type
    xplmType_Int = 1,            // 4-byte integer, native endian
    xplmType_Float = 2,          // 4-byte float, native endian
    xplmType_Double = 4,         // 8-byte double, native endian
    xplmType_FloatArray = 8,     // Array of 4-byte floats
    xplmType_IntArray = 16,      // Array of 4-byte integers
    xplmType_Data = 32           // Variable block of data
};
```

**Key Points:**

- Data types are bitfields - a dataref can support multiple types
- Use `XPLMGetDataRefTypes()` to check supported types
- Choose the most appropriate type for your use case

### XPLMDataRefInfo_t (XPLM400+)

```c
typedef struct {
    int structSize;              // Size of structure
    const char *name;            // Full dataref path
    XPLMDataTypeID type;         // Supported data types
    int writable;                // TRUE if writable
    XPLMPluginID owner;          // Plugin that created this dataref
} XPLMDataRefInfo_t;
```

## Data Reference Operations

### Core Lookup Functions

#### XPLMFindDataRef

```c
XPLM_API XPLMDataRef XPLMFindDataRef(const char *inDataRefName);
```

**Purpose**: Finds a dataref by its string name.

**Parameters**:

- `inDataRefName`: C-string containing the dataref path

**Returns**: XPLMDataRef handle or NULL if not found

**Example**:

```c
// Look up datarefs during plugin startup
static XPLMDataRef gAltitudeRef = NULL;
static XPLMDataRef gNavFreqRef = NULL;

int XPluginStart(char* outName, char* outSig, char* outDesc) {
    // Cache dataref handles
    gAltitudeRef = XPLMFindDataRef("sim/cockpit2/gauges/indicators/altitude_ft_pilot");
    gNavFreqRef = XPLMFindDataRef("sim/cockpit/radios/nav1_freq_hz");
  
    if (gAltitudeRef == NULL) {
        XPLMDebugString("Failed to find altitude dataref\n");
        return 0;
    }
  
    return 1;
}
```

#### XPLMCanWriteDataRef

```c
XPLM_API int XPLMCanWriteDataRef(XPLMDataRef inDataRef);
```

**Purpose**: Checks if a dataref is writable.

**Important Notes**:

- Even writable datarefs may not respond to writes if X-Plane overwrites them each frame
- Some datarefs require setting an "override" dataref to prevent X-Plane updates

#### XPLMIsDataRefGood

```c
XPLM_API int XPLMIsDataRefGood(XPLMDataRef inDataRef);
```

**Purpose**: Validates that a dataref handle is valid and not orphaned.

**Performance Note**: This function has overhead - avoid calling it frequently.

#### XPLMGetDataRefTypes

```c
XPLM_API XPLMDataTypeID XPLMGetDataRefTypes(XPLMDataRef inDataRef);
```

**Purpose**: Returns the supported data types for a dataref (bitwise OR).

**Example**:

```c
XPLMDataTypeID types = XPLMGetDataRefTypes(myDataRef);
if (types & xplmType_Float) {
    // Can read as float
    float value = XPLMGetDataf(myDataRef);
}
if (types & xplmType_Double) {
    // Can also read as double
    double preciseValue = XPLMGetDatad(myDataRef);
}
```

### Introspection Functions (XPLM400+)

#### XPLMCountDataRefs

```c
XPLM_API int XPLMCountDataRefs(void);
```

**Purpose**: Returns total number of registered datarefs.

**Use Case**: Building dataref browsers or development tools.

#### XPLMGetDataRefsByIndex

```c
XPLM_API void XPLMGetDataRefsByIndex(int offset, int count, XPLMDataRef *outDataRefs);
```

**Purpose**: Retrieves datarefs by index range (pagination support).

#### XPLMGetDataRefInfo

```c
XPLM_API void XPLMGetDataRefInfo(XPLMDataRef inDataRef, XPLMDataRefInfo_t *outInfo);
```

**Purpose**: Gets detailed information about a dataref.

**Example**:

```c
XPLMDataRefInfo_t info;
info.structSize = sizeof(XPLMDataRefInfo_t);
XPLMGetDataRefInfo(myDataRef, &info);

XPLMDebugString("Dataref: ");
XPLMDebugString(info.name);
if (info.writable) {
    XPLMDebugString(" (writable)\n");
} else {
    XPLMDebugString(" (read-only)\n");
}
```

## Data Accessors

### Integer Data Access

#### XPLMGetDatai / XPLMSetDatai

```c
XPLM_API int XPLMGetDatai(XPLMDataRef inDataRef);
XPLM_API void XPLMSetDatai(XPLMDataRef inDataRef, int inValue);
```

**Use Cases**:

- Boolean states (gear position, autopilot modes)
- Discrete settings (radio frequencies in Hz)
- Counters and indices

**Example**:

```c
// Read gear position
XPLMDataRef gearRef = XPLMFindDataRef("sim/aircraft/parts/acf_gear_deploy");
int gearDown = XPLMGetDatai(gearRef);

// Set transponder code
XPLMDataRef xpdrRef = XPLMFindDataRef("sim/cockpit/radios/transponder_code");
XPLMSetDatai(xpdrRef, 7000);
```

### Floating Point Data Access

#### XPLMGetDataf / XPLMSetDataf

```c
XPLM_API float XPLMGetDataf(XPLMDataRef inDataRef);
XPLM_API void XPLMSetDataf(XPLMDataRef inDataRef, float inValue);
```

**Use Cases**:

- Physical measurements (altitude, speed, temperature)
- Ratios and percentages (throttle position, flap extension)
- Angular measurements (heading, pitch, roll)

#### XPLMGetDatad / XPLMSetDatad

```c
XPLM_API double XPLMGetDatad(XPLMDataRef inDataRef);
XPLM_API void XPLMSetDatad(XPLMDataRef inDataRef, double inValue);
```

**Use Cases**:

- High-precision coordinates (latitude, longitude)
- Time values requiring precision
- Mathematical calculations requiring double precision

### Array Data Access

#### XPLMGetDatavi / XPLMSetDatavi (Integer Arrays)

```c
XPLM_API int XPLMGetDatavi(XPLMDataRef inDataRef, int *outValues, int inOffset, int inMax);
XPLM_API void XPLMSetDatavi(XPLMDataRef inDataRef, int *inValues, int inOffset, int inCount);
```

**Parameters**:

- `outValues`: Buffer to receive data (can be NULL to get array size)
- `inOffset`: Starting index in the dataref array
- `inMax`/`inCount`: Maximum number of elements to read/write

**Example**:

```c
// Get array size first
XPLMDataRef enginesRef = XPLMFindDataRef("sim/aircraft/engine/acf_num_engines");
int numEngines = XPLMGetDatavi(enginesRef, NULL, 0, 0);

// Read engine power data
int enginePower[8];
int actualRead = XPLMGetDatavi(enginePowerRef, enginePower, 0, numEngines);
```

#### XPLMGetDatavf / XPLMSetDatavf (Float Arrays)

```c
XPLM_API int XPLMGetDatavf(XPLMDataRef inDataRef, float *outValues, int inOffset, int inMax);
XPLM_API void XPLMSetDatavf(XPLMDataRef inDataRef, float *inValues, int inOffset, int inCount);
```

**Common Use Cases**:

- Engine parameters (N1, N2, EGT arrays)
- Control surface positions
- Weather data arrays
- Aircraft position arrays

### Binary Data Access

#### XPLMGetDatab / XPLMSetDatab

```c
XPLM_API int XPLMGetDatab(XPLMDataRef inDataRef, void *outValue, int inOffset, int inMaxBytes);
XPLM_API void XPLMSetDatab(XPLMDataRef inDataRef, void *inValue, int inOffset, int inLength);
```

**Use Cases**:

- String data
- Binary protocol data
- Custom data structures
- File content

## Publishing Custom Data

### Callback Function Types

The API provides function pointer types for each data access pattern:

```c
// Integer accessors
typedef int (*XPLMGetDatai_f)(void *inRefcon);
typedef void (*XPLMSetDatai_f)(void *inRefcon, int inValue);

// Float accessors
typedef float (*XPLMGetDataf_f)(void *inRefcon);
typedef void (*XPLMSetDataf_f)(void *inRefcon, float inValue);

// Double accessors
typedef double (*XPLMGetDatad_f)(void *inRefcon);
typedef void (*XPLMSetDatad_f)(void *inRefcon, double inValue);

// Integer array accessors
typedef int (*XPLMGetDatavi_f)(void *inRefcon, int *outValues, int inOffset, int inMax);
typedef void (*XPLMSetDatavi_f)(void *inRefcon, int *inValues, int inOffset, int inCount);

// Float array accessors
typedef int (*XPLMGetDatavf_f)(void *inRefcon, float *outValues, int inOffset, int inMax);
typedef void (*XPLMSetDatavf_f)(void *inRefcon, float *inValues, int inOffset, int inCount);

// Binary data accessors
typedef int (*XPLMGetDatab_f)(void *inRefcon, void *outValue, int inOffset, int inMaxLength);
typedef void (*XPLMSetDatab_f)(void *inRefcon, void *inValue, int inOffset, int inLength);
```

### XPLMRegisterDataAccessor

```c
XPLM_API XPLMDataRef XPLMRegisterDataAccessor(
    const char *inDataName,           // Unique dataref name
    XPLMDataTypeID inDataType,        // Supported data types (bitfield)
    int inIsWritable,                 // Whether dataref supports writing
    XPLMGetDatai_f inReadInt,         // Integer read callback (or NULL)
    XPLMSetDatai_f inWriteInt,        // Integer write callback (or NULL)
    XPLMGetDataf_f inReadFloat,       // Float read callback (or NULL)
    XPLMSetDataf_f inWriteFloat,      // Float write callback (or NULL)
    XPLMGetDatad_f inReadDouble,      // Double read callback (or NULL)
    XPLMSetDatad_f inWriteDouble,     // Double write callback (or NULL)
    XPLMGetDatavi_f inReadIntArray,   // Integer array read callback (or NULL)
    XPLMSetDatavi_f inWriteIntArray,  // Integer array write callback (or NULL)
    XPLMGetDatavf_f inReadFloatArray, // Float array read callback (or NULL)
    XPLMSetDatavf_f inWriteFloatArray,// Float array write callback (or NULL)
    XPLMGetDatab_f inReadData,        // Binary read callback (or NULL)
    XPLMSetDatab_f inWriteData,       // Binary write callback (or NULL)
    void *inReadRefcon,               // Reference pointer for read callbacks
    void *inWriteRefcon               // Reference pointer for write callbacks
);
```

### Example: Custom Integer Dataref

```c
// Global storage for our custom data
static int gCustomValue = 42;

// Read callback
int MyCustomIntRead(void *inRefcon) {
    return gCustomValue;
}

// Write callback
void MyCustomIntWrite(void *inRefcon, int inValue) {
    gCustomValue = inValue;
    // Optional: notify other systems of the change
    XPLMDebugString("Custom value changed\n");
}

// Registration
void RegisterCustomDataref() {
    XPLMDataRef myDataRef = XPLMRegisterDataAccessor(
        "myplugin/custom_value",           // Use your plugin prefix!
        xplmType_Int,                      // Integer type only
        1,                                 // Writable
        MyCustomIntRead,                   // Read callback
        MyCustomIntWrite,                  // Write callback
        NULL, NULL,                        // No float support
        NULL, NULL,                        // No double support
        NULL, NULL,                        // No int array support
        NULL, NULL,                        // No float array support
        NULL, NULL,                        // No binary data support
        NULL,                              // Read refcon
        NULL                               // Write refcon
    );
}
```

### Example: Custom Array Dataref

```c
#define MAX_ENGINES 8
static float gEngineTemperatures[MAX_ENGINES] = {0};
static int gNumEngines = 4;

int MyEngineTempsRead(void *inRefcon, float *outValues, int inOffset, int inMax) {
    // Return array size if outValues is NULL
    if (outValues == NULL) {
        return gNumEngines;
    }
  
    // Bounds checking
    if (inOffset >= gNumEngines) return 0;
    int available = gNumEngines - inOffset;
    int toReturn = (inMax < available) ? inMax : available;
  
    // Copy data
    for (int i = 0; i < toReturn; i++) {
        outValues[i] = gEngineTemperatures[inOffset + i];
    }
  
    return toReturn;
}

void MyEngineTempsWrite(void *inRefcon, float *inValues, int inOffset, int inCount) {
    // Bounds checking
    if (inOffset >= gNumEngines) return;
    int available = gNumEngines - inOffset;
    int toWrite = (inCount < available) ? inCount : available;
  
    // Update values
    for (int i = 0; i < toWrite; i++) {
        gEngineTemperatures[inOffset + i] = inValues[i];
    }
}
```

## Shared Data References

Shared datarefs allow multiple plugins to share data with automatic notification when values change.

### XPLMDataChanged_f

```c
typedef void (*XPLMDataChanged_f)(void *inRefcon);
```

Callback function called when shared data changes.

### XPLMShareData

```c
XPLM_API int XPLMShareData(
    const char *inDataName,              // Dataref name
    XPLMDataTypeID inDataType,           // Data type
    XPLMDataChanged_f inNotificationFunc,// Change notification callback
    void *inNotificationRefcon           // Reference pointer for callback
);
```

### XPLMUnshareData

```c
XPLM_API int XPLMUnshareData(
    const char *inDataName,
    XPLMDataTypeID inDataType,
    XPLMDataChanged_f inNotificationFunc,
    void *inNotificationRefcon
);
```

### Example: Shared Data Usage

```c
void OnSharedDataChanged(void *inRefcon) {
    // Shared data changed - read the new value
    XPLMDataRef sharedRef = XPLMFindDataRef("shared/autopilot_target_altitude");
    if (sharedRef) {
        float newAltitude = XPLMGetDataf(sharedRef);
        XPLMDebugString("Shared altitude changed\n");
    }
}

void SetupSharedData() {
    // Connect to shared dataref
    int success = XPLMShareData(
        "shared/autopilot_target_altitude",
        xplmType_Float,
        OnSharedDataChanged,
        NULL
    );
  
    if (success) {
        XPLMDebugString("Successfully connected to shared data\n");
    }
}
```

## Implementation Guidelines

### Plugin Lifecycle Integration

```c
// Global dataref handles
static XPLMDataRef gAltitudeRef = NULL;
static XPLMDataRef gCustomDataRef = NULL;

PLUGIN_API int XPluginStart(char* outName, char* outSig, char* outDesc) {
    strcpy(outName, "My Plugin");
    strcpy(outSig, "com.example.myplugin");
    strcpy(outDesc, "Example dataref usage");
  
    // Look up existing datarefs
    gAltitudeRef = XPLMFindDataRef("sim/cockpit2/gauges/indicators/altitude_ft_pilot");
  
    // Create custom datarefs
    gCustomDataRef = XPLMRegisterDataAccessor(
        "myplugin/my_data",
        xplmType_Float,
        1,
        NULL, NULL,
        MyFloatRead, MyFloatWrite,
        NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL,
        NULL, NULL
    );
  
    return 1;
}

PLUGIN_API void XPluginStop(void) {
    // Unregister custom datarefs
    if (gCustomDataRef) {
        XPLMUnregisterDataAccessor(gCustomDataRef);
        gCustomDataRef = NULL;
    }
}
```

### Naming Conventions

**Critical**: Never use the "sim/" prefix for custom datarefs - this is reserved for X-Plane.

**Recommended naming patterns**:

- `company/plugin/category/name` (e.g., "laminar/b737/autopilot/target_altitude")
- `author/system/parameter` (e.g., "johndoe/weather/visibility_meters")

**Register your prefix** at the X-Plane SDK website to avoid conflicts.

## Performance Considerations

### Optimization Strategies

1. **Cache Dataref Handles**: Look up datarefs once during startup
2. **Minimize Lookups**: Never call `XPLMFindDataRef()` in flight loops
3. **Batch Operations**: Group related dataref operations
4. **Appropriate Types**: Use the most suitable data type for your use case
5. **Avoid Unnecessary Validation**: Skip `XPLMIsDataRefGood()` in performance-critical code

### Performance Anti-patterns

```c
// BAD: Looking up datarefs every frame
float MyFlightLoop(...) {
    XPLMDataRef altRef = XPLMFindDataRef("sim/cockpit2/gauges/indicators/altitude_ft_pilot");
    float altitude = XPLMGetDataf(altRef);  // Expensive!
    return -1.0f;
}

// GOOD: Cache dataref handles
static XPLMDataRef gAltRef = NULL;

int XPluginStart(...) {
    gAltRef = XPLMFindDataRef("sim/cockpit2/gauges/indicators/altitude_ft_pilot");
    return 1;
}

float MyFlightLoop(...) {
    float altitude = XPLMGetDataf(gAltRef);  // Fast!
    return -1.0f;
}
```

## Best Practices

### Error Handling

```c
XPLMDataRef FindRequiredDataref(const char* name) {
    XPLMDataRef ref = XPLMFindDataRef(name);
    if (ref == NULL) {
        char msg[512];
        sprintf(msg, "ERROR: Required dataref not found: %s\n", name);
        XPLMDebugString(msg);
    }
    return ref;
}
```

### Type Safety

```c
void SafeDatarefWrite(XPLMDataRef ref, float value) {
    if (ref == NULL) return;
  
    // Verify dataref supports float writes
    XPLMDataTypeID types = XPLMGetDataRefTypes(ref);
    if (!(types & xplmType_Float)) {
        XPLMDebugString("ERROR: Dataref doesn't support float writes\n");
        return;
    }
  
    if (!XPLMCanWriteDataRef(ref)) {
        XPLMDebugString("WARNING: Dataref is read-only\n");
        return;
    }
  
    XPLMSetDataf(ref, value);
}
```

### Array Handling

```c
int SafeReadFloatArray(XPLMDataRef ref, float* buffer, int maxElements) {
    if (ref == NULL || buffer == NULL) return 0;
  
    // Check if dataref supports float arrays
    XPLMDataTypeID types = XPLMGetDataRefTypes(ref);
    if (!(types & xplmType_FloatArray)) {
        return 0;
    }
  
    // Get actual array size
    int arraySize = XPLMGetDatavf(ref, NULL, 0, 0);
    if (arraySize <= 0) return 0;
  
    // Read data safely
    int toRead = (maxElements < arraySize) ? maxElements : arraySize;
    return XPLMGetDatavf(ref, buffer, 0, toRead);
}
```

## Common Use Cases

### Aircraft Systems Integration

```c
typedef struct {
    XPLMDataRef altitude;
    XPLMDataRef airspeed;
    XPLMDataRef heading;
    XPLMDataRef verticalSpeed;
} FlightData_t;

static FlightData_t gFlightData;

void InitFlightDataRefs() {
    gFlightData.altitude = XPLMFindDataRef("sim/cockpit2/gauges/indicators/altitude_ft_pilot");
    gFlightData.airspeed = XPLMFindDataRef("sim/cockpit2/gauges/indicators/airspeed_kts_pilot");
    gFlightData.heading = XPLMFindDataRef("sim/cockpit2/gauges/indicators/heading_electric_deg_mag_pilot");
    gFlightData.verticalSpeed = XPLMFindDataRef("sim/cockpit2/gauges/indicators/vvi_fpm_pilot");
}

void LogFlightData() {
    char msg[512];
    sprintf(msg, "ALT: %.0f, SPD: %.0f, HDG: %.0f, VS: %.0f\n",
        XPLMGetDataf(gFlightData.altitude),
        XPLMGetDataf(gFlightData.airspeed),
        XPLMGetDataf(gFlightData.heading),
        XPLMGetDataf(gFlightData.verticalSpeed));
    XPLMDebugString(msg);
}
```

### Custom Autopilot System

```c
typedef struct {
    float targetAltitude;
    float targetHeading;
    float targetSpeed;
    int altitudeHold;
    int headingHold;
    int speedHold;
} AutopilotState_t;

static AutopilotState_t gAutopilotState = {0};

// Read callbacks
float APTargetAltitudeRead(void* refcon) { return gAutopilotState.targetAltitude; }
float APTargetHeadingRead(void* refcon) { return gAutopilotState.targetHeading; }
int APAltitudeHoldRead(void* refcon) { return gAutopilotState.altitudeHold; }

// Write callbacks
void APTargetAltitudeWrite(void* refcon, float value) {
    if (value >= 0 && value <= 50000) {  // Reasonable limits
        gAutopilotState.targetAltitude = value;
    }
}

void APAltitudeHoldWrite(void* refcon, int value) {
    gAutopilotState.altitudeHold = (value != 0) ? 1 : 0;
}

void RegisterAutopilotDatarefs() {
    XPLMRegisterDataAccessor("myplugin/autopilot/target_altitude", xplmType_Float, 1,
        NULL, NULL, APTargetAltitudeRead, APTargetAltitudeWrite,
        NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL);
      
    XPLMRegisterDataAccessor("myplugin/autopilot/altitude_hold", xplmType_Int, 1,
        APAltitudeHoldRead, APAltitudeHoldWrite, NULL, NULL,
        NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL);
}
```

### Weather Data Interface

```c
typedef struct {
    float visibility;
    float windSpeed;
    float windDirection;
    float temperature;
    float dewpoint;
    float pressure;
} WeatherData_t;

void ReadWeatherData(WeatherData_t* weather) {
    static XPLMDataRef visRef = NULL;
    static XPLMDataRef windSpeedRef = NULL;
    static XPLMDataRef windDirRef = NULL;
    static XPLMDataRef tempRef = NULL;
    static XPLMDataRef dewRef = NULL;
    static XPLMDataRef pressureRef = NULL;
  
    // Initialize refs on first call
    if (visRef == NULL) {
        visRef = XPLMFindDataRef("sim/weather/visibility_reported_m");
        windSpeedRef = XPLMFindDataRef("sim/weather/wind_speed_kt[0]");
        windDirRef = XPLMFindDataRef("sim/weather/wind_direction_degt[0]");
        tempRef = XPLMFindDataRef("sim/weather/temperature_ambient_c");
        dewRef = XPLMFindDataRef("sim/weather/dewpoi_sealevel_c");
        pressureRef = XPLMFindDataRef("sim/weather/barometer_sealevel_inhg");
    }
  
    // Read current weather
    weather->visibility = XPLMGetDataf(visRef);
    weather->windSpeed = XPLMGetDataf(windSpeedRef);
    weather->windDirection = XPLMGetDataf(windDirRef);
    weather->temperature = XPLMGetDataf(tempRef);
    weather->dewpoint = XPLMGetDataf(dewRef);
    weather->pressure = XPLMGetDataf(pressureRef);
}
```

## Error Handling

### Common Error Scenarios

1. **Dataref Not Found**: `XPLMFindDataRef()` returns NULL
2. **Type Mismatch**: Attempting to read/write wrong data type
3. **Read-Only Access**: Attempting to write to read-only dataref
4. **Array Bounds**: Reading/writing beyond array limits
5. **Orphaned Dataref**: Plugin providing dataref has been unloaded

### Error Handling Patterns

```c
// Defensive dataref reading
float SafeGetDataf(XPLMDataRef ref, float defaultValue) {
    if (ref == NULL) {
        return defaultValue;
    }
  
    XPLMDataTypeID types = XPLMGetDataRefTypes(ref);
    if (!(types & xplmType_Float)) {
        return defaultValue;
    }
  
    return XPLMGetDataf(ref);
}

// Validate dataref on startup
int ValidateDatarefs() {
    int allValid = 1;
  
    if (gAltitudeRef == NULL) {
        XPLMDebugString("ERROR: Altitude dataref not found\n");
        allValid = 0;
    }
  
    if (gSpeedRef == NULL) {
        XPLMDebugString("ERROR: Speed dataref not found\n");
        allValid = 0;
    }
  
    return allValid;
}
```

## Version Compatibility

### XPLM400 Features

- Dataref introspection (`XPLMCountDataRefs`, `XPLMGetDataRefsByIndex`)
- Dataref information queries (`XPLMGetDataRefInfo`)
- Dataref change notifications

### Backward Compatibility

```c
#if defined(XPLM400)
    // Use new introspection features
    int totalRefs = XPLMCountDataRefs();
    XPLMDebugString("Total datarefs available: %d\n", totalRefs);
#endif
```
