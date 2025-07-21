# XPLMPlugin API Documentation

## Overview

The XPLMPlugin API provides essential functionality for plugin discovery, management, inter-plugin communication, and system feature control. This API enables plugins to find and interact with other plugins, manage plugin states, send messages between plugins, and enable advanced X-Plane features.

## Key Features

- **Plugin Discovery**: Find and enumerate loaded plugins by ID, path, or signature
- **Plugin State Management**: Enable, disable, and reload plugins programmatically
- **Inter-Plugin Messaging**: Send custom messages and respond to system notifications
- **Feature Management**: Enable advanced X-Plane features for enhanced plugin capabilities
- **System Integration**: Handle X-Plane lifecycle events and system changes

## Architecture

### Plugin Identification

Plugins are identified using multiple methods:

1. **Plugin ID (XPLMPluginID)**: Unique integer assigned at runtime
2. **Signature**: Persistent string identifier (recommended for inter-plugin communication)
3. **File Path**: Absolute path to plugin file
4. **Name**: Human-readable plugin name

### Plugin Lifecycle

Plugins go through defined states:

1. **Loaded**: Plugin DLL loaded into memory
2. **Started**: XPluginStart() called successfully
3. **Enabled**: XPluginEnable() called, plugin is active
4. **Disabled**: XPluginDisable() called, plugin inactive but loaded
5. **Stopped**: XPluginStop() called, plugin preparing for unload

### Messaging System

The plugin messaging system supports two types of communication:

- **Commands**: Directed messages to induce specific behavior (< 0x80000000)
- **Notifications**: Broadcast messages for informational purposes (≥ 0x80000000)

## Data Types

### XPLMPluginID

```c
typedef int XPLMPluginID;

#define XPLM_NO_PLUGIN_ID    (-1)  // Invalid/no plugin
#define XPLM_PLUGIN_XPLANE   (0)   // X-Plane itself
```

Unique identifier for each loaded plugin. Values change between X-Plane sessions.

## Plugin Discovery Functions

### XPLMGetMyID

```c
XPLM_API XPLMPluginID XPLMGetMyID(void);
```

Returns the plugin ID of the calling plugin.

**Returns**: Current plugin's unique ID

**Usage:**

```c
static XPLMPluginID gMyPluginID = XPLM_NO_PLUGIN_ID;

PLUGIN_API int XPluginStart(char* name, char* sig, char* desc) {
    gMyPluginID = XPLMGetMyID();
  
    char message[256];
    snprintf(message, sizeof(message), "My plugin ID is: %d\n", gMyPluginID);
    XPLMDebugString(message);
  
    return 1;
}
```

### XPLMCountPlugins

```c
XPLM_API int XPLMCountPlugins(void);
```

Returns the total number of loaded plugins (both enabled and disabled).

**Returns**: Total plugin count including disabled plugins

### XPLMGetNthPlugin

```c
XPLM_API XPLMPluginID XPLMGetNthPlugin(int inIndex);
```

Returns the plugin ID for the Nth plugin in the system.

**Parameters:**

- `inIndex`: Zero-based plugin index (0 to XPLMCountPlugins()-1)

**Returns**: Plugin ID or XPLM_NO_PLUGIN_ID if invalid index

**Note**: Plugin order is arbitrary and may change between calls.

**Example - Plugin Enumeration:**

```c
void ListAllPlugins() {
    int pluginCount = XPLMCountPlugins();
    XPLMDebugString("=== Loaded Plugins ===\n");
  
    for (int i = 0; i < pluginCount; i++) {
        XPLMPluginID pluginID = XPLMGetNthPlugin(i);
        if (pluginID != XPLM_NO_PLUGIN_ID) {
            char name[256], signature[256], description[256];
          
            XPLMGetPluginInfo(pluginID, name, NULL, signature, description);
          
            int enabled = XPLMIsPluginEnabled(pluginID);
          
            printf("Plugin %d: %s (%s) - %s\n", 
                   pluginID, name, signature, enabled ? "ENABLED" : "DISABLED");
            printf("  Description: %s\n", description);
        }
    }
}
```

### XPLMFindPluginByPath

