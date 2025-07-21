# XPCListener - Observer Pattern Listener Implementation

## Overview

The `XPCListener` class represents the observer side of the Observer pattern implementation, working in conjunction with `XPCBroadcaster`. This abstract base class provides automatic lifetime management and maintains bidirectional relationships with broadcasters for robust event handling in X-Plane plugins.

## Files

- **Header**: `CHeaders/Wrappers/XPCListener.h`
- **Implementation**: `CHeaders/Wrappers/XPCListener.cpp`
- **Dependencies**: `XPCBroadcaster.h`, `<vector>`, `<algorithm>`

## Class Declaration

```cpp
class XPCListener {
public:
    XPCListener();
    virtual ~XPCListener();
  
    virtual void ListenToMessage(int inMessage, void* inParam) = 0;
  
private:
    typedef std::vector<XPCBroadcaster*> BroadcastVector;
    BroadcastVector mBroadcasters;
  
    friend class XPCBroadcaster;
  
    void BroadcasterAdded(XPCBroadcaster* inBroadcaster);
    void BroadcasterRemoved(XPCBroadcaster* inBroadcaster);
};
```

## Constructor and Destructor

### XPCListener()

Initializes an empty listener with no broadcaster relationships.

**Usage**:

```cpp
class FlightDataListener : public XPCListener {
public:
    void ListenToMessage(int inMessage, void* inParam) override {
        // Implementation
    }
};

FlightDataListener listener;
```

### ~XPCListener()

**Critical Cleanup Process**: The destructor ensures proper cleanup by:

1. Iterating through all registered broadcasters
2. Calling `RemoveListener(this)` on each broadcaster
3. Using a while loop that continues until `mBroadcasters` is empty
4. Each `RemoveListener()` call triggers `BroadcasterRemoved()` which updates the vector

**Implementation Note**: The while loop pattern handles the case where the vector is modified during iteration as each removal updates the container.

## Abstract Interface

### ListenToMessage(int inMessage, void* inParam) = 0

Pure virtual method that derived classes must implement to handle broadcast messages.

**Parameters**:

- `inMessage`: Integer identifier for the message type
- `inParam`: Pointer to message-specific data (may be nullptr)

**Implementation Examples**:

```cpp
class AircraftSystemListener : public XPCListener {
public:
    void ListenToMessage(int inMessage, void* inParam) override {
        switch(inMessage) {
            case ALTITUDE_CHANGED: {
                float* altitude = static_cast<float*>(inParam);
                HandleAltitudeChange(*altitude);
                break;
            }
          
            case ENGINE_STATE_CHANGED: {
                EngineData* data = static_cast<EngineData*>(inParam);
                HandleEngineStateChange(*data);
                break;
            }
          
            case GEAR_DEPLOYED: {
                bool* deployed = static_cast<bool*>(inParam);
                HandleGearDeployment(*deployed);
                break;
            }
          
            default:
                // Unknown message - ignore or log
                break;
        }
    }

private:
    void HandleAltitudeChange(float newAltitude);
    void HandleEngineStateChange(const EngineData& data);
    void HandleGearDeployment(bool deployed);
};
```

## Private Interface (Friend Access)

### BroadcasterAdded(XPCBroadcaster* inBroadcaster)

Called by `XPCBroadcaster::AddListener()` to establish bidirectional relationship.

**Behavior**:

- Adds the broadcaster to the internal `mBroadcasters` vector
- Maintains the list for cleanup purposes
- Called automatically - not for direct use

### BroadcasterRemoved(XPCBroadcaster* inBroadcaster)

Called by `XPCBroadcaster::RemoveListener()` or broadcaster destructor to break relationship.

**Behavior**:

- Searches for the broadcaster in `mBroadcasters` using `std::find`
- Removes the broadcaster if found
- Safe to call even if broadcaster not in list
- Called automatically - not for direct use

## Implementation Patterns

### 1. Multi-Source Listening

Listen to multiple broadcasters with message type discrimination:

