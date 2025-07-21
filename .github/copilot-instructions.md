# X-Plane 12 Plugin SDK - AI Coding Instructions

This repository contains the official X-Plane 12 Plugin SDK for developing flight simulator plugins in C/C++ and Pascal/Delphi.

## Architecture Overview

**Plugin System**: X-Plane plugins are dynamic libraries (.xpl files) that implement 5 required entry point functions:
- `XPluginStart()` - Initialize plugin, register features, return plugin info
- `XPluginStop()` - Cleanup resources when plugin unloads
- `XPluginEnable()` - Enable plugin functionality (called after start)
- `XPluginDisable()` - Disable plugin functionality (called before stop)
- `XPluginReceiveMessage()` - Handle inter-plugin and system messages

**Dual Language Support**:
- `CHeaders/` - Complete C/C++ API headers with comprehensive documentation
- `Delphi/` - Pascal/Delphi interface files (direct translations of C headers)
- Both provide identical functionality; choose based on your language preference

**Core API Modules** (in `CHeaders/XPLM/`):
- `XPLMCamera.h` - Camera position control and custom views
- `XPLMDataAccess.h` - Read/write simulator data via datarefs (nav radios, aircraft state, etc.)
- `XPLMDefs.h` - Platform macros, export definitions, basic types
- `XPLMDisplay.h` - Window management, drawing callbacks, avionics device integration  
- `XPLMGraphics.h` - OpenGL state management, coordinate conversions, texture handling
- `XPLMInstance.h` - Modern 3D object instancing (replaces deprecated drawing callbacks)
- `XPLMMap.h` - Custom map layers with OpenGL drawing, icons, and labels
- `XPLMMenus.h` - Plugin menu creation and management
- `XPLMNavigation.h` - Navigation database access, FMS programming
- `XPLMPlanes.h` - Aircraft control, loading, multiplayer aircraft management
- `XPLMPlugin.h` - Plugin discovery, management, inter-plugin communication
- `XPLMProcessing.h` - Flight loop callbacks for periodic tasks
- `XPLMScenery.h` - Terrain probing, object placement, scenery library access
- `XPLMSound.h` - FMOD audio integration for custom sounds
- `XPLMUtilities.h` - Commands, keyboard input, file I/O, system info
- `XPLMWeather.h` - Weather system access and modification

## Platform-Specific Conventions

**Preprocessor Platform Macros** (must define exactly one):
- `APL=1` - macOS compilation target
- `IBM=1` - Windows compilation target  
- `LIN=1` - Linux compilation target

**Function Export Macros**:
- `PLUGIN_API` - Export your 5 required plugin callbacks
- `XPLM_API` - X-Plane SDK functions (for linking)

**File System Paths**: Enable `XPLM_USE_NATIVE_PATHS` feature for consistent Unix-style paths across all platforms (recommended).

## Data Access Patterns

**Dataref Workflow**:
1. Look up dataref by string path once during plugin startup: `XPLMFindDataRef("sim/cockpit/radios/nav1_freq_hz")`
2. Cache the opaque `XPLMDataRef` handle for the plugin's lifetime
3. Read/write data using typed functions: `XPLMGetDataf()`, `XPLMSetDatai()`, etc.
4. Check dataref type compatibility with `XPLMGetDataRefTypes()`

**Custom Datarefs**: Register read/write callbacks via `XPLMRegisterDataAccessor()` to publish your own data.

## Drawing & UI Architecture

**Modern Window System** (recommended):
- Use `XPLMCreateWindowEx()` with `XPLMCreateWindow_t` structure
- Supports high-DPI scaling, X-Plane 11 styling, VR mode
- Windows use "global desktop" coordinates, not panel coordinates
- Enable `XPLM_USE_NATIVE_WIDGET_WINDOWS` for modern widget integration

**Avionics Device Integration**:
- Register callbacks for built-in cockpit devices (GPS, PFD, etc.) via `XPLMRegisterAvionicsCallbacksEx()`
- Coordinates are in texels with origin at bottom-left
- Supports both 3D cockpit and popup window rendering

**Deprecated Drawing Callbacks**: Avoid direct rendering via `XPLMRegisterDrawCallback()` - use XPLMInstance API for 3D objects.

## Command System

**Command Registration**:
- Create commands: `XPLMCreateCommand("my_plugin/my_command", "Description")`
- Register handlers: `XPLMRegisterCommandHandler()` with begin/continue/end phase handling
- Commands have 3 phases: begin → continue (optional repeats) → end
- Always balance `XPLMCommandBegin()` and `XPLMCommandEnd()` calls

## Memory Management

**Plugin Lifetime**: 
- Look up datarefs, commands, and other opaque handles during `XPluginStart()`
- Register callbacks during `XPluginEnable()`
- Unregister callbacks during `XPluginDisable()`
- Clean up resources during `XPluginStop()`