```c
XPLM_API XPLMPluginID XPLMFindPluginByPath(const char * inPath);
```

Finds a plugin by its file system path.

**Parameters:**

- `inPath`: Absolute path to plugin file

**Returns**: Plugin ID or XPLM_NO_PLUGIN_ID if not found

**Usage:**

```c
XPLMPluginID FindMyBuddyPlugin() {
    // Try to find a related plugin
    char systemPath[512];
    XPLMGetSystemPath(systemPath);
  
    char buddyPath[1024];
    snprintf(buddyPath, sizeof(buddyPath), 
             "%sResources%splugins%sBuddyPlugin%sBuddyPlugin.xpl",
             systemPath, XPLMGetDirectorySeparator(),
             XPLMGetDirectorySeparator(), XPLMGetDirectorySeparator());
  
    XPLMPluginID buddy = XPLMFindPluginByPath(buddyPath);
    if (buddy != XPLM_NO_PLUGIN_ID) {
        XPLMDebugString("Found buddy plugin!\n");
        return buddy;
    }
  
    XPLMDebugString("Buddy plugin not found\n");
    return XPLM_NO_PLUGIN_ID;
}
```

### XPLMFindPluginBySignature

```c
XPLM_API XPLMPluginID XPLMFindPluginBySignature(const char * inSignature);
```

Finds a plugin by its unique signature string.

**Parameters:**

- `inSignature`: Plugin signature (e.g., "com.example.myplugin")

**Returns**: Plugin ID or XPLM_NO_PLUGIN_ID if not found

**Best Practice**: Use signatures for reliable inter-plugin communication as they're independent of file paths and human-readable names.

**Example:**

```c
// Check for autopilot plugin
XPLMPluginID autopilot = XPLMFindPluginBySignature("com.example.autopilot");
if (autopilot != XPLM_NO_PLUGIN_ID && XPLMIsPluginEnabled(autopilot)) {
    // Communicate with autopilot
    XPLMSendMessageToPlugin(autopilot, MSG_ENGAGE_AUTOPILOT, NULL);
}
```

### XPLMGetPluginInfo

```c
XPLM_API void XPLMGetPluginInfo(
    XPLMPluginID inPlugin,      // Plugin to query
    char *       outName,       // Plugin name (256+ chars, can be NULL)
    char *       outFilePath,   // Plugin file path (256+ chars, can be NULL)  
    char *       outSignature,  // Plugin signature (256+ chars, can be NULL)
    char *       outDescription // Plugin description (256+ chars, can be NULL)
);
```

Retrieves comprehensive information about a plugin.

**Parameters:**

- `inPlugin`: Plugin ID to query
- `outName`: Buffer for human-readable plugin name (or NULL)
- `outFilePath`: Buffer for absolute file path (or NULL)
- `outSignature`: Buffer for unique signature (or NULL)
- `outDescription`: Buffer for plugin description (or NULL)

**Buffer Requirements**: All buffers should be at least 256 characters.

**Example - Plugin Inspector:**

```c
void InspectPlugin(XPLMPluginID pluginID) {
    char name[256], path[512], signature[256], description[512];
  
    XPLMGetPluginInfo(pluginID, name, path, signature, description);
  
    printf("=== Plugin Information ===\n");
    printf("ID: %d\n", pluginID);
    printf("Name: %s\n", name);
    printf("Signature: %s\n", signature);
    printf("Path: %s\n", path);
    printf("Description: %s\n", description);
    printf("Enabled: %s\n", XPLMIsPluginEnabled(pluginID) ? "Yes" : "No");
}
```

## Plugin State Management

### XPLMIsPluginEnabled

```c
XPLM_API int XPLMIsPluginEnabled(XPLMPluginID inPluginID);
```

Checks if a plugin is currently enabled and running.

**Parameters:**

- `inPluginID`: Plugin ID to check

**Returns**: 1 if enabled, 0 if disabled or invalid ID

### XPLMEnablePlugin

```c
XPLM_API int XPLMEnablePlugin(XPLMPluginID inPluginID);
```

Attempts to enable a disabled plugin.

**Parameters:**