```cpp
class FlightDataAggregator : public XPCListener {
private:
    enum MessageSources {
        ENGINE_MESSAGES = 1000,
        NAVIGATION_MESSAGES = 2000,
        WEATHER_MESSAGES = 3000
    };

public:
    void SetupListeners() {
        engineBroadcaster->AddListener(this);
        navigationBroadcaster->AddListener(this);
        weatherBroadcaster->AddListener(this);
    }
  
    void ListenToMessage(int inMessage, void* inParam) override {
        if (inMessage >= ENGINE_MESSAGES && inMessage < NAVIGATION_MESSAGES) {
            HandleEngineMessage(inMessage, inParam);
        }
        else if (inMessage >= NAVIGATION_MESSAGES && inMessage < WEATHER_MESSAGES) {
            HandleNavigationMessage(inMessage, inParam);
        }
        else if (inMessage >= WEATHER_MESSAGES) {
            HandleWeatherMessage(inMessage, inParam);
        }
    }
};
```

### 2. Conditional Processing

Implement smart filtering and conditional processing:

```cpp
class ConditionalListener : public XPCListener {
private:
    bool isEnabled;
    std::set<int> interestedMessages;
  
public:
    ConditionalListener() : isEnabled(true) {
        interestedMessages.insert(ALTITUDE_CHANGED);
        interestedMessages.insert(SPEED_CHANGED);
    }
  
    void SetEnabled(bool enabled) { isEnabled = enabled; }
  
    void AddInterest(int messageType) {
        interestedMessages.insert(messageType);
    }
  
    void ListenToMessage(int inMessage, void* inParam) override {
        if (!isEnabled) return;
      
        if (interestedMessages.find(inMessage) == interestedMessages.end()) {
            return; // Not interested in this message
        }
      
        ProcessMessage(inMessage, inParam);
    }
};
```

### 3. State-Based Handling

Maintain internal state and respond accordingly:

```cpp
class StatefulListener : public XPCListener {
private:
    enum FlightPhase { GROUND, TAXI, TAKEOFF, CRUISE, APPROACH, LANDING };
    FlightPhase currentPhase;
  
public:
    void ListenToMessage(int inMessage, void* inParam) override {
        switch(inMessage) {
            case ALTITUDE_CHANGED: {
                float altitude = *static_cast<float*>(inParam);
                UpdateFlightPhase(altitude);
                HandleAltitudeForPhase(altitude);
                break;
            }
          
            case GEAR_STATE_CHANGED: {
                bool deployed = *static_cast<bool*>(inParam);
                if (deployed && currentPhase == CRUISE) {
                    currentPhase = APPROACH;
                    OnPhaseChange(APPROACH);
                }
                break;
            }
        }
    }
  
private:
    void UpdateFlightPhase(float altitude);
    void HandleAltitudeForPhase(float altitude);
    void OnPhaseChange(FlightPhase newPhase);
};
```

## X-Plane Integration Examples

### 1. UI Update Listener

Connect flight data changes to UI updates:

```cpp
class UIUpdateListener : public XPCListener {
private:
    XPLMWindowID window;
    bool needsRedraw;
  
public:
    void ListenToMessage(int inMessage, void* inParam) override {
        switch(inMessage) {
            case FLIGHT_DATA_UPDATED:
                needsRedraw = true;
                break;
              
            case UI_SETTINGS_CHANGED:
                RefreshUILayout();
                needsRedraw = true;
                break;
        }
      
        if (needsRedraw && XPLMGetWindowIsVisible(window)) {
            // Trigger window redraw - X-Plane will call draw callback
        }
    }
  
    // Called from X-Plane draw callback
    void DrawWindow() {
        if (needsRedraw) {
            RenderUpdatedData();
            needsRedraw = false;
        }
    }
};
```

### 2. Dataref Publishing Listener

Update X-Plane datarefs based on plugin events:

```cpp
class DatarefPublisher : public XPCListener {
private:
    XPLMDataRef customAltitudeRef;
    XPLMDataRef customSpeedRef;
  
public:
    DatarefPublisher() {
        // Register custom datarefs
        customAltitudeRef = XPLMRegisterDataAccessor(
            "myplugin/custom/altitude",
            xplmType_Float, 1,
            nullptr, nullptr,
            GetAltitudeCallback, SetAltitudeCallback,
            nullptr, nullptr, nullptr, nullptr, nullptr, nullptr,
            this
        );
    }
  
    void ListenToMessage(int inMessage, void* inParam) override {
        switch(inMessage) {
            case CUSTOM_ALTITUDE_CALCULATED: {
                float* altitude = static_cast<float*>(inParam);
                customAltitude = *altitude;
                break;
            }
          
            case CUSTOM_SPEED_CALCULATED: {
                float* speed = static_cast<float*>(inParam);
                customSpeed = *speed;
                break;
            }
        }
    }
  
private:
    float customAltitude, customSpeed;
  
    static float GetAltitudeCallback(void* refcon) {
        return static_cast<DatarefPublisher*>(refcon)->customAltitude;
    }
  
    static void SetAltitudeCallback(void* refcon, float value) {
        static_cast<DatarefPublisher*>(refcon)->customAltitude = value;
    }
};
```

### 3. Command Response Listener

Execute X-Plane commands based on plugin events:

```cpp
class CommandExecutor : public XPCListener {
private:
    XPLMCommandRef gearToggleCmd;
    XPLMCommandRef lightToggleCmd;
  
public:
    CommandExecutor() {
        gearToggleCmd = XPLMFindCommand("sim/flight_controls/landing_gear_toggle");
        lightToggleCmd = XPLMFindCommand("sim/lights/landing_lights_toggle");
    }
  
    void ListenToMessage(int inMessage, void* inParam) override {
        switch(inMessage) {
            case AUTO_GEAR_TRIGGER: {
                float* altitude = static_cast<float*>(inParam);
                if (*altitude < 1000.0f) {  // Below 1000 feet
                    XPLMCommandOnce(gearToggleCmd);
                }
                break;
            }
          
            case AUTO_LIGHTS_TRIGGER: {
                bool* isDark = static_cast<bool*>(inParam);
                if (*isDark) {
                    XPLMCommandOnce(lightToggleCmd);
                }
                break;
            }
        }
    }
};
```

## Error Handling and Debugging

### 1. Safe Message Processing

Implement error handling for robust operation:

```cpp
class SafeListener : public XPCListener {
public:
    void ListenToMessage(int inMessage, void* inParam) override {
        try {
            ProcessMessageSafe(inMessage, inParam);
        }
        catch (const std::exception& e) {
            XPLMDebugString(("Listener error: " + std::string(e.what()) + "\n").c_str());
        }
        catch (...) {
            XPLMDebugString("Unknown listener error\n");
        }
    }
  
private:
    void ProcessMessageSafe(int inMessage, void* inParam) {
        // Validate parameters
        if (inMessage < 0) {
            throw std::invalid_argument("Invalid message type");
        }
      
        // Type-safe parameter access
        switch(inMessage) {
            case FLOAT_MESSAGE:
                if (!inParam) throw std::invalid_argument("Null parameter for float message");
                ProcessFloatMessage(*static_cast<float*>(inParam));
                break;
        }
    }
};
```

### 2. Debug Tracing

Add debugging capabilities for development:

```cpp
class TracingListener : public XPCListener {
private:
    bool debugEnabled;
    std::string listenerName;
  
public:
    TracingListener(const std::string& name) 
        : listenerName(name), debugEnabled(false) {}
  
    void SetDebugEnabled(bool enabled) { debugEnabled = enabled; }
  
    void ListenToMessage(int inMessage, void* inParam) override {
        if (debugEnabled) {
            char debugMsg[256];
            snprintf(debugMsg, sizeof(debugMsg), 
                     "[%s] Received message %d, param: %p\n",
                     listenerName.c_str(), inMessage, inParam);
            XPLMDebugString(debugMsg);
        }
      
        ProcessMessage(inMessage, inParam);
    }
  
private:
    void ProcessMessage(int inMessage, void* inParam);
};
```

## Memory Management Guidelines

### 1. Parameter Lifetime

Understand parameter ownership and lifetime:

```cpp
class MemoryAwareListener : public XPCListener {
public:
    void ListenToMessage(int inMessage, void* inParam) override {
        switch(inMessage) {
            case STRING_MESSAGE: {
                // Copy string data immediately - don't store pointer
                const char* str = static_cast<const char*>(inParam);
                std::string safeCopy(str);
                ProcessStringMessage(safeCopy);
                break;
            }
          
            case TEMPORARY_DATA: {
                // Process immediately - don't store pointer
                TempData* data = static_cast<TempData*>(inParam);
                ProcessTempData(*data);  // Dereference and copy
                break;
            }
          
            case PERSISTENT_DATA: {
                // Safe to store pointer if documented as persistent
                PersistentData* data = static_cast<PersistentData*>(inParam);
                persistentDataPtr = data;
                break;
            }
        }
    }
};
```

