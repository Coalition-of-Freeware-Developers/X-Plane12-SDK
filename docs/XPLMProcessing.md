# XPLMProcessing API Documentation

## Overview

The XPLMProcessing API provides the foundation for time-based and periodic processing in X-Plane plugins. This API enables plugins to register callbacks that execute during X-Plane's flight loop, allowing for regular updates, data processing, physics calculations, and system monitoring without blocking the main simulation thread.

## Key Features

- **Flight Loop Integration**: Execute code before or after X-Plane's flight model calculations
- **Flexible Scheduling**: Schedule callbacks by time intervals or frame counts
- **Performance-Aware Design**: Built-in timing mechanisms to maintain simulator performance
- **Dual API Support**: Legacy callback system and modern ID-based management
- **Physics Integration**: Hook into X-Plane's flight model processing pipeline

## Architecture

### Flight Loop Overview

X-Plane's flight loop is the core simulation cycle that:

1. Processes input from controls and systems
2. Calculates flight model physics
3. Updates aircraft state and systems
4. Renders the visual scene
5. Handles audio and other subsystems

Plugin callbacks can be scheduled to run:

- **Before Flight Model**: Ideal for setting up conditions, modifying inputs
- **After Flight Model**: Perfect for reading results, logging data, post-processing

### Timing and Scheduling

The API supports two scheduling modes:

- **Time-based**: Callbacks scheduled by wall-clock seconds
- **Frame-based**: Callbacks scheduled by flight loop iterations

**Important**: Scheduling is "first callback after deadline" - callbacks may be slightly late to prevent running faster than requested intervals.

### Performance Considerations

- Callbacks should complete quickly to avoid impacting frame rate
- Expensive operations should be spread across multiple callback invocations
- Inactive callbacks should return 0 to suspend execution
- Use caching and avoid redundant calculations

## Data Types and Enumerations

### XPLMFlightLoopPhaseType (XPLM210+)

```c
enum {
    xplm_FlightLoop_Phase_BeforeFlightModel = 0,
    xplm_FlightLoop_Phase_AfterFlightModel  = 1
};
typedef int XPLMFlightLoopPhaseType;
```

Specifies when your callback executes relative to X-Plane's flight model integration:

- **BeforeFlightModel**: Execute before physics calculations

  - Use for: Input preprocessing, condition setup, data injection
  - Example: Overriding control surface positions
- **AfterFlightModel**: Execute after physics calculations

  - Use for: Data logging, result processing, system updates
  - Example: Recording flight parameters, updating displays

### XPLMFlightLoopID (XPLM210+)

```c
typedef void * XPLMFlightLoopID;
```

Opaque handle for modern flight loop callbacks. Provides:

- Unique identification for each callback
- Simplified management and cleanup
- Enhanced scheduling control

### XPLMFlightLoop_f

```c
typedef float (* XPLMFlightLoop_f)(
    float inElapsedSinceLastCall,
    float inElapsedTimeSinceLastFlightLoop,
    int   inCounter,
    void  *inRefcon
);
```

Your flight loop callback function. Parameters:

- **inElapsedSinceLastCall**: Wall time since your last callback (seconds)
- **inElapsedTimeSinceLastFlightLoop**: Wall time since any flight loop dispatch (seconds)
- **inCounter**: Monotonic counter, incremented each flight loop cycle
- **inRefcon**: Your reference constant provided at registration

**Return Value Controls Next Execution**:

- **0**: Stop receiving callbacks (callback becomes inactive but remains registered)
- **Positive**: Seconds until next callback (minimum interval)
- **Negative**: Flight loop cycles until next callback (-1 = next cycle)

### XPLMCreateFlightLoop_t (XPLM210+)

```c
typedef struct {
    int                      structSize;
    XPLMFlightLoopPhaseType  phase;
    XPLMFlightLoop_f         callbackFunc;
    void                     *refcon;
} XPLMCreateFlightLoop_t;
```

Configuration structure for modern flight loop creation:

- **structSize**: Set to `sizeof(XPLMCreateFlightLoop_t)` for forward compatibility
- **phase**: When to execute (before/after flight model)
- **callbackFunc**: Your callback function pointer
- **refcon**: Optional reference data passed to callback

## Core Functions

### Timing and Synchronization

#### XPLMGetElapsedTime

```c
XPLM_API float XPLMGetElapsedTime(void);
```

Returns elapsed wall time since simulator startup in seconds.

**Characteristics**:

- Continues counting during pause
- Low precision (not suitable for high-precision timing)
- Adequate for general scheduling and logging

**Warning**: Not suitable for timing-critical applications like network synchronization.