- `inPluginID`: Plugin ID to enable

**Returns**: 1 if successfully enabled, 0 if failed

**Notes:**

- Plugin must be loaded and started
- Plugin's XPluginEnable() function must return success
- Some plugins may fail to enable due to resource constraints

**Example:**

```c
int EnableRequiredPlugin(const char* signature) {
    XPLMPluginID plugin = XPLMFindPluginBySignature(signature);
  
    if (plugin == XPLM_NO_PLUGIN_ID) {
        XPLMDebugString("Required plugin not found\n");
        return 0;
    }
  
    if (XPLMIsPluginEnabled(plugin)) {
        XPLMDebugString("Required plugin already enabled\n");
        return 1;
    }
  
    if (XPLMEnablePlugin(plugin)) {
        XPLMDebugString("Successfully enabled required plugin\n");
        return 1;
    } else {
        XPLMDebugString("Failed to enable required plugin\n");
        return 0;
    }
}
```

### XPLMDisablePlugin

```c
XPLM_API void XPLMDisablePlugin(XPLMPluginID inPluginID);
```

Disables an enabled plugin.

**Parameters:**

- `inPluginID`: Plugin ID to disable

**Notes:**

- Plugin's XPluginDisable() function will be called
- Plugin remains loaded but becomes inactive
- Cannot disable your own plugin

### XPLMReloadPlugins

```c
XPLM_API void XPLMReloadPlugins(void);
```

Reloads all plugins in the system.

**Behavior:**

1. All plugins receive XPluginDisable() calls
2. All plugins receive XPluginStop() calls
3. Plugin DLLs are unloaded from memory
4. Complete plugin system restart occurs
5. Plugins are reloaded and restarted

**Warning**: Your plugin will be unloaded after this call returns!

**Usage:**

```c
void OnReloadRequested() {
    XPLMDebugString("Reloading all plugins...\n");
  
    // Save any important state before reload
    SavePluginState();
  
    // Trigger reload - we won't return from this normally
    XPLMReloadPlugins();
}
```

## Inter-Plugin Messaging

### System Messages

X-Plane sends these predefined messages to all plugins:

#### XPLM_MSG_PLANE_CRASHED (101)

```c
#define XPLM_MSG_PLANE_CRASHED 101
```

Sent when the user's aircraft crashes. Parameter is ignored.

#### XPLM_MSG_PLANE_LOADED (102)

```c
#define XPLM_MSG_PLANE_LOADED 102
```

Sent when a new aircraft is loaded. Parameter contains aircraft index (0 = user).

#### XPLM_MSG_AIRPORT_LOADED (103)

```c
#define XPLM_MSG_AIRPORT_LOADED 103
```

Sent when user's aircraft is positioned at a new airport. Parameter ignored.

#### XPLM_MSG_SCENERY_LOADED (104)

```c
#define XPLM_MSG_SCENERY_LOADED 104
```

Sent when new scenery is loaded. Parameter ignored.

#### XPLM_MSG_AIRPLANE_COUNT_CHANGED (105)

```c
#define XPLM_MSG_AIRPLANE_COUNT_CHANGED 105
```

Sent when user adjusts aircraft count. Parameter ignored.

#### XPLM_MSG_PLANE_UNLOADED (106) - XPLM200+

```c
#define XPLM_MSG_PLANE_UNLOADED 106
```

Sent when aircraft is unloaded. Parameter contains aircraft index.

#### XPLM_MSG_WILL_WRITE_PREFS (107) - XPLM210+

```c
#define XPLM_MSG_WILL_WRITE_PREFS 107
```

Sent before X-Plane writes preferences. Use to reset temporary dataref changes.

#### XPLM_MSG_LIVERY_LOADED (108) - XPLM210+

```c
#define XPLM_MSG_LIVERY_LOADED 108
```

Sent after livery loads for aircraft. Parameter contains aircraft index.

#### XPLM_MSG_ENTERED_VR (109) - XPLM301+

```c
#define XPLM_MSG_ENTERED_VR 109
```

Sent before entering VR mode. Parameter ignored.

#### XPLM_MSG_EXITING_VR (110) - XPLM301+

