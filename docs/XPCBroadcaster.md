# XPCBroadcaster - Observer Pattern Implementation

## Overview

The `XPCBroadcaster` class provides a robust implementation of the Observer pattern (also known as the Publisher-Subscriber pattern) for X-Plane plugin development. This C++ wrapper class manages a collection of listeners and provides thread-safe message broadcasting capabilities with reentrancy protection.

## Class Hierarchy

```
XPCBroadcaster (Publisher/Subject)
    ↓ manages
XPCListener (Observer)
```

## Files

- **Header**: `CHeaders/Wrappers/XPCBroadcaster.h`
- **Implementation**: `CHeaders/Wrappers/XPCBroadcaster.cpp`
- **Dependencies**: `XPCListener.h`, `<vector>`, `<algorithm>`

## Class Declaration

```cpp
class XPCBroadcaster {
public:
    XPCBroadcaster();
    virtual ~XPCBroadcaster();
  
    void AddListener(XPCListener* inListener);
    void RemoveListener(XPCListener* inListener);

protected:
    void BroadcastMessage(int inMessage, void* inParam = 0);

private:
    typedef std::vector<XPCListener*> ListenerVector;
    ListenerVector mListeners;
    ListenerVector::iterator* mIterator;  // Reentrancy support
};
```

## Constructor and Destructor

### XPCBroadcaster()

Initializes a new broadcaster instance with an empty listener list and null iterator.

**Usage**:

```cpp
class MyEventSource : public XPCBroadcaster {
    // Implementation
};

MyEventSource eventSource;
```

### ~XPCBroadcaster()

**Critical Cleanup Process**: The destructor performs essential cleanup by:

1. Iterating through all registered listeners
2. Notifying each listener that the broadcaster is being removed via `BroadcasterRemoved(this)`
3. Using reentrancy-safe iteration with `mIterator` tracking

**Implementation Details**:

- Creates a local iterator and assigns it to `mIterator` for reentrancy protection
- Calls `BroadcasterRemoved()` on each listener to maintain bidirectional relationship integrity
- Listeners automatically remove themselves from their internal broadcaster lists

## Public Methods

### AddListener(XPCListener* inListener)

Registers a listener to receive broadcast messages from this broadcaster.

**Parameters**:

- `inListener`: Pointer to the listener object to register

**Behavior**:

1. Adds the listener to the internal vector
2. Calls `inListener->BroadcasterAdded(this)` to establish bidirectional relationship
3. The listener maintains its own list of broadcasters for cleanup purposes

**Usage**:

```cpp
class MyListener : public XPCListener {
public:
    void ListenToMessage(int inMessage, void* inParam) override {
        switch(inMessage) {
            case ALTITUDE_CHANGED:
                HandleAltitudeChange(static_cast<float*>(inParam));
                break;
            case SPEED_CHANGED:
                HandleSpeedChange(static_cast<float*>(inParam));
                break;
        }
    }
};

MyEventSource broadcaster;
MyListener listener;
broadcaster.AddListener(&listener);
```

### RemoveListener(XPCListener* inListener)

Unregisters a listener from receiving broadcast messages.

**Parameters**:

- `inListener`: Pointer to the listener object to unregister

**Behavior**:

1. Searches for the listener in the internal vector using `std::find`
2. If not found, returns immediately (safe to call multiple times)
3. **Reentrancy Protection**: If currently broadcasting, adjusts the active iterator to prevent skipping elements
4. Removes the listener from the vector
5. Calls `inListener->BroadcasterRemoved(this)` to break bidirectional relationship

**Reentrancy Safety**: The method handles the case where a listener removes itself or other listeners during message processing by adjusting the iterator position.

**Usage**:

```cpp
broadcaster.RemoveListener(&listener);
```

## Protected Methods

### BroadcastMessage(int inMessage, void* inParam = 0)

Sends a message to all registered listeners. This method is protected, meaning only derived classes can initiate broadcasts.

**Parameters**:

- `inMessage`: Integer identifier for the message type (user-defined constants)
- `inParam`: Optional pointer to message-specific data (defaults to nullptr)

**Reentrancy Protection**:

- Sets `mIterator` to point to the current iterator during broadcast
- This allows `RemoveListener()` to adjust the iterator if modifications occur during iteration
- Resets `mIterator` to `nullptr` after broadcasting completes

**Usage in Derived Classes**:

```cpp
class FlightDataBroadcaster : public XPCBroadcaster {
private:
    enum Messages {
        ALTITUDE_CHANGED = 1,
        SPEED_CHANGED = 2,
        HEADING_CHANGED = 3
    };
  
public:
    void UpdateAltitude(float newAltitude) {
        // Update internal state
        currentAltitude = newAltitude;
      
        // Notify all listeners
        BroadcastMessage(ALTITUDE_CHANGED, &currentAltitude);
    }
  
    void UpdateSpeed(float newSpeed) {
        currentSpeed = newSpeed;
        BroadcastMessage(SPEED_CHANGED, &currentSpeed);
    }
};
```

## Implementation Guidelines

### 1. Message Type Definitions

Define meaningful message constants for type safety and code clarity:

```cpp
class AircraftSystemsBroadcaster : public XPCBroadcaster {
public:
    // Message type constants
    enum SystemMessages {
        ENGINE_START = 100,
        GEAR_DEPLOYED = 101,
        FLAPS_CHANGED = 102,
        AUTOPILOT_ENGAGED = 103
    };
  
    void NotifyEngineStart(int engineNumber) {
        BroadcastMessage(ENGINE_START, &engineNumber);
    }
};
```

### 2. Data Passing Conventions

Use structured data for complex messages:

```cpp
struct GearStateData {
    bool isDeployed;
    float deploymentPercent;
    int gearIndex;
};

void NotifyGearChange(const GearStateData& gearData) {
    // Create a copy for thread safety
    GearStateData* data = new GearStateData(gearData);
    BroadcastMessage(GEAR_DEPLOYED, data);
    // Note: Listeners are responsible for cleanup if dynamic allocation is used
}
```

### 3. X-Plane Integration Patterns

#### Flight Loop Integration

```cpp
class FlightDataManager : public XPCBroadcaster {
private:
    XPLMDataRef altitudeRef;
    XPLMDataRef speedRef;
    float lastAltitude, lastSpeed;
  
public:
    FlightDataManager() {
        altitudeRef = XPLMFindDataRef("sim/cockpit2/gauges/indicators/altitude_ft_pilot");
        speedRef = XPLMFindDataRef("sim/cockpit2/gauges/indicators/airspeed_kts_pilot");
    }
  
    // Called from flight loop callback
    void UpdateFlightData() {
        float currentAlt = XPLMGetDataf(altitudeRef);
        float currentSpeed = XPLMGetDataf(speedRef);
      
        if (abs(currentAlt - lastAltitude) > 10.0f) {
            lastAltitude = currentAlt;
            BroadcastMessage(ALTITUDE_CHANGED, &lastAltitude);
        }
      
        if (abs(currentSpeed - lastSpeed) > 5.0f) {
            lastSpeed = currentSpeed;
            BroadcastMessage(SPEED_CHANGED, &lastSpeed);
        }
    }
};
```

#### Command Handler Integration

```cpp
class CommandBroadcaster : public XPCBroadcaster {
public:
    enum CommandMessages {
        CUSTOM_COMMAND_EXECUTED = 200,
        SYSTEM_COMMAND_INTERCEPTED = 201
    };
  
    // Command handler callback
    static int CommandHandler(XPLMCommandRef cmd, XPLMCommandPhase phase, void* refcon) {
        CommandBroadcaster* broadcaster = static_cast<CommandBroadcaster*>(refcon);
      
        if (phase == xplm_CommandBegin) {
            broadcaster->BroadcastMessage(CUSTOM_COMMAND_EXECUTED, cmd);
        }
      
        return 1; // Allow other handlers to process
    }
};
```

## Use Cases

### 1. Plugin Communication

Facilitate communication between different components of a complex plugin:

```cpp
// Main plugin class
class WeatherRadarPlugin : public XPCBroadcaster {
    enum Messages { WEATHER_UPDATED = 1, DISPLAY_MODE_CHANGED = 2 };
  
public:
    void UpdateWeatherData(const WeatherData& data) {
        BroadcastMessage(WEATHER_UPDATED, const_cast<WeatherData*>(&data));
    }
};

// UI component
class RadarDisplay : public XPCListener {
public:
    void ListenToMessage(int message, void* param) override {
        if (message == WeatherRadarPlugin::WEATHER_UPDATED) {
            WeatherData* data = static_cast<WeatherData*>(param);
            RefreshDisplay(*data);
        }
    }
};
```

### 2. System State Monitoring

Monitor and broadcast X-Plane system state changes:

```cpp
class SystemMonitor : public XPCBroadcaster {
private:
    XPLMDataRef gearRef, flapRef, engineRef;
  
public:
    enum SystemEvents {
        GEAR_STATE_CHANGED = 10,
        FLAP_STATE_CHANGED = 11,
        ENGINE_STATE_CHANGED = 12
    };
  
    void CheckSystemStates() {
        // Check and broadcast state changes
        float gearPos = XPLMGetDataf(gearRef);
        if (gearPos != lastGearPos) {
            lastGearPos = gearPos;
            BroadcastMessage(GEAR_STATE_CHANGED, &gearPos);
        }
    }
};
```

### 3. Multi-Window Coordination

Coordinate multiple plugin windows:

```cpp
class WindowManager : public XPCBroadcaster {
public:
    enum WindowEvents {
        WINDOW_CLOSED = 50,
        WINDOW_MOVED = 51,
        FOCUS_CHANGED = 52
    };
  
    void OnWindowClose(int windowId) {
        BroadcastMessage(WINDOW_CLOSED, &windowId);
    }
};

class DependentWindow : public XPCListener {
public:
    void ListenToMessage(int message, void* param) override {
        if (message == WindowManager::WINDOW_CLOSED) {
            int closedId = *static_cast<int*>(param);
            if (closedId == parentWindowId) {
                CloseThisWindow();
            }
        }
    }
};
```

## Thread Safety Considerations

### Reentrancy Protection

The class provides protection against modification during iteration:

- `mIterator` tracks the current broadcast iteration
- `RemoveListener()` adjusts the iterator if removing elements during broadcast
- This prevents crashes from iterator invalidation

### Single-Thread Assumption

The implementation assumes single-threaded access typical in X-Plane plugins:

- No mutex protection for the listener vector
- Suitable for main thread callbacks (flight loops, draw callbacks, etc.)
- For multi-threaded scenarios, external synchronization is required

## Memory Management

### Listener Lifetime

- Listeners must outlive their registration with broadcasters
- The `XPCListener` destructor automatically removes itself from all broadcasters
- Broadcasters notify listeners when destroyed via `BroadcasterRemoved()`

### Message Parameter Lifetime

- Parameters passed to `BroadcastMessage()` must remain valid during the entire broadcast
- Consider copying data if it might be modified during listener processing
- Be careful with dynamic allocation - establish clear ownership rules

## Best Practices

### 1. Message Design

```cpp
// Good: Structured message types
enum FlightEvents {
    ENGINE_STARTED = 1000,    // Use distinctive number ranges
    ENGINE_STOPPED = 1001,
    GEAR_EXTENDED = 2000,
    GEAR_RETRACTED = 2001
};

// Better: Use namespaced constants or class enums
class FlightSystemBroadcaster : public XPCBroadcaster {
public:
    enum class Events : int {
        EngineStarted = 1000,
        EngineStopped = 1001,
        GearExtended = 2000,
        GearRetracted = 2001
    };
};
```

### 2. Error Handling

```cpp
class SafeBroadcaster : public XPCBroadcaster {
protected:
    void SafeBroadcastMessage(int message, void* param = nullptr) {
        try {
            BroadcastMessage(message, param);
        } catch (const std::exception& e) {
            XPLMDebugString(("Broadcast error: " + std::string(e.what()) + "\n").c_str());
        }
    }
};
```

### 3. Performance Optimization

```cpp
class OptimizedBroadcaster : public XPCBroadcaster {
private:
    bool hasListeners() const { return !mListeners.empty(); }
  
public:
    void ConditionalBroadcast(int message, void* param = nullptr) {
        // Avoid processing if no listeners
        if (hasListeners()) {
            BroadcastMessage(message, param);
        }
    }
};
```

## Integration with X-Plane Plugin Lifecycle

### Plugin Initialization

```cpp
PLUGIN_API int XPluginStart(char* outName, char* outSig, char* outDesc) {
    // Initialize broadcaster during plugin start
    gFlightDataBroadcaster = new FlightDataManager();
  
    // Initialize other systems that will listen
    gUIManager = new UIManager();
    gFlightDataBroadcaster->AddListener(gUIManager);
  
    return 1;
}
```

### Plugin Cleanup

```cpp
PLUGIN_API void XPluginStop(void) {
    // Broadcaster destructor automatically notifies listeners
    delete gFlightDataBroadcaster;
    delete gUIManager;
}
```
