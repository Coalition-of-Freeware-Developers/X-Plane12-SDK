# XPLMDisplay.h - X-Plane 12 SDK Display and UI API Documentation

## Overview

The XPLMDisplay API is the cornerstone of X-Plane plugin user interface development, providing comprehensive functionality for creating windows, handling input, managing drawing contexts, and integrating with X-Plane's rendering pipeline. This API supports both legacy and modern window systems, avionics device integration, and various input handling mechanisms.

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Drawing System Architecture](#drawing-system-architecture)
- [Window System](#window-system)
- [Avionics API](#avionics-api)
- [Input Handling](#input-handling)
- [Modern vs Legacy Windows](#modern-vs-legacy-windows)
- [Implementation Guidelines](#implementation-guidelines)
- [Use Cases](#use-cases)
- [Performance Considerations](#performance-considerations)
- [Best Practices](#best-practices)
- [Common Patterns](#common-patterns)
- [Error Handling](#error-handling)
- [Advanced Features](#advanced-features)

## Architecture Overview

The Display API provides multiple layers of functionality:

**Drawing Integration**:

- Direct drawing callbacks (deprecated)
- Modern window-based drawing
- Avionics device integration
- OpenGL context management

**Window Management**:

- Legacy windows (pixel-based coordinates)
- Modern windows (boxel-based coordinates with scaling)
- Multi-monitor support
- VR integration
- Pop-out window support

**Input Processing**:

- Window-specific callbacks
- Key sniffers for low-level access
- Hot key registration
- Mouse interaction handling

## Drawing System Architecture

### Rendering Pipeline

X-Plane's rendering follows a structured pipeline:

1. **3D Scene Rendering**: Ground, objects, aircraft
2. **Cockpit Overlay**: Panel and gauges (alpha blended)
3. **UI Layer**: Windows, menus, dialogs

### Drawing Phases (Deprecated)

The legacy drawing phases are deprecated but still available for compatibility:

```c
enum {
    // 3D Phases (DEPRECATED as of XPLM302)
    xplm_Phase_FirstScene    = 0,   // Earliest 3D drawing point
    xplm_Phase_Terrain       = 5,   // Land and water
    xplm_Phase_Airports      = 10,  // Airport details
    xplm_Phase_Vectors       = 15,  // Roads, trails
    xplm_Phase_Objects       = 20,  // 3D objects
    xplm_Phase_Airplanes     = 25,  // Aircraft rendering
    xplm_Phase_LastScene     = 30,  // Latest 3D drawing point
  
    // Modern 3D Phase
    xplm_Phase_Modern3D      = 31,  // Modern 3D drawing (limited support)
  
    // 2D Phases (Active)
    xplm_Phase_FirstCockpit  = 35,  // First 2D drawing phase
    xplm_Phase_Panel         = 40,  // Non-moving panel parts
    xplm_Phase_Gauges        = 45,  // Moving panel parts
    xplm_Phase_Window        = 50,  // Plugin windows
    xplm_Phase_LastCockpit   = 55,  // Last 2D drawing phase
};
```

**Important**: Legacy 3D drawing phases are deprecated and will be removed. Use the XPLMInstance API for 3D object drawing instead.

### Modern Drawing Approach

```c
// DEPRECATED - Don't use for new code
XPLMRegisterDrawCallback(MyDrawCallback, xplm_Phase_Objects, 1, NULL);

// RECOMMENDED - Use windowing system
XPLMCreateWindowEx(&windowParams);

// RECOMMENDED - Use instancing for 3D objects
XPLMCreateInstance(objectRef, datarefs);
```

## Window System

### Modern Windows (XPLM300+)

Modern windows provide advanced features and better integration with X-Plane 11+:

**Features**:

- High-DPI scaling support ("boxel" units)
- Native X-Plane styling
- Pop-out window capability
- VR support
- Multi-monitor awareness
- Global desktop coordinate system

**Coordinate System**:

- Units: Boxels (scaled pixels)
- Origin: Lower-left of global desktop space
- Multi-monitor aware
- Automatic scaling for high-DPI displays

### Legacy Windows (Pre-XPLM300)

Legacy windows use traditional pixel coordinates:

**Features**:

- Direct pixel manipulation
- Simple coordinate system
- Limited to main X-Plane window

**Coordinate System**:

- Units: Screen pixels
- Origin: Lower-left of main X-Plane window
- Single monitor only

### Window Creation

#### Modern Window Creation

```c
XPLMCreateWindow_t windowDef = {0};
windowDef.structSize = sizeof(XPLMCreateWindow_t);
windowDef.left = 100;
windowDef.top = 400;
windowDef.right = 400;
windowDef.bottom = 200;
windowDef.visible = 1;
windowDef.drawWindowFunc = MyDrawCallback;
windowDef.handleMouseClickFunc = MyMouseCallback;
windowDef.handleKeyFunc = MyKeyCallback;
windowDef.handleCursorFunc = MyCursorCallback;
windowDef.handleMouseWheelFunc = MyWheelCallback;
windowDef.handleRightClickFunc = MyRightClickCallback;
windowDef.refcon = myData;
windowDef.decorateAsFloatingWindow = xplm_WindowDecorationRoundRectangle;
windowDef.layer = xplm_WindowLayerFloatingWindows;

XPLMWindowID window = XPLMCreateWindowEx(&windowDef);
XPLMSetWindowTitle(window, "My Plugin Window");
```

#### Legacy Window Creation

```c
XPLMWindowID window = XPLMCreateWindow(
    100, 400, 400, 200,  // left, top, right, bottom
    1,                   // visible
    MyDrawCallback,      // draw function
    MyKeyCallback,       // key handler
    MyMouseCallback,     // mouse handler
    myData               // refcon
);
```

### Window Positioning and Geometry

#### Positioning Modes

Modern windows support various positioning modes:

```c
enum {
    xplm_WindowPositionFree                = 0,  // User/gravity controlled
    xplm_WindowCenterOnMonitor            = 1,  // Center on specific monitor
    xplm_WindowFullScreenOnMonitor        = 2,  // Full screen on monitor
    xplm_WindowFullScreenOnAllMonitors    = 3,  // Span all monitors
    xplm_WindowPopOut                     = 4,  // OS window
    xplm_WindowVR                         = 5,  // VR headset (XPLM301+)
};

// Set positioning mode
XPLMSetWindowPositioningMode(window, xplm_WindowCenterOnMonitor, 0);
```

#### Gravity System

Control how windows resize with the main X-Plane window:

```c
// Gravity values: 0.0 = stick to left/bottom, 1.0 = stick to right/top, 0.5 = center
XPLMSetWindowGravity(window,
    0.0f,  // left gravity
    1.0f,  // top gravity  
    0.0f,  // right gravity
    1.0f   // bottom gravity
);
```

#### Size Constraints

```c
// Set minimum and maximum window size
XPLMSetWindowResizingLimits(window,
    200, 100,  // minimum width, height
    800, 600   // maximum width, height
);
```

### Window Callbacks

#### Draw Callback

```c
void MyDrawCallback(XPLMWindowID inWindowID, void *inRefcon) {
    int left, top, right, bottom;
    XPLMGetWindowGeometry(inWindowID, &left, &top, &right, &bottom);
  
    // Draw window background
    XPLMDrawTranslucentDarkBox(left, top, right, bottom);
  
    // Draw content
    float color[] = {1.0f, 1.0f, 1.0f};
    XPLMDrawString(color, left + 10, top - 20, "Hello World!", 
                   NULL, xplmFont_Proportional);
}
```

#### Mouse Callback

```c
int MyMouseCallback(XPLMWindowID inWindowID, int x, int y, 
                   XPLMMouseStatus inMouse, void *inRefcon) {
    switch (inMouse) {
        case xplm_MouseDown:
            // Handle mouse press
            HandleMouseDown(x, y);
            break;
        case xplm_MouseDrag:
            // Handle mouse drag
            HandleMouseDrag(x, y);
            break;
        case xplm_MouseUp:
            // Handle mouse release
            HandleMouseUp(x, y);
            break;
    }
    return 1; // Consume the event
}
```

#### Keyboard Callback

```c
void MyKeyCallback(XPLMWindowID inWindowID, char inKey, XPLMKeyFlags inFlags,
                  char inVirtualKey, void *inRefcon, int losingFocus) {
    if (losingFocus) {
        // Window is losing keyboard focus
        HandleFocusLoss();
        return;
    }
  
    // Handle key press
    if (inKey == XPLM_KEY_RETURN) {
        ProcessEnterKey();
    } else if (inKey == XPLM_KEY_ESCAPE) {
        CloseWindow();
    } else {
        ProcessCharacter(inKey);
    }
}
```

## Avionics API

The Avionics API (XPLM400+) allows customization of built-in cockpit devices and creation of new devices.

### Built-in Device IDs

```c
enum {
    xplm_device_GNS430_1,        // GNS430, pilot side
    xplm_device_GNS430_2,        // GNS430, copilot side
    xplm_device_GNS530_1,        // GNS530, pilot side
    xplm_device_GNS530_2,        // GNS530, copilot side
    xplm_device_CDU739_1,        // CDU, pilot side
    xplm_device_CDU739_2,        // CDU, copilot side
    xplm_device_G1000_PFD_1,     // G1000 PFD, pilot
    xplm_device_G1000_MFD,       // G1000 MFD
    xplm_device_G1000_PFD_2,     // G1000 PFD, copilot
    // ... more devices
};
```

### Customizing Built-in Devices

```c
XPLMCustomizeAvionics_t customizeParams = {0};
customizeParams.structSize = sizeof(XPLMCustomizeAvionics_t);
customizeParams.deviceId = xplm_device_GNS430_1;
customizeParams.drawCallbackBefore = MyBeforeDrawCallback;
customizeParams.drawCallbackAfter = MyAfterDrawCallback;
customizeParams.screenTouchCallback = MyTouchCallback;
customizeParams.refcon = myDeviceData;

XPLMAvionicsID avionicsHandle = XPLMRegisterAvionicsCallbacksEx(&customizeParams);
```

### Creating Custom Devices

```c
XPLMCreateAvionics_t createParams = {0};
createParams.structSize = sizeof(XPLMCreateAvionics_t);
createParams.screenWidth = 480;
createParams.screenHeight = 320;
createParams.bezelWidth = 540;
createParams.bezelHeight = 380;
createParams.screenOffsetX = 30;
createParams.screenOffsetY = 30;
createParams.drawOnDemand = 0;
createParams.drawCallback = MyScreenDrawCallback;
createParams.bezelDrawCallback = MyBezelDrawCallback;
createParams.screenTouchCallback = MyScreenTouchCallback;
createParams.deviceID = "com.example.my_device";
createParams.deviceName = "Example GPS Device";
createParams.refcon = myDeviceData;

XPLMAvionicsID deviceHandle = XPLMCreateAvionicsEx(&createParams);
```

### Avionics Callbacks

#### Drawing Callback

```c
int MyAvionicsDrawCallback(XPLMDeviceID inDeviceID, int inIsBefore, void *inRefcon) {
    if (inIsBefore) {
        // Draw before X-Plane
        DrawCustomOverlay();
        return 1; // Let X-Plane draw
    } else {
        // Draw after X-Plane
        DrawCustomElements();
        return 1; // Return value ignored for 'after' calls
    }
}
```

#### Touch Callback

```c
int MyTouchCallback(int x, int y, XPLMMouseStatus inMouse, void *inRefcon) {
    switch (inMouse) {
        case xplm_MouseDown:
            return HandleScreenTouch(x, y);
        case xplm_MouseDrag:
            return HandleScreenDrag(x, y);
        case xplm_MouseUp:
            return HandleScreenRelease(x, y);
    }
    return 0; // Pass through to X-Plane
}
```

### Avionics Device Management

```c
// Control popup window visibility
XPLMSetAvionicsPopupVisible(deviceHandle, 1);
if (XPLMIsAvionicsPopupVisible(deviceHandle)) {
    // Popup is visible
}

// Pop out device to OS window
XPLMPopOutAvionics(deviceHandle);
if (XPLMIsAvionicsPoppedOut(deviceHandle)) {
    // Device is popped out
}

// Control brightness
XPLMSetAvionicsBrightnessRheo(deviceHandle, 0.8f);
float brightness = XPLMGetAvionicsBrightnessRheo(deviceHandle);

// Check cursor position
int cursorX, cursorY;
if (XPLMIsCursorOverAvionics(deviceHandle, &cursorX, &cursorY)) {
    // Cursor is over device screen
}
```

## Input Handling

### Key Sniffers

Low-level keyboard access for libraries and special use cases:

```c
int MyKeySniffer(char inChar, XPLMKeyFlags inFlags, char inVirtualKey, void *inRefcon) {
    // Handle special key combinations
    if ((inFlags & xplm_ShiftFlag) && inVirtualKey == XPLM_VK_F1) {
        // Handle Shift+F1
        ProcessSpecialCommand();
        return 0; // Consume the key
    }
  
    return 1; // Pass through
}

// Register key sniffer
XPLMRegisterKeySniffer(MyKeySniffer, 0, NULL); // After windows

// Unregister when done
XPLMUnregisterKeySniffer(MyKeySniffer, 0, NULL);
```

### Hot Keys

High-level key binding system:

```c
void MyHotKeyCallback(void *inRefcon) {
    TogglePluginWindow();
}

XPLMHotKeyID hotKey = XPLMRegisterHotKey(
    XPLM_VK_F1,                    // Virtual key
    xplm_ShiftFlag,                // Modifiers
    "Toggle My Plugin Window",     // Description
    MyHotKeyCallback,              // Callback
    NULL                           // Refcon
);

// Later, unregister
XPLMUnregisterHotKey(hotKey);
```

## Modern vs Legacy Windows

### Feature Comparison

| Feature            | Modern Windows  | Legacy Windows |
| ------------------ | --------------- | -------------- |
| Coordinate Units   | Boxels (scaled) | Pixels         |
| High-DPI Support   | ✓              | ✗             |
| Multi-monitor      | ✓              | ✗             |
| Pop-out Support    | ✓              | ✗             |
| VR Support         | ✓              | ✗             |
| Native Styling     | ✓              | ✗             |
| Global Coordinates | ✓              | ✗             |

### Migration Guidelines

Converting legacy windows to modern:

```c
// Legacy approach
XPLMWindowID legacyWindow = XPLMCreateWindow(
    100, 400, 300, 200, 1,
    DrawCallback, KeyCallback, MouseCallback, NULL
);

// Modern equivalent
XPLMCreateWindow_t params = {0};
params.structSize = sizeof(params);
params.left = 100;
params.top = 400;
params.right = 300;
params.bottom = 200;
params.visible = 1;
params.drawWindowFunc = DrawCallback;
params.handleKeyFunc = KeyCallback;
params.handleMouseClickFunc = MouseCallback;
params.handleCursorFunc = DefaultCursorCallback;
params.handleMouseWheelFunc = DefaultWheelCallback;
params.refcon = NULL;
params.decorateAsFloatingWindow = xplm_WindowDecorationRoundRectangle;
params.layer = xplm_WindowLayerFloatingWindows;

XPLMWindowID modernWindow = XPLMCreateWindowEx(&params);
```

## Implementation Guidelines

### Basic Window Setup

```c
typedef struct {
    XPLMWindowID window;
    int isVisible;
    char title[256];
    // ... other data
} MyWindowData_t;

static MyWindowData_t gMyWindow;

void CreateMyWindow() {
    XPLMCreateWindow_t params = {0};
    params.structSize = sizeof(params);
    params.left = 200;
    params.top = 600;
    params.right = 600;
    params.bottom = 200;
    params.visible = 1;
    params.drawWindowFunc = DrawMyWindow;
    params.handleMouseClickFunc = HandleMyWindowClick;
    params.handleKeyFunc = HandleMyWindowKey;
    params.handleCursorFunc = HandleMyWindowCursor;
    params.handleMouseWheelFunc = HandleMyWindowWheel;
    params.handleRightClickFunc = HandleMyWindowRightClick;
    params.refcon = &gMyWindow;
    params.decorateAsFloatingWindow = xplm_WindowDecorationRoundRectangle;
    params.layer = xplm_WindowLayerFloatingWindows;
  
    gMyWindow.window = XPLMCreateWindowEx(&params);
    gMyWindow.isVisible = 1;
    strcpy(gMyWindow.title, "My Plugin");
  
    XPLMSetWindowTitle(gMyWindow.window, gMyWindow.title);
}

void DrawMyWindow(XPLMWindowID inWindowID, void *inRefcon) {
    MyWindowData_t *data = (MyWindowData_t *)inRefcon;
  
    int left, top, right, bottom;
    XPLMGetWindowGeometry(inWindowID, &left, &top, &right, &bottom);
  
    // Draw window content
    XPLMDrawTranslucentDarkBox(left, top, right, bottom);
  
    float white[] = {1.0f, 1.0f, 1.0f};
    XPLMDrawString(white, left + 10, top - 20, "Hello from my plugin!", 
                   NULL, xplmFont_Proportional);
}
```

### Multi-Monitor Support

```c
void HandleMonitorConfiguration() {
    // Get global desktop bounds
    int left, top, right, bottom;
    XPLMGetScreenBoundsGlobal(&left, &top, &right, &bottom);
  
    printf("Global desktop: %d,%d to %d,%d\n", left, top, right, bottom);
  
    // Enumerate monitors
    XPLMGetAllMonitorBoundsGlobal(MonitorCallback, NULL);
}

void MonitorCallback(int inMonitorIndex, int inLeftBx, int inTopBx, 
                    int inRightBx, int inBottomBx, void *inRefcon) {
    printf("Monitor %d: %d,%d to %d,%d (boxels)\n", 
           inMonitorIndex, inLeftBx, inTopBx, inRightBx, inBottomBx);
}

// Position window on specific monitor
void PositionOnMonitor(XPLMWindowID window, int monitorIndex) {
    XPLMSetWindowPositioningMode(window, xplm_WindowCenterOnMonitor, monitorIndex);
}
```

### Pop-out Window Management

```c
void HandlePopOut() {
    if (!XPLMWindowIsPoppedOut(gMyWindow.window)) {
        // Pop out the window
        XPLMSetWindowPositioningMode(gMyWindow.window, xplm_WindowPopOut, -1);
      
        // Set initial OS window geometry
        XPLMSetWindowGeometryOS(gMyWindow.window, 100, 100, 500, 400);
    } else {
        // Already popped out, maybe resize
        int left, top, right, bottom;
        XPLMGetWindowGeometryOS(gMyWindow.window, &left, &top, &right, &bottom);
      
        // Resize if needed
        XPLMSetWindowGeometryOS(gMyWindow.window, left, top, right + 100, bottom);
    }
}
```

## Use Cases

### 1. Plugin Configuration Dialog

```c
typedef struct {
    XPLMWindowID window;
    int setting1;
    float setting2;
    char textBuffer[256];
    int textCursorPos;
} ConfigDialog_t;

static ConfigDialog_t gConfigDialog;

void ShowConfigDialog() {
    if (!gConfigDialog.window) {
        CreateConfigWindow();
    }
    XPLMSetWindowIsVisible(gConfigDialog.window, 1);
    XPLMTakeKeyboardFocus(gConfigDialog.window);
}

void DrawConfigWindow(XPLMWindowID inWindowID, void *inRefcon) {
    ConfigDialog_t *config = (ConfigDialog_t *)inRefcon;
  
    int left, top, right, bottom;
    XPLMGetWindowGeometry(inWindowID, &left, &top, &right, &bottom);
  
    // Draw background
    XPLMDrawTranslucentDarkBox(left, top, right, bottom);
  
    // Draw settings
    float white[] = {1.0f, 1.0f, 1.0f};
    char buffer[256];
  
    sprintf(buffer, "Setting 1: %d", config->setting1);
    XPLMDrawString(white, left + 10, top - 30, buffer, NULL, xplmFont_Proportional);
  
    sprintf(buffer, "Setting 2: %.2f", config->setting2);
    XPLMDrawString(white, left + 10, top - 50, buffer, NULL, xplmFont_Proportional);
  
    // Draw text input field
    XPLMDrawString(white, left + 10, top - 70, "Text:", NULL, xplmFont_Proportional);
    XPLMDrawString(white, left + 60, top - 70, config->textBuffer, NULL, xplmFont_Proportional);
}

void HandleConfigKey(XPLMWindowID inWindowID, char inKey, XPLMKeyFlags inFlags,
                    char inVirtualKey, void *inRefcon, int losingFocus) {
    ConfigDialog_t *config = (ConfigDialog_t *)inRefcon;
  
    if (losingFocus) return;
  
    if (inKey >= 32 && inKey <= 126) { // Printable characters
        int len = strlen(config->textBuffer);
        if (len < sizeof(config->textBuffer) - 1) {
            config->textBuffer[len] = inKey;
            config->textBuffer[len + 1] = '\0';
        }
    } else if (inKey == XPLM_KEY_DELETE && strlen(config->textBuffer) > 0) {
        int len = strlen(config->textBuffer);
        config->textBuffer[len - 1] = '\0';
    }
}
```

### 2. Instrument Display

```c
typedef struct {
    XPLMWindowID window;
    XPLMDataRef altitudeRef;
    XPLMDataRef speedRef;
    XPLMDataRef headingRef;
} InstrumentDisplay_t;

static InstrumentDisplay_t gInstruments;

void CreateInstrumentWindow() {
    // Initialize datarefs
    gInstruments.altitudeRef = XPLMFindDataRef("sim/cockpit2/gauges/indicators/altitude_ft_pilot");
    gInstruments.speedRef = XPLMFindDataRef("sim/cockpit2/gauges/indicators/airspeed_kts_pilot");
    gInstruments.headingRef = XPLMFindDataRef("sim/cockpit2/gauges/indicators/heading_electric_deg_mag_pilot");
  
    // Create window
    XPLMCreateWindow_t params = {0};
    params.structSize = sizeof(params);
    params.left = 50;
    params.top = 300;
    params.right = 250;
    params.bottom = 50;
    params.visible = 1;
    params.drawWindowFunc = DrawInstruments;
    params.decorateAsFloatingWindow = xplm_WindowDecorationRoundRectangle;
    params.layer = xplm_WindowLayerFloatingWindows;
    params.refcon = &gInstruments;
  
    gInstruments.window = XPLMCreateWindowEx(&params);
    XPLMSetWindowTitle(gInstruments.window, "Flight Instruments");
}

void DrawInstruments(XPLMWindowID inWindowID, void *inRefcon) {
    InstrumentDisplay_t *inst = (InstrumentDisplay_t *)inRefcon;
  
    int left, top, right, bottom;
    XPLMGetWindowGeometry(inWindowID, &left, &top, &right, &bottom);
  
    // Get current values
    float altitude = XPLMGetDataf(inst->altitudeRef);
    float speed = XPLMGetDataf(inst->speedRef);
    float heading = XPLMGetDataf(inst->headingRef);
  
    // Draw background
    XPLMDrawTranslucentDarkBox(left, top, right, bottom);
  
    // Draw instruments
    float white[] = {1.0f, 1.0f, 1.0f};
    float green[] = {0.0f, 1.0f, 0.0f};
  
    char buffer[256];
    sprintf(buffer, "ALT: %.0f ft", altitude);
    XPLMDrawString(green, left + 10, top - 25, buffer, NULL, xplmFont_Proportional);
  
    sprintf(buffer, "SPD: %.0f kts", speed);
    XPLMDrawString(green, left + 10, top - 45, buffer, NULL, xplmFont_Proportional);
  
    sprintf(buffer, "HDG: %.0f°", heading);
    XPLMDrawString(green, left + 10, top - 65, buffer, NULL, xplmFont_Proportional);
}
```

### 3. Custom GPS Device

```c
typedef struct {
    XPLMAvionicsID deviceHandle;
    char display[20][40];  // 20 lines, 40 chars each
    int currentPage;
    int cursorX, cursorY;
} CustomGPS_t;

static CustomGPS_t gGPS;

void CreateCustomGPS() {
    XPLMCreateAvionics_t params = {0};
    params.structSize = sizeof(params);
    params.screenWidth = 480;
    params.screenHeight = 320;
    params.bezelWidth = 540;
    params.bezelHeight = 380;
    params.screenOffsetX = 30;
    params.screenOffsetY = 30;
    params.drawOnDemand = 0;
    params.drawCallback = DrawGPSScreen;
    params.bezelDrawCallback = DrawGPSBezel;
    params.screenTouchCallback = HandleGPSTouch;
    params.deviceID = "com.example.custom_gps";
    params.deviceName = "Example GPS";
    params.refcon = &gGPS;
  
    gGPS.deviceHandle = XPLMCreateAvionicsEx(&params);
    InitializeGPSDisplay();
}

void DrawGPSScreen(void *inRefcon) {
    CustomGPS_t *gps = (CustomGPS_t *)inRefcon;
  
    // Clear screen
    glClearColor(0.0f, 0.0f, 0.2f, 1.0f);
    glClear(GL_COLOR_BUFFER_BIT);
  
    // Draw text display
    float green[] = {0.0f, 1.0f, 0.0f};
    for (int line = 0; line < 20; line++) {
        XPLMDrawString(green, 10, 300 - (line * 15), gps->display[line], 
                      NULL, xplmFont_Proportional);
    }
  
    // Draw cursor if needed
    if (gps->cursorX >= 0 && gps->cursorY >= 0) {
        // Draw blinking cursor
        static float blinkTime = 0.0f;
        blinkTime += 0.016f;
      
        if (fmod(blinkTime, 1.0f) < 0.5f) {
            float yellow[] = {1.0f, 1.0f, 0.0f};
            XPLMDrawString(yellow, 10 + gps->cursorX * 8, 300 - gps->cursorY * 15, 
                          "_", NULL, xplmFont_Proportional);
        }
    }
}
```

## Performance Considerations

### Efficient Drawing

```c
void OptimizedDrawCallback(XPLMWindowID inWindowID, void *inRefcon) {
    // Cache window geometry
    static int lastLeft = -1, lastTop = -1, lastRight = -1, lastBottom = -1;
    static int geometryChanged = 1;
  
    int left, top, right, bottom;
    XPLMGetWindowGeometry(inWindowID, &left, &top, &right, &bottom);
  
    if (left != lastLeft || top != lastTop || right != lastRight || bottom != lastBottom) {
        lastLeft = left; lastTop = top; lastRight = right; lastBottom = bottom;
        geometryChanged = 1;
    }
  
    // Only recalculate layout when geometry changes
    if (geometryChanged) {
        RecalculateLayout(left, top, right, bottom);
        geometryChanged = 0;
    }
  
    // Draw efficiently
    DrawCachedContent();
}
```

### Minimize Dataref Access

```c
// BAD: Reading datarefs every draw
void SlowDrawCallback(XPLMWindowID inWindowID, void *inRefcon) {
    float altitude = XPLMGetDataf(XPLMFindDataRef("sim/cockpit2/gauges/indicators/altitude_ft_pilot"));
    // ... draw altitude
}

// GOOD: Cache datarefs, read periodically
static XPLMDataRef gAltitudeRef = NULL;
static float gCachedAltitude = 0.0f;
static float gLastUpdateTime = 0.0f;

void FastDrawCallback(XPLMWindowID inWindowID, void *inRefcon) {
    float currentTime = XPLMGetElapsedTime();
  
    // Update cached value every 0.1 seconds
    if (currentTime - gLastUpdateTime > 0.1f) {
        if (!gAltitudeRef) {
            gAltitudeRef = XPLMFindDataRef("sim/cockpit2/gauges/indicators/altitude_ft_pilot");
        }
        if (gAltitudeRef) {
            gCachedAltitude = XPLMGetDataf(gAltitudeRef);
        }
        gLastUpdateTime = currentTime;
    }
  
    // Use cached value for drawing
    DrawAltitude(gCachedAltitude);
}
```

## Best Practices

### Window Lifecycle Management

```c
typedef struct {
    XPLMWindowID window;
    int isCreated;
    int isVisible;
} WindowManager_t;

static WindowManager_t gWindowMgr = {0};

void CreateWindowIfNeeded() {
    if (!gWindowMgr.isCreated) {
        // Create window
        gWindowMgr.window = CreateMyWindow();
        gWindowMgr.isCreated = 1;
        gWindowMgr.isVisible = 1;
    }
}

void ToggleWindowVisibility() {
    CreateWindowIfNeeded();
  
    gWindowMgr.isVisible = !gWindowMgr.isVisible;
    XPLMSetWindowIsVisible(gWindowMgr.window, gWindowMgr.isVisible);
}

void CleanupWindows() {
    if (gWindowMgr.isCreated) {
        XPLMDestroyWindow(gWindowMgr.window);
        gWindowMgr.isCreated = 0;
        gWindowMgr.isVisible = 0;
    }
}

// Plugin callbacks
int XPluginEnable(void) {
    // Don't create windows here - create on demand
    return 1;
}

void XPluginDisable(void) {
    // Clean up all windows
    CleanupWindows();
}
```

### Input Validation

```c
int SafeMouseCallback(XPLMWindowID inWindowID, int x, int y, 
                     XPLMMouseStatus inMouse, void *inRefcon) {
    // Validate window geometry
    int left, top, right, bottom;
    XPLMGetWindowGeometry(inWindowID, &left, &top, &right, &bottom);
  
    // Check if click is within bounds
    if (x < left || x > right || y < bottom || y > top) {
        return 0; // Pass through invalid clicks
    }
  
    // Validate refcon
    if (!inRefcon) {
        XPLMDebugString("ERROR: NULL refcon in mouse callback\n");
        return 0;
    }
  
    // Process valid input
    MyWindowData_t *data = (MyWindowData_t *)inRefcon;
    return ProcessMouseInput(data, x - left, top - y, inMouse);
}
```

### Error Recovery

```c
XPLMWindowID CreateRobustWindow() {
    XPLMCreateWindow_t params = {0};
    params.structSize = sizeof(params);
  
    // Set reasonable defaults
    int screenWidth, screenHeight;
    XPLMGetScreenSize(&screenWidth, &screenHeight);
  
    params.left = screenWidth / 4;
    params.right = 3 * screenWidth / 4;
    params.bottom = screenHeight / 4;
    params.top = 3 * screenHeight / 4;
    params.visible = 1;
  
    // Provide all required callbacks
    params.drawWindowFunc = SafeDrawCallback;
    params.handleMouseClickFunc = SafeMouseCallback;
    params.handleKeyFunc = SafeKeyCallback;
    params.handleCursorFunc = DefaultCursorCallback;
    params.handleMouseWheelFunc = DefaultWheelCallback;
  
    params.refcon = AllocateWindowData();
    params.decorateAsFloatingWindow = xplm_WindowDecorationRoundRectangle;
    params.layer = xplm_WindowLayerFloatingWindows;
  
    XPLMWindowID window = XPLMCreateWindowEx(&params);
    if (!window) {
        XPLMDebugString("ERROR: Failed to create window\n");
        FreeWindowData(params.refcon);
        return NULL;
    }
  
    XPLMSetWindowTitle(window, "My Plugin");
    return window;
}
```