```c
#define XPLM_MSG_EXITING_VR 110
```

Sent before leaving VR mode. Parameter ignored.

#### XPLM_MSG_RELEASE_PLANES (111) - XPLM303+

```c
#define XPLM_MSG_RELEASE_PLANES 111
```

Sent when another plugin requests aircraft control. Sender is plugin ID requesting control.

#### XPLM_MSG_FMOD_BANK_LOADED (112) - XPLM400+

```c
#define XPLM_MSG_FMOD_BANK_LOADED 112
```

Sent after FMOD sound banks load. Parameter is XPLMBankID (0=master, 1=radio).

#### XPLM_MSG_FMOD_BANK_UNLOADING (113) - XPLM400+

```c
#define XPLM_MSG_FMOD_BANK_UNLOADING 113
```

Sent before FMOD sound banks unload. Parameter is XPLMBankID.

#### XPLM_MSG_DATAREFS_ADDED (114) - XPLM400+

```c
#define XPLM_MSG_DATAREFS_ADDED 114
```

Sent when datarefs are added (requires XPLM_WANTS_DATAREF_NOTIFICATIONS feature). Parameter contains new total count.

### Message Handling Example

```c
PLUGIN_API void XPluginReceiveMessage(XPLMPluginID inFromWho, int inMessage, void* inParam) {
    switch (inMessage) {
        case XPLM_MSG_PLANE_LOADED: {
            int aircraftIndex = (int)(intptr_t)inParam;
            printf("Aircraft loaded at index %d\n", aircraftIndex);
          
            if (aircraftIndex == 0) {
                // User aircraft changed
                OnUserAircraftLoaded();
                RefreshAircraftSpecificFeatures();
            }
            break;
        }
      
        case XPLM_MSG_AIRPORT_LOADED:
            XPLMDebugString("User positioned at new airport\n");
            UpdateNavigationDatabase();
            break;
          
        case XPLM_MSG_SCENERY_LOADED:
            XPLMDebugString("New scenery loaded\n");
            RefreshSceneryObjects();
            break;
          
        case XPLM_MSG_WILL_WRITE_PREFS:
            XPLMDebugString("Preparing for preferences write\n");
            RestoreDefaultDatarefs();
            break;
          
        case XPLM_MSG_ENTERED_VR:
            XPLMDebugString("Entering VR mode\n");
            AdaptUIForVR();
            break;
          
        case XPLM_MSG_EXITING_VR:
            XPLMDebugString("Exiting VR mode\n");
            RestoreNormalUI();
            break;
          
        case XPLM_MSG_RELEASE_PLANES:
            HandlePlaneReleaseRequest(inFromWho);
            break;
          
        default:
            // Handle custom messages
            if (inMessage >= 0x8000000) {
                HandleCustomNotification(inFromWho, inMessage, inParam);
            } else {
                HandleCustomCommand(inFromWho, inMessage, inParam);
            }
            break;
    }
}
```

### XPLMSendMessageToPlugin

```c
XPLM_API void XPLMSendMessageToPlugin(
    XPLMPluginID inPlugin,  // Target plugin ID (or XPLM_NO_PLUGIN_ID for broadcast)
    int          inMessage, // Message ID
    void *       inParam    // Message parameter
);
```

Sends a message to another plugin or broadcasts to all plugins.

**Parameters:**

- `inPlugin`: Target plugin ID, or XPLM_NO_PLUGIN_ID to broadcast to all
- `inMessage`: Message identifier (custom messages should follow conventions)
- `inParam`: Message-specific parameter (can be NULL)

**Message Conventions:**

- **Commands**: Values < 0x80000000
- **Notifications**: Values ≥ 0x80000000

**Custom Message Example:**