**C++ Wrapper Classes**: `CHeaders/Wrappers/` provides C++ RAII wrappers for widgets and other objects.

## Build Configuration

**Link Libraries** (64-bit only):
- Windows: `Libraries/Win/XPLM_64.lib`, `Libraries/Win/XPWidgets_64.lib`
- macOS: `Libraries/Mac/XPLM.framework`, `Libraries/Mac/XPWidgets.framework`
- Linux: No link libraries needed (symbols resolved at runtime)

**Plugin Packaging** (modern format):
```
<plugin_name>/
  <abi>/
    <plugin_name>.xpl
```
Where `<abi>` is `win_x64`, `mac_x64`, or `lin_x64`.

## Common Patterns

**Flight Loop Integration**: Use `XPLMRegisterFlightLoopCallback()` for periodic tasks, NOT for drawing.

**Feature Detection**: Check X-Plane capabilities with `XPLMHasFeature()` and enable with `XPLMEnableFeature()`.

**Error Handling**: Install error callbacks via `XPLMSetErrorCallback()` for debugging - remove in production builds.

## Version Compatibility

SDK 4.1.1 targets X-Plane 12.1.0+. Use `#if defined(XPLM410)` preprocessor guards for version-specific APIs.

## Implementation Examples

### Basic Plugin Structure
```c
#include "XPLMPlugin.h"
#include "XPLMDataAccess.h"
#include "XPLMProcessing.h"
#include "XPLMUtilities.h"

// Define platform
#define IBM 1  // Windows

// Global variables
static XPLMDataRef gAltitudeRef = NULL;
static XPLMCommandRef gMyCommandRef = NULL;

// Required plugin callbacks
PLUGIN_API int XPluginStart(char* outName, char* outSig, char* outDesc) {
    strcpy(outName, "Example Plugin");
    strcpy(outSig, "com.example.plugin");
    strcpy(outDesc, "Example X-Plane plugin");
    
    // Enable modern features
    XPLMEnableFeature("XPLM_USE_NATIVE_PATHS", 1);
    
    // Look up datarefs once at startup
    gAltitudeRef = XPLMFindDataRef("sim/cockpit2/gauges/indicators/altitude_ft_pilot");
    
    // Create custom command
    gMyCommandRef = XPLMCreateCommand("example/toggle_feature", "Toggle Example Feature");
    
    return 1;
}

PLUGIN_API int XPluginEnable(void) {
    // Register callbacks during enable
    XPLMRegisterCommandHandler(gMyCommandRef, MyCommandHandler, 1, NULL);
    XPLMRegisterFlightLoopCallback(MyFlightLoopCallback, -1.0, NULL);
    return 1;
}

PLUGIN_API void XPluginDisable(void) {
    // Unregister callbacks during disable
    XPLMUnregisterCommandHandler(gMyCommandRef, MyCommandHandler, 1, NULL);
    XPLMUnregisterFlightLoopCallback(MyFlightLoopCallback, NULL);
}

PLUGIN_API void XPluginStop(void) {
    // Cleanup resources
}

PLUGIN_API void XPluginReceiveMessage(XPLMPluginID inFromWho, int inMessage, void* inParam) {
    // Handle inter-plugin messages
    if (inMessage == XPLM_MSG_PLANE_LOADED) {
        // Plane changed - refresh datarefs if needed
    }
}
```

### Dataref Operations
```c
// Reading data
float altitude = XPLMGetDataf(gAltitudeRef);
int gear_position = XPLMGetDatai(XPLMFindDataRef("sim/aircraft/parts/acf_gear_deploy"));

// Writing data (if dataref is writable)
XPLMSetDataf(XPLMFindDataRef("sim/operation/override/override_throttles"), 1.0f);

// Array datarefs
float engine_power[8];
int num_engines = XPLMGetDatavf(XPLMFindDataRef("sim/aircraft/engine/acf_num_engines"), 
                                engine_power, 0, 8);

// Custom dataref
int MyDatarefGetter(void* inRefcon) {
    return *(int*)inRefcon;
}
void MyDatarefSetter(void* inRefcon, int inValue) {
    *(int*)inRefcon = inValue;
}

static int my_custom_value = 42;
XPLMRegisterDataAccessor("example/my_custom_int", xplmType_Int, 1,
                        MyDatarefGetter, MyDatarefSetter,
                        NULL, NULL, NULL, NULL, NULL, NULL,
                        NULL, NULL, &my_custom_value);
```