### 2. Resource Cleanup

Implement proper resource management:

```cpp
class ResourceManagingListener : public XPCListener {
private:
    std::vector<std::unique_ptr<ProcessedData>> dataQueue;
  
public:
    void ListenToMessage(int inMessage, void* inParam) override {
        if (inMessage == DATA_RECEIVED) {
            // Create managed resource
            auto processedData = std::make_unique<ProcessedData>();
            processedData->Initialize(static_cast<RawData*>(inParam));
          
            dataQueue.push_back(std::move(processedData));
          
            // Limit queue size
            if (dataQueue.size() > MAX_QUEUE_SIZE) {
                dataQueue.erase(dataQueue.begin());
            }
        }
    }
  
    ~ResourceManagingListener() {
        // Automatic cleanup via unique_ptr
    }
};
```

## Performance Considerations

### 1. Fast Message Filtering

Optimize for common cases:

```cpp
class OptimizedListener : public XPCListener {
private:
    std::unordered_set<int> interestedMessages;
  
public:
    void ListenToMessage(int inMessage, void* inParam) override {
        // Fast hash lookup for filtering
        if (interestedMessages.find(inMessage) == interestedMessages.end()) {
            return;
        }
      
        ProcessMessage(inMessage, inParam);
    }
};
```

### 2. Deferred Processing

Defer expensive operations:

```cpp
class DeferredProcessor : public XPCListener {
private:
    struct PendingMessage {
        int message;
        std::shared_ptr<void> data;
        std::chrono::steady_clock::time_point timestamp;
    };
  
    std::queue<PendingMessage> pendingMessages;
  
public:
    void ListenToMessage(int inMessage, void* inParam) override {
        // Store for later processing
        PendingMessage pending;
        pending.message = inMessage;
        pending.timestamp = std::chrono::steady_clock::now();
      
        // Copy parameter data if needed
        if (RequiresDataCopy(inMessage)) {
            pending.data = CopyParameterData(inMessage, inParam);
        }
      
        pendingMessages.push(pending);
    }
  
    // Call from flight loop or timer
    void ProcessPendingMessages() {
        while (!pendingMessages.empty()) {
            auto& pending = pendingMessages.front();
            ProcessDeferredMessage(pending);
            pendingMessages.pop();
        }
    }
};
```

## Best Practices

### 1. Message Type Organization

Use clear, organized message type definitions:

```cpp
class WellOrganizedListener : public XPCListener {
public:
    // Use scoped enums for type safety
    enum class SystemMessages : int {
        EngineStart = 1000,
        EngineStop = 1001,
        GearExtend = 1100,
        GearRetract = 1101
    };
  
    void ListenToMessage(int inMessage, void* inParam) override {
        auto message = static_cast<SystemMessages>(inMessage);
      
        switch(message) {
            case SystemMessages::EngineStart:
                HandleEngineStart(inParam);
                break;
            case SystemMessages::GearExtend:
                HandleGearExtend(inParam);
                break;
        }
    }
};
```

### 2. Interface Design

Design clear listener interfaces:

```cpp
// Abstract base for specific listener types
class FlightDataListener : public XPCListener {
public:
    void ListenToMessage(int inMessage, void* inParam) final override {
        // Type-safe dispatch
        switch(inMessage) {
            case ALTITUDE_MSG:
                OnAltitudeChanged(*static_cast<float*>(inParam));
                break;
            case SPEED_MSG:
                OnSpeedChanged(*static_cast<float*>(inParam));
                break;
        }
    }

protected:
    virtual void OnAltitudeChanged(float altitude) = 0;
    virtual void OnSpeedChanged(float speed) = 0;
};

// Concrete implementation
class ConcreteFlightListener : public FlightDataListener {
protected:
    void OnAltitudeChanged(float altitude) override {
        // Implementation
    }
  
    void OnSpeedChanged(float speed) override {
        // Implementation
    }
};
```