```c
// Define custom messages (in shared header)
#define AUTOPILOT_MSG_ENGAGE     0x01000001  // Command
#define AUTOPILOT_MSG_DISENGAGE  0x01000002  // Command  
#define AUTOPILOT_NOTIFY_ENGAGED 0x81000001  // Notification
#define AUTOPILOT_NOTIFY_DISENGAGED 0x81000002 // Notification

// Send command to specific autopilot plugin
void EngageAutopilot() {
    XPLMPluginID autopilot = XPLMFindPluginBySignature("com.example.autopilot");
    if (autopilot != XPLM_NO_PLUGIN_ID) {
        XPLMSendMessageToPlugin(autopilot, AUTOPILOT_MSG_ENGAGE, NULL);
    }
}

// Broadcast notification to all plugins
void NotifyAutopilotEngaged() {
    XPLMSendMessageToPlugin(XPLM_NO_PLUGIN_ID, AUTOPILOT_NOTIFY_ENGAGED, NULL);
}

// In autopilot plugin's message handler
PLUGIN_API void XPluginReceiveMessage(XPLMPluginID from, int msg, void* param) {
    switch (msg) {
        case AUTOPILOT_MSG_ENGAGE:
            DoEngageAutopilot();
            // Notify all plugins
            XPLMSendMessageToPlugin(XPLM_NO_PLUGIN_ID, AUTOPILOT_NOTIFY_ENGAGED, NULL);
            break;
          
        case AUTOPILOT_MSG_DISENGAGE:
            DoDisengageAutopilot();
            XPLMSendMessageToPlugin(XPLM_NO_PLUGIN_ID, AUTOPILOT_NOTIFY_DISENGAGED, NULL);
            break;
    }
}
```

## Feature Management (XPLM200+)

### Feature System Overview

The feature system allows plugins to enable advanced X-Plane capabilities that are normally disabled for backward compatibility or performance reasons.

### Standard Features

#### XPLM_WANTS_REFLECTIONS

```c
"XPLM_WANTS_REFLECTIONS"
```

Enables drawing callbacks during reflection and shadow rendering passes.

- Check `sim/graphics/view/plane_render_type` dataref to determine rendering context
- Simplify or skip rendering for reflections
- Skip non-solid drawing for shadows

#### XPLM_USE_NATIVE_PATHS

```c
"XPLM_USE_NATIVE_PATHS"
```

Enables Unix-style paths on all platforms:

- **macOS**: Native Unix paths
- **Windows**: Forward slashes with drive letters (C:/)
- **Linux**: Native paths
- **Recommendation**: All plugins should enable this

#### XPLM_USE_NATIVE_WIDGET_WINDOWS

```c
"XPLM_USE_NATIVE_WIDGET_WINDOWS"
```

Enables modern widget windows with:

- High-DPI support
- UI scaling compliance
- VR mode compatibility
- Modern window backing

#### XPLM_WANTS_DATAREF_NOTIFICATIONS

```c
"XPLM_WANTS_DATAREF_NOTIFICATIONS"
```

Enables XPLM_MSG_DATAREFS_ADDED messages when new datarefs are registered.

### Feature Management Functions

#### XPLMHasFeature

```c
XPLM_API int XPLMHasFeature(const char * inFeature);
```

Checks if X-Plane supports a specific feature.

**Parameters:**

- `inFeature`: Feature name string

**Returns**: 1 if supported, 0 if not supported

#### XPLMIsFeatureEnabled

```c
XPLM_API int XPLMIsFeatureEnabled(const char * inFeature);
```

Checks if a feature is currently enabled for your plugin.

**Parameters:**

- `inFeature`: Feature name string

**Returns**: 1 if enabled, 0 if disabled

**Error**: Calling with unsupported feature is an error

#### XPLMEnableFeature

```c
XPLM_API void XPLMEnableFeature(const char * inFeature, int inEnable);
```

Enables or disables a feature for your plugin.

**Parameters:**

- `inFeature`: Feature name string
- `inEnable`: 1 to enable, 0 to disable

**Usage Example:**