### Modern Window Creation
```c
#include "XPLMDisplay.h"

static XPLMWindowID gWindow = NULL;

void CreateModernWindow(void) {
    XPLMCreateWindow_t params = {0};
    params.structSize = sizeof(params);
    params.left = 100;
    params.top = 400;
    params.right = 400;
    params.bottom = 200;
    params.visible = 1;
    params.drawWindowFunc = MyDrawWindowCallback;
    params.handleKeyFunc = MyHandleKeyCallback;
    params.handleMouseClickFunc = MyHandleMouseClickCallback;
    params.handleCursorFunc = MyHandleCursorCallback;
    params.handleMouseWheelFunc = MyHandleMouseWheelCallback;
    params.handleRightClickFunc = MyHandleRightClickCallback;
    params.refcon = NULL;
    params.decorateAsFloatingWindow = xplm_WindowDecorationRoundRectangle;
    params.layer = xplm_WindowLayerFloatingWindows;
    
    gWindow = XPLMCreateWindowEx(&params);
    XPLMSetWindowTitle(gWindow, "Example Window");
}

void MyDrawWindowCallback(XPLMWindowID inWindowID, void* inRefcon) {
    int left, top, right, bottom;
    XPLMGetWindowGeometry(inWindowID, &left, &top, &right, &bottom);
    
    XPLMDrawTranslucentDarkBox(left, top, right, bottom);
    
    // Draw text
    XPLMDrawString(RGB(1,1,1), left + 10, top - 20, "Hello World!", NULL, xplmFont_Proportional);
}
```

### Flight Loop Integration
```c
float MyFlightLoopCallback(float inElapsedSinceLastCall, 
                          float inElapsedTimeSinceLastFlightLoop,
                          int inCounter, 
                          void* inRefcon) {
    // Periodic processing - runs every flight loop cycle
    static int counter = 0;
    counter++;
    
    if (counter % 60 == 0) { // Every ~1 second at 60fps
        float altitude = XPLMGetDataf(gAltitudeRef);
        // Do something with altitude data
    }
    
    return -1.0f; // Schedule for next flight loop
}
```

### Command System Usage
```c
int MyCommandHandler(XPLMCommandRef inCommand, XPLMCommandPhase inPhase, void* inRefcon) {
    if (inPhase == xplm_CommandBegin) {
        // Command started
        XPLMDebugString("Example: Command began\n");
        return 1; // Let other handlers process too
    }
    else if (inPhase == xplm_CommandEnd) {
        // Command ended
        XPLMDebugString("Example: Command ended\n");
    }
    return 1;
}

// Triggering existing X-Plane commands
XPLMCommandRef landing_lights = XPLMFindCommand("sim/lights/landing_lights_toggle");
XPLMCommandOnce(landing_lights);
```

### Avionics Device Integration
```c
#include "XPLMDisplay.h"

void SetupGPSDevice(void) {
    XPLMCustomizeAvionics_t params = {0};
    params.structSize = sizeof(params);
    params.deviceId = xplm_device_GNS430_1;
    params.drawCallback = MyGPSDrawCallback;
    params.screenTouchCallback = MyGPSTouchCallback;
    params.refcon = NULL;
    
    XPLMAvionicsID avionics_handle = XPLMRegisterAvionicsCallbacksEx(&params);
}

void MyGPSDrawCallback(XPLMDeviceID inDeviceID, int inIsBefore, void* inRefcon) {
    if (inIsBefore) return; // Let X-Plane draw first
    
    // Draw custom overlay on GPS screen
    // Coordinates are in texels, origin at bottom-left
    XPLMSetGraphicsState(0, 1, 0, 1, 1, 0, 0);
    glColor3f(0, 1, 0); // Green overlay
    glBegin(GL_LINES);
    glVertex2f(10, 10);
    glVertex2f(100, 100);
    glEnd();
}
```

### Modern 3D Object Instancing
```c
#include "XPLMInstance.h"

static XPLMObjectRef gObjectRef = NULL;
static XPLMInstanceRef gInstanceRef = NULL;

void LoadAndInstance3DObject(void) {
    // Load OBJ file
    gObjectRef = XPLMLoadObject("Resources/plugins/MyPlugin/objects/my_object.obj");
    
    // Create instance with dataref control
    const char* datarefs[] = {
        "sim/aircraft/engine/acf_num_engines",
        "sim/cockpit2/gauges/indicators/altitude_ft_pilot",
        NULL // Null-terminated
    };
    
    gInstanceRef = XPLMCreateInstance(gObjectRef, datarefs);
    
    // Set initial position
    XPLMDrawInfo_t position = {0};
    position.x = 0;
    position.y = 100;  // 100 meters up
    position.z = 0;
    position.pitch = 0;
    position.heading = 0;
    position.roll = 0;
    
    float dataref_values[] = {4.0f, 5000.0f}; // Engine count, altitude
    XPLMInstanceSetPosition(gInstanceRef, &position, dataref_values);
}
```