**Example**:

```c
static float startTime = 0.0f;

float MyCallback(float elapsed, float total, int counter, void *ref) {
    if (startTime == 0.0f) {
        startTime = XPLMGetElapsedTime();
    }
  
    float sessionTime = XPLMGetElapsedTime() - startTime;
    // Log session duration
  
    return 1.0f; // Next callback in 1 second
}
```

#### XPLMGetCycleNumber

```c
XPLM_API int XPLMGetCycleNumber(void);
```

Returns monotonic counter starting at zero, incremented for each simulation cycle.

**Use Cases**:

- Frame-rate independent scheduling
- Synchronization between callbacks
- Performance monitoring

**Example**:

```c
static int lastCycle = 0;

float MyCallback(float elapsed, float total, int counter, void *ref) {
    int currentCycle = XPLMGetCycleNumber();
    int cyclesSinceLastCall = currentCycle - lastCycle;
    lastCycle = currentCycle;
  
    // Process based on elapsed cycles
    return -10.0f; // Next callback in 10 cycles
}
```

### Legacy Flight Loop API

#### XPLMRegisterFlightLoopCallback

```c
XPLM_API void XPLMRegisterFlightLoopCallback(
    XPLMFlightLoop_f inFlightLoop,
    float            inInterval,
    void             *inRefcon
);
```

Registers a legacy flight loop callback (pre-flight-model phase only).

**Parameters**:

- **inFlightLoop**: Callback function pointer
- **inInterval**: Initial scheduling interval
  - Positive: Seconds from registration to first callback
  - Negative: Cycles from registration to first callback
  - Zero: Inactive (registered but not scheduled)
- **inRefcon**: Reference constant passed to callback

**Limitations**:

- Only supports pre-flight-model execution
- Less flexible than modern API
- Maintained for backward compatibility

**Example**:

```c
float LoggingCallback(float elapsed, float total, int counter, void *ref) {
    // Log aircraft data every second
    float altitude = XPLMGetDataf(altitudeRef);
    LogData(altitude);
    return 1.0f; // Next callback in 1 second
}

void RegisterLogging() {
    XPLMRegisterFlightLoopCallback(LoggingCallback, 1.0f, NULL);
}
```

#### XPLMUnregisterFlightLoopCallback

```c
XPLM_API void XPLMUnregisterFlightLoopCallback(
    XPLMFlightLoop_f inFlightLoop,
    void             *inRefcon
);
```

Unregisters a legacy flight loop callback.

**Important Notes**:

- Never call from within your callback function
- Use callback return value of 0 to deactivate instead
- Only use with callbacks registered via `XPLMRegisterFlightLoopCallback`

**Example**:

```c
void CleanupLogging() {
    XPLMUnregisterFlightLoopCallback(LoggingCallback, NULL);
}
```

#### XPLMSetFlightLoopCallbackInterval

```c
XPLM_API void XPLMSetFlightLoopCallbackInterval(
    XPLMFlightLoop_f inFlightLoop,
    float            inInterval,
    int              inRelativeToNow,
    void             *inRefcon
);
```

Changes the scheduling interval for a legacy callback.

**Parameters**:

- **inFlightLoop**: Callback function to modify
- **inInterval**: New interval (same format as registration)
- **inRelativeToNow**:
  - 1: Time relative to this call
  - 0: Time relative to last callback execution
- **inRefcon**: Reference constant for identification

**Important**: Never call from within your callback - use return value instead.

### Modern Flight Loop API (XPLM210+)

#### XPLMCreateFlightLoop

```c
XPLM_API XPLMFlightLoopID XPLMCreateFlightLoop(
    XPLMCreateFlightLoop_t *inParams
);
```

Creates a modern flight loop callback with enhanced control.

**Advantages**:

- Supports both pre and post-flight-model phases
- Unique ID-based management
- Enhanced scheduling control
- Better performance tracking

**Example**:

```c
XPLMFlightLoopID gPreFlightCallback = NULL;
XPLMFlightLoopID gPostFlightCallback = NULL;

float PreFlightProcessor(float elapsed, float total, int counter, void *ref) {
    // Setup conditions before flight model
    return 0.0167f; // ~60 FPS
}

float PostFlightProcessor(float elapsed, float total, int counter, void *ref) {
    // Process results after flight model
    return 0.0167f; // ~60 FPS
}

void CreateCallbacks() {
    // Pre-flight model callback
    XPLMCreateFlightLoop_t preParams = {
        .structSize = sizeof(XPLMCreateFlightLoop_t),
        .phase = xplm_FlightLoop_Phase_BeforeFlightModel,
        .callbackFunc = PreFlightProcessor,
        .refcon = NULL
    };
    gPreFlightCallback = XPLMCreateFlightLoop(&preParams);
  
    // Post-flight model callback
    XPLMCreateFlightLoop_t postParams = {
        .structSize = sizeof(XPLMCreateFlightLoop_t),
        .phase = xplm_FlightLoop_Phase_AfterFlightModel,
        .callbackFunc = PostFlightProcessor,
        .refcon = NULL
    };
    gPostFlightCallback = XPLMCreateFlightLoop(&postParams);
  
    // Schedule both callbacks
    XPLMScheduleFlightLoop(gPreFlightCallback, 0.0167f, 1);
    XPLMScheduleFlightLoop(gPostFlightCallback, 0.0167f, 1);
}
```

#### XPLMDestroyFlightLoop

```c
XPLM_API void XPLMDestroyFlightLoop(
    XPLMFlightLoopID inFlightLoopID
);
```

Destroys a modern flight loop callback and releases resources.

**Important**:

- Only use with callbacks created via `XPLMCreateFlightLoop`
- Automatically unschedules the callback
- Safe to call even if callback is currently scheduled

**Example**:

```c
void CleanupCallbacks() {
    if (gPreFlightCallback) {
        XPLMDestroyFlightLoop(gPreFlightCallback);
        gPreFlightCallback = NULL;
    }
    if (gPostFlightCallback) {
        XPLMDestroyFlightLoop(gPostFlightCallback);
        gPostFlightCallback = NULL;
    }
}
```

#### XPLMScheduleFlightLoop

```c
XPLM_API void XPLMScheduleFlightLoop(
    XPLMFlightLoopID inFlightLoopID,
    float            inInterval,
    int              inRelativeToNow
);
```

Schedules or reschedules a modern flight loop callback.

**Parameters**:

- **inFlightLoopID**: Callback to schedule
- **inInterval**: Scheduling interval
  - Positive: Duration in seconds
  - Negative: Number of flight loop cycles
- **inRelativeToNow**:
  - 1: Time relative to this call
  - 0: Time relative to last callback execution

**Use Cases**:

- Initial activation of created callbacks
- Dynamic scheduling changes
- Reactivating suspended callbacks

**Example - Adaptive Scheduling**:

```c
float AdaptiveCallback(float elapsed, float total, int counter, void *ref) {
    static int highFrequencyMode = 0;
  
    // Do processing...
  
    if (NeedsHighFrequency()) {
        if (!highFrequencyMode) {
            highFrequencyMode = 1;
            // Switch to high frequency externally
            return 0.01f; // 100 Hz
        }
    } else {
        if (highFrequencyMode) {
            highFrequencyMode = 0;
            return 0.1f; // 10 Hz
        }
    }
  
    return highFrequencyMode ? 0.01f : 0.1f;
}
```

## Implementation Guidelines

### Best Practices

#### 1. Callback Performance

```c
float OptimizedCallback(float elapsed, float total, int counter, void *ref) {
    // Use static variables to maintain state
    static int processCounter = 0;
    static float accumulatedTime = 0.0f;
  
    // Spread expensive operations across multiple calls
    processCounter++;
    accumulatedTime += elapsed;
  
    if (processCounter >= 10) { // Every 10 calls
        DoExpensiveOperation();
        processCounter = 0;
    }
  
    // Quick operations every callback
    DoQuickUpdate();
  
    return 0.0167f; // ~60 FPS
}
```

#### 2. Resource Management

```c
typedef struct {
    XPLMFlightLoopID callbackID;
    XPLMDataRef dataRefs[10];
    int isActive;
} FlightLoopContext;

FlightLoopContext* CreateFlightLoopSystem() {
    FlightLoopContext* ctx = malloc(sizeof(FlightLoopContext));
  
    // Initialize datarefs once
    ctx->dataRefs[0] = XPLMFindDataRef("sim/aircraft/position/altitude");
    // ... initialize other datarefs
  
    // Create callback
    XPLMCreateFlightLoop_t params = {
        .structSize = sizeof(XPLMCreateFlightLoop_t),
        .phase = xplm_FlightLoop_Phase_AfterFlightModel,
        .callbackFunc = SystemCallback,
        .refcon = ctx
    };
    ctx->callbackID = XPLMCreateFlightLoop(&params);
    ctx->isActive = 0;
  
    return ctx;
}

void DestroyFlightLoopSystem(FlightLoopContext* ctx) {
    if (ctx->callbackID) {
        XPLMDestroyFlightLoop(ctx->callbackID);
    }
    free(ctx);
}
```