```c
PLUGIN_API int XPluginStart(char* name, char* sig, char* desc) {
    // Enable modern features
    if (XPLMHasFeature("XPLM_USE_NATIVE_PATHS")) {
        XPLMEnableFeature("XPLM_USE_NATIVE_PATHS", 1);
        XPLMDebugString("Enabled native paths\n");
    }
  
    if (XPLMHasFeature("XPLM_USE_NATIVE_WIDGET_WINDOWS")) {
        XPLMEnableFeature("XPLM_USE_NATIVE_WIDGET_WINDOWS", 1);  
        XPLMDebugString("Enabled native widget windows\n");
    }
  
    if (XPLMHasFeature("XPLM_WANTS_DATAREF_NOTIFICATIONS")) {
        XPLMEnableFeature("XPLM_WANTS_DATAREF_NOTIFICATIONS", 1);
        XPLMDebugString("Enabled dataref notifications\n");
    }
  
    return 1;
}
```

### XPLMFeatureEnumerator_f

```c
typedef void (* XPLMFeatureEnumerator_f)(
    const char * inFeature,
    void *       inRef
);
```

Callback function type for feature enumeration.

#### XPLMEnumerateFeatures

```c
XPLM_API void XPLMEnumerateFeatures(
    XPLMFeatureEnumerator_f inEnumerator,
    void *                  inRef
);
```

Enumerates all features supported by the current X-Plane version.

**Parameters:**

- `inEnumerator`: Callback function called for each feature
- `inRef`: User data passed to callback

**Example:**

```c
void FeatureCallback(const char* feature, void* ref) {
    printf("Available feature: %s\n", feature);
  
    // Enable all modern features
    if (strstr(feature, "XPLM_USE_NATIVE") != NULL) {
        XPLMEnableFeature(feature, 1);
        printf("  -> Enabled\n");
    }
}

void DiscoverAndEnableFeatures() {
    XPLMDebugString("=== Available X-Plane Features ===\n");
    XPLMEnumerateFeatures(FeatureCallback, NULL);
}
```

## Implementation Patterns

### Plugin Dependency System

```c
typedef struct {
    const char* signature;
    const char* name;
    int required;
    int minVersion;
} PluginDependency;

static PluginDependency gDependencies[] = {
    {"com.laminar.xplane.autopilot", "X-Plane Autopilot", 1, 0},
    {"com.example.weather", "Weather Plugin", 0, 2},
    {"com.example.sounds", "Enhanced Sounds", 0, 1}
};

int CheckDependencies() {
    int allRequired = 1;
  
    for (int i = 0; i < sizeof(gDependencies) / sizeof(gDependencies[0]); i++) {
        XPLMPluginID plugin = XPLMFindPluginBySignature(gDependencies[i].signature);
      
        if (plugin == XPLM_NO_PLUGIN_ID) {
            if (gDependencies[i].required) {
                printf("ERROR: Required plugin not found: %s\n", gDependencies[i].name);
                allRequired = 0;
            } else {
                printf("Optional plugin not found: %s\n", gDependencies[i].name);
            }
        } else {
            if (!XPLMIsPluginEnabled(plugin)) {
                if (gDependencies[i].required) {
                    printf("ERROR: Required plugin disabled: %s\n", gDependencies[i].name);
                    allRequired = 0;
                } else {
                    printf("Optional plugin disabled: %s\n", gDependencies[i].name);
                }
            } else {
                printf("Dependency satisfied: %s\n", gDependencies[i].name);
            }
        }
    }
  
    return allRequired;
}
```

### Plugin Communication Protocol

```c
// Shared protocol definitions (in common header)
#define PROTOCOL_VERSION 1

typedef struct {
    int version;
    int messageType;
    int dataLength;
} ProtocolHeader;

#define MSG_HANDSHAKE     0x01000001
#define MSG_DATA_REQUEST  0x01000002  
#define MSG_DATA_RESPONSE 0x01000003
#define NOTIFY_STATUS     0x81000001

// Send structured message
void SendProtocolMessage(XPLMPluginID target, int msgType, void* data, int dataLen) {
    // Allocate buffer for header + data
    int totalSize = sizeof(ProtocolHeader) + dataLen;
    char* buffer = malloc(totalSize);
  
    ProtocolHeader* header = (ProtocolHeader*)buffer;
    header->version = PROTOCOL_VERSION;
    header->messageType = msgType;
    header->dataLength = dataLen;
  
    if (data && dataLen > 0) {
        memcpy(buffer + sizeof(ProtocolHeader), data, dataLen);
    }
  
    XPLMSendMessageToPlugin(target, msgType, buffer);
    free(buffer);
}

// Handle structured message
void HandleProtocolMessage(XPLMPluginID sender, int message, void* param) {
    if (!param) return;
  
    ProtocolHeader* header = (ProtocolHeader*)param;
    void* data = (char*)param + sizeof(ProtocolHeader);
  
    if (header->version != PROTOCOL_VERSION) {
        XPLMDebugString("Protocol version mismatch\n");
        return;
    }
  
    switch (header->messageType) {
        case MSG_HANDSHAKE:
            HandleHandshake(sender, data, header->dataLength);
            break;
        case MSG_DATA_REQUEST:
            HandleDataRequest(sender, data, header->dataLength);
            break;
        // ... handle other message types
    }
}
```

### System Event Handler

```c
typedef struct {
    void (*onPlaneLoaded)(int aircraftIndex);
    void (*onAirportLoaded)(void);
    void (*onSceneryLoaded)(void);
    void (*onVRModeChanged)(int entering);
    void (*onPreferencesWrite)(void);
} SystemEventCallbacks;

static SystemEventCallbacks gEventCallbacks = {0};

void RegisterSystemEventCallbacks(SystemEventCallbacks* callbacks) {
    gEventCallbacks = *callbacks;
}

PLUGIN_API void XPluginReceiveMessage(XPLMPluginID from, int msg, void* param) {
    switch (msg) {
        case XPLM_MSG_PLANE_LOADED:
            if (gEventCallbacks.onPlaneLoaded) {
                gEventCallbacks.onPlaneLoaded((int)(intptr_t)param);
            }
            break;
          
        case XPLM_MSG_AIRPORT_LOADED:
            if (gEventCallbacks.onAirportLoaded) {
                gEventCallbacks.onAirportLoaded();
            }
            break;
          
        case XPLM_MSG_SCENERY_LOADED:
            if (gEventCallbacks.onSceneryLoaded) {
                gEventCallbacks.onSceneryLoaded();
            }
            break;
          
        case XPLM_MSG_ENTERED_VR:
            if (gEventCallbacks.onVRModeChanged) {
                gEventCallbacks.onVRModeChanged(1);
            }
            break;
          
        case XPLM_MSG_EXITING_VR:
            if (gEventCallbacks.onVRModeChanged) {
                gEventCallbacks.onVRModeChanged(0);
            }
            break;
          
        case XPLM_MSG_WILL_WRITE_PREFS:
            if (gEventCallbacks.onPreferencesWrite) {
                gEventCallbacks.onPreferencesWrite();
            }
            break;
    }
}

// Usage
void OnPlaneLoadedHandler(int index) {
    printf("Aircraft loaded at index %d\n", index);
    if (index == 0) {
        RefreshUserAircraftData();
    }
}

void OnVRModeChangedHandler(int entering) {
    if (entering) {
        AdaptForVR();
    } else {
        RestoreDesktopMode();
    }
}

void InitializeEventHandling() {
    SystemEventCallbacks callbacks = {
        .onPlaneLoaded = OnPlaneLoadedHandler,
        .onVRModeChanged = OnVRModeChangedHandler,
        // ... other callbacks
    };
  
    RegisterSystemEventCallbacks(&callbacks);
}
```

## Best Practices

### Plugin Startup Sequence

```c
PLUGIN_API int XPluginStart(char* name, char* sig, char* desc) {
    // 1. Set plugin info
    strcpy(name, "My Advanced Plugin");
    strcpy(sig, "com.example.advanced");
    strcpy(desc, "Advanced plugin with dependencies");
  
    // 2. Enable modern features first
    EnableModernFeatures();
  
    // 3. Get plugin ID
    gMyPluginID = XPLMGetMyID();
  
    // 4. Check dependencies
    if (!CheckDependencies()) {
        XPLMDebugString("Dependency check failed\n");
        return 0;
    }
  
    // 5. Initialize core systems
    InitializeCore();
  
    return 1;
}

void EnableModernFeatures() {
    const char* features[] = {
        "XPLM_USE_NATIVE_PATHS",
        "XPLM_USE_NATIVE_WIDGET_WINDOWS", 
        "XPLM_WANTS_DATAREF_NOTIFICATIONS",
        NULL
    };
  
    for (int i = 0; features[i]; i++) {
        if (XPLMHasFeature(features[i])) {
            XPLMEnableFeature(features[i], 1);
            printf("Enabled feature: %s\n", features[i]);
        }
    }
}
```