#### 3. Error Handling

```c
float SafeCallback(float elapsed, float total, int counter, void *ref) {
    try {
        // Your processing code
        return 1.0f; // Continue normally
    } catch (...) {
        XPLMDebugString("Flight loop callback error - suspending\n");
        return 0.0f; // Suspend on error
    }
}
```

### Common Use Cases

#### 1. Data Logging System

```c
typedef struct {
    FILE* logFile;
    float lastLogTime;
    float logInterval;
} LoggingContext;

float DataLogger(float elapsed, float total, int counter, void *ref) {
    LoggingContext* ctx = (LoggingContext*)ref;
    float currentTime = XPLMGetElapsedTime();
  
    if (currentTime - ctx->lastLogTime >= ctx->logInterval) {
        // Read and log data
        float altitude = XPLMGetDataf(altitudeDataRef);
        float speed = XPLMGetDataf(speedDataRef);
      
        fprintf(ctx->logFile, "%.2f,%.2f,%.2f\n", currentTime, altitude, speed);
        fflush(ctx->logFile);
      
        ctx->lastLogTime = currentTime;
    }
  
    return ctx->logInterval; // Log at specified interval
}
```

#### 2. System Monitor

```c
float SystemMonitor(float elapsed, float total, int counter, void *ref) {
    static float lastUpdate = 0.0f;
    float currentTime = XPLMGetElapsedTime();
  
    // Check critical systems every 0.1 seconds
    if (currentTime - lastUpdate >= 0.1f) {
        float fuelLevel = XPLMGetDataf(fuelDataRef);
        float engineTemp = XPLMGetDataf(tempDataRef);
      
        // Check for critical conditions
        if (fuelLevel < 0.05f) { // 5% fuel
            TriggerLowFuelWarning();
        }
      
        if (engineTemp > 800.0f) { // Over temp
            TriggerOverheatWarning();
        }
      
        lastUpdate = currentTime;
    }
  
    return 0.05f; // Check every 50ms for responsiveness
}
```

#### 3. Physics Integration

```c
float PhysicsIntegrator(float elapsed, float total, int counter, void *ref) {
    // Read current state
    float altitude = XPLMGetDataf(altitudeRef);
    float velocity = XPLMGetDataf(velocityRef);
  
    // Apply custom physics
    float customForce = CalculateCustomForce(altitude, velocity);
  
    // Modify flight model (if dataref is writable)
    XPLMSetDataf(customForceRef, customForce);
  
    return -1.0f; // Every flight loop for smooth integration
}
```

#### 4. Conditional Processing

```c
float ConditionalProcessor(float elapsed, float total, int counter, void *ref) {
    static int isInFlight = 0;
    float groundSpeed = XPLMGetDataf(groundSpeedRef);
  
    // Detect flight state changes
    int wasInFlight = isInFlight;
    isInFlight = (groundSpeed > 5.0f); // 5 m/s threshold
  
    if (isInFlight && !wasInFlight) {
        // Just took off
        OnTakeoff();
        return 0.1f; // Higher frequency during flight
    } else if (!isInFlight && wasInFlight) {
        // Just landed
        OnLanding();
        return 1.0f; // Lower frequency on ground
    }
  
    if (isInFlight) {
        ProcessFlightData();
        return 0.1f; // 10 Hz during flight
    } else {
        ProcessGroundData();
        return 1.0f; // 1 Hz on ground
    }
}
```

## Performance Optimization

### Scheduling Strategies

1. **Variable Frequency**: Adjust callback frequency based on flight phase
2. **Distributed Processing**: Spread expensive operations across multiple callbacks
3. **Conditional Execution**: Skip processing when not needed
4. **Batch Operations**: Process multiple items per callback

### Memory Management

- Pre-allocate data structures during plugin initialization
- Use object pools for frequently created/destroyed objects
- Cache dataref handles - don't look up every callback
- Release resources during plugin shutdown

### Threading Considerations

- Flight loop callbacks execute on the main thread
- Don't perform blocking operations (file I/O, network)
- Use separate threads for heavy computation, synchronize carefully
- Consider using background processing with result polling

## Common Pitfalls and Solutions

### Problem: Drawing in Flight Loop Callbacks

**Wrong**:

```c
float BadCallback(float elapsed, float total, int counter, void *ref) {
    // DON'T DO THIS - Drawing not allowed!
    XPLMDrawString(0, 100, 100, "Hello", NULL, xplmFont_Basic);
    return 1.0f;
}
```

**Correct**:

```c
// Use drawing callbacks for graphics
void MyDrawCallback(XPLMWindowID window, void* refcon) {
    XPLMDrawString(0, 100, 100, "Hello", NULL, xplmFont_Basic);
}

float GoodCallback(float elapsed, float total, int counter, void *ref) {
    // Update data that drawing callback will use
    UpdateDisplayData();
    return 1.0f;
}
```

### Problem: Calling Schedule Functions from Callback

**Wrong**:

```c
float BadCallback(float elapsed, float total, int counter, void *ref) {
    // DON'T DO THIS - Use return value instead
    XPLMSetFlightLoopCallbackInterval(BadCallback, 0.5f, 1, ref);
    return 1.0f;
}
```

**Correct**:

```c
float GoodCallback(float elapsed, float total, int counter, void *ref) {
    // Use return value to control scheduling
    if (NeedsSlowUpdate()) {
        return 2.0f; // Slow down
    } else {
        return 0.1f; // Speed up
    }
}
```

### Problem: Blocking Operations

**Wrong**:

```c
float BlockingCallback(float elapsed, float total, int counter, void *ref) {
    // DON'T DO THIS - Blocks the sim
    FILE* file = fopen("data.txt", "w");
    for (int i = 0; i < 1000000; i++) {
        fprintf(file, "line %d\n", i);
    }
    fclose(file);
    return 1.0f;
}
```

**Correct**:

```c
float NonBlockingCallback(float elapsed, float total, int counter, void *ref) {
    static int writeIndex = 0;
    static FILE* file = NULL;
  
    // Open file once
    if (!file) {
        file = fopen("data.txt", "w");
        writeIndex = 0;
    }
  
    // Write small amount per callback
    if (file && writeIndex < 1000000) {
        for (int i = 0; i < 100; i++) { // Only 100 lines per callback
            fprintf(file, "line %d\n", writeIndex++);
            if (writeIndex >= 1000000) break;
        }
      
        if (writeIndex >= 1000000) {
            fclose(file);
            file = NULL;
        }
    }
  
    return 0.05f; // 20 Hz
}
```

## Integration with Other APIs

### Data Access Integration

```c
float DataIntegratedCallback(float elapsed, float total, int counter, void *ref) {
    // Read multiple datarefs efficiently
    static XPLMDataRef refs[5] = {NULL};
    if (!refs[0]) {
        refs[0] = XPLMFindDataRef("sim/aircraft/position/altitude");
        refs[1] = XPLMFindDataRef("sim/aircraft/position/latitude");
        refs[2] = XPLMFindDataRef("sim/aircraft/position/longitude");
        refs[3] = XPLMFindDataRef("sim/aircraft/position/groundspeed");
        refs[4] = XPLMFindDataRef("sim/aircraft/position/heading");
    }
  
    float values[5];
    values[0] = XPLMGetDataf(refs[0]); // altitude
    values[1] = XPLMGetDataf(refs[1]); // latitude
    values[2] = XPLMGetDataf(refs[2]); // longitude
    values[3] = XPLMGetDataf(refs[3]); // groundspeed
    values[4] = XPLMGetDataf(refs[4]); // heading
  
    // Process the data
    ProcessFlightData(values);
  
    return 0.1f; // 10 Hz
}
```

### Menu Integration

```c
void MenuHandler(void* menuRef, void* itemRef) {
    int itemIndex = (intptr_t)itemRef;
  
    if (itemIndex == 0) { // Start monitoring
        XPLMScheduleFlightLoop(gMonitoringCallback, 0.1f, 1);
    } else if (itemIndex == 1) { // Stop monitoring
        XPLMScheduleFlightLoop(gMonitoringCallback, 0.0f, 1); // Deactivate
    }
}
```

## Version Compatibility

- **XPLM200+**: Basic flight loop callbacks, terrain probing
- **XPLM210+**: Modern flight loop API with phase control and IDs
- **XPLM300+**: Enhanced timing and scheduling features
- **XPLM400+**: Additional performance optimizations

Always check version compatibility:

```c
#if defined(XPLM210)
    // Use modern API
    XPLMCreateFlightLoop(&params);
#else
    // Fall back to legacy API
    XPLMRegisterFlightLoopCallback(callback, interval, refcon);
#endif
```

## Summary

The XPLMProcessing API is for creating responsive, efficient X-Plane plugins. Key takeaways:

1. **Choose the Right API**: Use modern flight loop API for new development
2. **Performance First**: Keep callbacks fast and suspend when inactive
3. **Proper Scheduling**: Use appropriate intervals for different operations
4. **Resource Management**: Initialize once, clean up properly
5. **Integration**: Coordinate with other X-Plane APIs for full functionality