### Error Handling

```c
int SafeGetPluginInfo(XPLMPluginID plugin, char* name, int nameSize, 
                     char* sig, int sigSize) {
    if (plugin == XPLM_NO_PLUGIN_ID) {
        XPLMDebugString("Invalid plugin ID\n");
        return 0;
    }
  
    // Initialize buffers
    if (name) memset(name, 0, nameSize);
    if (sig) memset(sig, 0, sigSize);
  
    // Get info with size-limited buffers
    char tempName[256], tempSig[256];
    XPLMGetPluginInfo(plugin, tempName, NULL, tempSig, NULL);
  
    // Copy with bounds checking
    if (name && nameSize > 0) {
        strncpy(name, tempName, nameSize - 1);
        name[nameSize - 1] = '\0';
    }
  
    if (sig && sigSize > 0) {
        strncpy(sig, tempSig, sigSize - 1);
        sig[sigSize - 1] = '\0';
    }
  
    return 1;
}
```

### Plugin Manager

```c
typedef struct PluginInfo {
    XPLMPluginID id;
    char name[128];
    char signature[128]; 
    char description[256];
    int enabled;
    struct PluginInfo* next;
} PluginInfo;

PluginInfo* BuildPluginList() {
    PluginInfo* head = NULL;
    PluginInfo* tail = NULL;
  
    int count = XPLMCountPlugins();
  
    for (int i = 0; i < count; i++) {
        XPLMPluginID id = XPLMGetNthPlugin(i);
        if (id == XPLM_NO_PLUGIN_ID) continue;
      
        PluginInfo* info = malloc(sizeof(PluginInfo));
        memset(info, 0, sizeof(PluginInfo));
      
        info->id = id;
        XPLMGetPluginInfo(id, info->name, NULL, info->signature, info->description);
        info->enabled = XPLMIsPluginEnabled(id);
        info->next = NULL;
      
        if (head == NULL) {
            head = tail = info;
        } else {
            tail->next = info;
            tail = info;
        }
    }
  
    return head;
}

void FreePluginList(PluginInfo* head) {
    while (head) {
        PluginInfo* next = head->next;
        free(head);
        head = next;
    }
}
```

## Version Compatibility

- **Base Plugin Functions**: All SDK versions
- **Plugin Management**: XPLM200+
- **Feature System**: XPLM200+
- **VR Messages**: XPLM301+
- **Plane Release**: XPLM303+
- **FMOD Messages**: XPLM400+
- **Dataref Notifications**: XPLM400+

## Integration Examples

### Autopilot Plugin Communication

```c
// Autopilot interface
typedef struct {
    float targetHeading;
    float targetAltitude;
    float targetSpeed;
    int mode;  // 0=off, 1=heading, 2=nav, 3=approach
} AutopilotState;

static XPLMPluginID gAutopilotPlugin = XPLM_NO_PLUGIN_ID;

int InitAutopilotInterface() {
    gAutopilotPlugin = XPLMFindPluginBySignature("com.example.autopilot");
  
    if (gAutopilotPlugin != XPLM_NO_PLUGIN_ID && 
        XPLMIsPluginEnabled(gAutopilotPlugin)) {
        XPLMDebugString("Autopilot plugin found and enabled\n");
        return 1;
    }
  
    XPLMDebugString("Autopilot plugin not available\n");
    return 0;
}

void SetAutopilotHeading(float heading) {
    if (gAutopilotPlugin == XPLM_NO_PLUGIN_ID) return;
  
    AutopilotState state = {
        .targetHeading = heading,
        .mode = 1
    };
  
    XPLMSendMessageToPlugin(gAutopilotPlugin, MSG_SET_AUTOPILOT_STATE, &state);
}
```
