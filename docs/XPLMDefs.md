# XPLMDefs.h - X-Plane 12 SDK Core Definitions Documentation

## Overview

The XPLMDefs.h header file provides the foundational definitions, data types, and platform-specific macros for the X-Plane 12 Plugin SDK. This header must be included before any other XPLM headers and establishes the core infrastructure for cross-platform plugin development.

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Platform Macros](#platform-macros)
- [DLL Export/Import Definitions](#dll-exportimport-definitions)
- [Core Data Types](#core-data-types)
- [Plugin Identification](#plugin-identification)
- [Key and Mouse Input](#key-and-mouse-input)
- [Virtual Key Codes](#virtual-key-codes)
- [Fixed-Size Data Structures](#fixed-size-data-structures)
- [Implementation Guidelines](#implementation-guidelines)
- [Platform-Specific Considerations](#platform-specific-considerations)
- [Best Practices](#best-practices)
- [Common Use Cases](#common-use-cases)

## Architecture Overview

XPLMDefs.h serves as the foundation layer for the entire X-Plane SDK by providing:

- **Cross-Platform Abstraction**: Unified API across Windows, macOS, and Linux
- **Function Export Mechanisms**: Proper DLL/shared library symbol visibility
- **Core Type Definitions**: Basic data types used throughout the SDK
- **Input System Integration**: Key codes and mouse interaction definitions
- **Version Identification**: SDK version constants and compatibility macros

This header ensures that plugins can be developed once and compiled for all supported platforms with minimal platform-specific code.

## Platform Macros

### Required Platform Definition

**Critical**: Exactly one of these macros must be defined before including any XPLM headers:

```c
#define APL 1    // macOS compilation target
#define IBM 1    // Windows compilation target  
#define LIN 1    // Linux compilation target
```

### Implementation Methods

**Preprocessor Definition**:

```c
// Method 1: In source code
#define IBM 1
#include "XPLMDefs.h"

// Method 2: Compiler command line
// gcc -DIBM=1 -c myplugin.c
// cl /DIBM=1 myplugin.c
```

**CMake Example**:

```cmake
if(WIN32)
    target_compile_definitions(myplugin PRIVATE IBM=1)
elseif(APPLE)
    target_compile_definitions(myplugin PRIVATE APL=1)
elseif(UNIX)
    target_compile_definitions(myplugin PRIVATE LIN=1)
endif()
```

**Makefile Example**:

```makefile
# Windows
CFLAGS += -DIBM=1

# macOS  
CFLAGS += -DAPL=1

# Linux
CFLAGS += -DLIN=1
```

## DLL Export/Import Definitions

### PLUGIN_API Macro

The `PLUGIN_API` macro ensures proper export of the five required plugin entry points.

**Definition per Platform**:

```c
// Windows (IBM=1)
#define PLUGIN_API extern "C" __declspec(dllexport)  // C++
#define PLUGIN_API __declspec(dllexport)             // C

// macOS (APL=1) - GCC 4+
#define PLUGIN_API extern "C" __attribute__((visibility("default")))  // C++
#define PLUGIN_API __attribute__((visibility("default")))             // C

// Linux (LIN=1) - GCC 4+
#define PLUGIN_API extern "C" __attribute__((visibility("default")))  // C++
#define PLUGIN_API __attribute__((visibility("default")))             // C
```

**Usage**:

```c
// Required plugin entry points
PLUGIN_API int XPluginStart(char* outName, char* outSig, char* outDesc);
PLUGIN_API void XPluginStop(void);
PLUGIN_API int XPluginEnable(void);
PLUGIN_API void XPluginDisable(void);
PLUGIN_API void XPluginReceiveMessage(XPLMPluginID inFrom, int inMsg, void* inParam);
```

### XPLM_API Macro

The `XPLM_API` macro handles SDK function imports/exports.

**For Plugin Development** (normal usage):

```c
// Functions provided by X-Plane SDK
XPLM_API XPLMDataRef XPLMFindDataRef(const char* inDataRefName);
XPLM_API float XPLMGetDataf(XPLMDataRef inDataRef);
```

**Plugin developers typically don't need to use XPLM_API directly** - it's handled automatically by the SDK headers.

## Core Data Types

### XPLMPluginID

```c
typedef int XPLMPluginID;

// Special constants
#define XPLM_NO_PLUGIN_ID    (-1)    // No plugin/invalid ID
#define XPLM_PLUGIN_XPLANE   (0)     // X-Plane itself
```

**Purpose**: Unique identifier for each loaded plugin within the current X-Plane session.

**Important Characteristics**:

- IDs are unique within a single X-Plane session
- IDs may change between X-Plane launches
- IDs are reassigned when plugins are reloaded
- Use `XPLMFindPluginBySignature()` for persistent plugin identification

**Usage**:

```c
// Get your own plugin ID
XPLMPluginID myID = XPLMGetMyID();

// Find another plugin by signature
XPLMPluginID otherPlugin = XPLMFindPluginBySignature("com.example.otherplugin");
if (otherPlugin != XPLM_NO_PLUGIN_ID) {
    // Plugin is loaded and available
}

// Check if message came from X-Plane itself
void XPluginReceiveMessage(XPLMPluginID inFrom, int inMsg, void* inParam) {
    if (inFrom == XPLM_PLUGIN_XPLANE) {
        // Message from X-Plane core
    }
}
```

### XPLMKeyFlags

```c
enum {
    xplm_ShiftFlag      = 1,     // Shift key modifier
    xplm_OptionAltFlag  = 2,     // Option/Alt key modifier  
    xplm_ControlFlag    = 4,     // Control key modifier
    xplm_DownFlag       = 8,     // Key press (down)
    xplm_UpFlag         = 16,    // Key release (up)
};
typedef int XPLMKeyFlags;
```

**Purpose**: Platform-independent key modifier and state flags.

**Key Modifier Mapping**:

- **Windows**: Control = Ctrl, Option/Alt = Alt
- **macOS**: Control = Ctrl (not Cmd), Option/Alt = Option
- **Linux**: Control = Ctrl, Option/Alt = Alt

**Flag Combinations**:

```c
void HandleKeyInput(char inKey, XPLMKeyFlags inFlags, char inVirtualKey, void* inRefcon) {
    // Check for modifier combinations
    if (inFlags & xplm_ControlFlag) {
        // Control key is held
    }
  
    if ((inFlags & (xplm_ControlFlag | xplm_ShiftFlag)) == (xplm_ControlFlag | xplm_ShiftFlag)) {
        // Both Control and Shift are held
    }
  
    // Check key state
    if (inFlags & xplm_DownFlag) {
        // Key is being pressed down
    } else if (inFlags & xplm_UpFlag) {
        // Key is being released
    } else {
        // Key is being held (repeated)
    }
}
```

### XPLMMouseStatus

```c
enum {
    xplm_MouseDown = 1,    // Mouse button pressed
    xplm_MouseDrag = 2,    // Mouse dragged while button held
    xplm_MouseUp   = 3,    // Mouse button released
};
typedef int XPLMMouseStatus;
```

**Purpose**: Describes mouse interaction phases.

**Event Sequence**: Mouse events follow a guaranteed sequence:

1. `xplm_MouseDown` - Initial click
2. `xplm_MouseDrag` - Zero or more drag events (optional)
3. `xplm_MouseUp` - Final release

**Usage**:

```c
int HandleMouseClick(XPLMWindowID inWindowID, int x, int y, XPLMMouseStatus inMouse, void* inRefcon) {
    switch (inMouse) {
        case xplm_MouseDown:
            // Start of interaction - capture mouse
            startDragOperation(x, y);
            return 1; // Consume the click
          
        case xplm_MouseDrag:
            // Update drag operation
            updateDragOperation(x, y);
            return 1;
          
        case xplm_MouseUp:
            // Complete the operation
            completeDragOperation(x, y);
            return 1;
    }
    return 0; // Don't consume
}
```

### XPLMCursorStatus (XPLM200+)

```c
enum {
    xplm_CursorDefault = 0,    // X-Plane manages cursor normally
    xplm_CursorHidden  = 1,    // Hide cursor
    xplm_CursorArrow   = 2,    // Show default arrow cursor
    xplm_CursorCustom  = 3,    // Plugin manages cursor
};
typedef int XPLMCursorStatus;
```

**Purpose**: Controls cursor appearance and behavior in windows.

## Key and Mouse Input

### ASCII Control Key Codes

Predefined constants for common control keys:

```c
#define XPLM_KEY_RETURN      13    // Enter/Return key
#define XPLM_KEY_ESCAPE      27    // Escape key
#define XPLM_KEY_TAB         9     // Tab key
#define XPLM_KEY_DELETE      8     // Delete/Backspace key
#define XPLM_KEY_LEFT        28    // Left arrow key
#define XPLM_KEY_RIGHT       29    // Right arrow key
#define XPLM_KEY_UP          30    // Up arrow key
#define XPLM_KEY_DOWN        31    // Down arrow key

// Numeric keys
#define XPLM_KEY_0           48    // '0' key
#define XPLM_KEY_1           49    // '1' key
// ... through XPLM_KEY_9 (57)

#define XPLM_KEY_DECIMAL     46    // Decimal point
```

**Important Notes**:

- ASCII codes consider modifier keys (Shift affects capitalization)
- Control key combinations may produce NULL characters
- Not all keystrokes generate ASCII values
- Use virtual key codes for non-ASCII keys

## Virtual Key Codes

Virtual key codes provide platform-independent identification of physical keyboard keys.

### Standard Keys

```c
#define XPLM_VK_BACK         0x08    // Backspace
#define XPLM_VK_TAB          0x09    // Tab
#define XPLM_VK_RETURN       0x0D    // Enter
#define XPLM_VK_ESCAPE       0x1B    // Escape
#define XPLM_VK_SPACE        0x20    // Spacebar

// Navigation keys
#define XPLM_VK_PRIOR        0x21    // Page Up
#define XPLM_VK_NEXT         0x22    // Page Down
#define XPLM_VK_END          0x23    // End
#define XPLM_VK_HOME         0x24    // Home
#define XPLM_VK_LEFT         0x25    // Left Arrow
#define XPLM_VK_UP           0x26    // Up Arrow
#define XPLM_VK_RIGHT        0x27    // Right Arrow
#define XPLM_VK_DOWN         0x28    // Down Arrow
```

### Alphanumeric Keys

```c
// Numbers (0x30 - 0x39)
#define XPLM_VK_0            0x30
#define XPLM_VK_1            0x31
// ... through XPLM_VK_9 (0x39)

// Letters (0x41 - 0x5A)
#define XPLM_VK_A            0x41
#define XPLM_VK_B            0x42
// ... through XPLM_VK_Z (0x5A)
```

### Numeric Keypad

```c
#define XPLM_VK_NUMPAD0      0x60    // Keypad 0
#define XPLM_VK_NUMPAD1      0x61    // Keypad 1
// ... through XPLM_VK_NUMPAD9 (0x69)

#define XPLM_VK_MULTIPLY     0x6A    // Keypad *
#define XPLM_VK_ADD          0x6B    // Keypad +
#define XPLM_VK_SUBTRACT     0x6D    // Keypad -
#define XPLM_VK_DECIMAL      0x6E    // Keypad .
#define XPLM_VK_DIVIDE       0x6F    // Keypad /
```

### Function Keys

```c
#define XPLM_VK_F1           0x70    // F1
#define XPLM_VK_F2           0x71    // F2
// ... through XPLM_VK_F24 (0x87)
```

### Extended Keys (X-Plane Specific)

```c
#define XPLM_VK_EQUAL        0xB0    // =
#define XPLM_VK_MINUS        0xB1    // -
#define XPLM_VK_RBRACE       0xB2    // ]
#define XPLM_VK_LBRACE       0xB3    // [
#define XPLM_VK_QUOTE        0xB4    // '
#define XPLM_VK_SEMICOLON    0xB5    // ;
#define XPLM_VK_BACKSLASH    0xB6    // \
#define XPLM_VK_COMMA        0xB7    // ,
#define XPLM_VK_SLASH        0xB8    // /
#define XPLM_VK_PERIOD       0xB9    // .
#define XPLM_VK_BACKQUOTE    0xBA    // `
```

### Virtual Key Usage

```c
void HandleKeyPress(char inKey, XPLMKeyFlags inFlags, char inVirtualKey, void* inRefcon) {
    switch (inVirtualKey) {
        case XPLM_VK_F1:
            ShowHelp();
            break;
          
        case XPLM_VK_ESCAPE:
            CloseDialog();
            break;
          
        case XPLM_VK_RETURN:
            if (inFlags & xplm_ControlFlag) {
                ExecuteCommand();
            } else {
                AcceptInput();
            }
            break;
          
        case XPLM_VK_DELETE:
            DeleteSelectedItem();
            break;
    }
}
```

## Fixed-Size Data Structures

### XPLMFixedString150_t

```c
typedef struct {
    char buffer[150];    // Fixed-size string buffer
} XPLMFixedString150_t;
```

**Purpose**: Provides a standardized fixed-size string container for API consistency.

**Usage**:

```c
XPLMFixedString150_t pluginName;
strcpy(pluginName.buffer, "My Plugin Name");

// Ensure null termination
pluginName.buffer[149] = '\0';

// Safe string operations
strncpy(pluginName.buffer, userInput, 149);
pluginName.buffer[149] = '\0';
```

## Implementation Guidelines

### Plugin Structure Template

```c
// File: MyPlugin.c
#define IBM 1  // Set platform (or use build system)

#include "XPLMDefs.h"
#include "XPLMPlugin.h"
#include "XPLMDataAccess.h"
#include "XPLMUtilities.h"

// Plugin entry points
PLUGIN_API int XPluginStart(char* outName, char* outSig, char* outDesc) {
    // Use fixed-size string for safety
    strncpy(outName, "My Plugin", 255);
    strncpy(outSig, "com.mycompany.myplugin", 255);
    strncpy(outDesc, "Description of my plugin", 255);
  
    // Null-terminate for safety
    outName[255] = '\0';
    outSig[255] = '\0';
    outDesc[255] = '\0';
  
    return 1;  // Success
}

PLUGIN_API void XPluginStop(void) {
    // Cleanup resources
}

PLUGIN_API int XPluginEnable(void) {
    // Initialize functionality
    return 1;
}

PLUGIN_API void XPluginDisable(void) {
    // Disable functionality
}

PLUGIN_API void XPluginReceiveMessage(XPLMPluginID inFrom, int inMsg, void* inParam) {
    // Handle messages
}
```

### Cross-Platform Considerations

```c
// Platform-specific includes (if needed)
#if IBM
    #include <windows.h>
    #define PATH_SEPARATOR "\\"
#elif APL
    #include <CoreFoundation/CoreFoundation.h>
    #define PATH_SEPARATOR "/"
#elif LIN
    #include <unistd.h>
    #define PATH_SEPARATOR "/"
#endif

// Platform-specific functionality
void GetSystemInfo() {
#if IBM
    // Windows-specific code
    SYSTEM_INFO sysInfo;
    GetSystemInfo(&sysInfo);
#elif APL
    // macOS-specific code
    SInt32 major, minor, bugfix;
    Gestalt(gestaltSystemVersionMajor, &major);
#elif LIN
    // Linux-specific code
    struct utsname unameData;
    uname(&unameData);
#endif
}
```

## Platform-Specific Considerations

### Windows (IBM=1)

**Compiler Requirements**:

- Visual Studio 2015 or later recommended
- MinGW-w64 supported
- Must link against `XPLM_64.lib`

**DLL Considerations**:

```c
// Windows-specific plugin initialization
#if IBM
BOOL APIENTRY DllMain(HMODULE hModule, DWORD fdwReason, LPVOID lpReserved) {
    switch (fdwReason) {
        case DLL_PROCESS_ATTACH:
            // Initialize Windows-specific resources
            break;
        case DLL_PROCESS_DETACH:
            // Cleanup Windows-specific resources
            break;
    }
    return TRUE;
}
#endif
```

### macOS (APL=1)

**Framework Requirements**:

- Link against `XPLM.framework`
- May require `XPWidgets.framework` for UI
- Use Xcode or command-line tools

**Bundle Structure**:

```
MyPlugin.xpl/
├── Contents/
│   ├── Info.plist
│   ├── MacOS/
│   │   └── MyPlugin (executable)
│   └── Resources/
```

### Linux (LIN=1)

**Build Requirements**:

- GCC 4.8+ or Clang 3.4+
- No link libraries required (runtime symbol resolution)

**Shared Library Configuration**:

```bash
# Compile with proper flags
gcc -fPIC -DLIN=1 -shared -o MyPlugin.xpl MyPlugin.c

# Check symbol visibility
nm -D MyPlugin.xpl | grep XPlugin
```

## Best Practices

### Safe String Handling

```c
// GOOD: Safe string operations
void SafeStringCopy(char* dest, const char* src, size_t maxLen) {
    strncpy(dest, src, maxLen - 1);
    dest[maxLen - 1] = '\0';  // Ensure null termination
}

// GOOD: Using fixed-size structures
void GetPluginInfo(XPLMFixedString150_t* name) {
    SafeStringCopy(name->buffer, "My Plugin Name", sizeof(name->buffer));
}

// BAD: Unsafe string operations
void UnsafeStringCopy(char* dest, const char* src) {
    strcpy(dest, src);  // No bounds checking!
}
```

### Platform Detection

```c
// GOOD: Clear platform detection
void InitializePlatformSpecific() {
#if defined(IBM)
    InitializeWindows();
#elif defined(APL)
    InitializeMacOS();
#elif defined(LIN)
    InitializeLinux();
#else
    #error "Platform not defined"
#endif
}

// ACCEPTABLE: Runtime platform detection
void RuntimePlatformCheck() {
    XPLMHostApplicationID hostID;
    int xplaneVersion, xplmVersion;
    XPLMGetVersions(&xplaneVersion, &xplmVersion, &hostID);
  
    // Use hostID if needed for runtime decisions
}
```

### Plugin ID Management

```c
// GOOD: Store plugin IDs appropriately
static XPLMPluginID gMyPluginID = XPLM_NO_PLUGIN_ID;
static XPLMPluginID gDependentPluginID = XPLM_NO_PLUGIN_ID;

int XPluginStart(...) {
    // Get own ID
    gMyPluginID = XPLMGetMyID();
  
    // Find dependent plugin by signature (persistent)
    gDependentPluginID = XPLMFindPluginBySignature("com.example.required");
  
    if (gDependentPluginID == XPLM_NO_PLUGIN_ID) {
        XPLMDebugString("Required plugin not found\n");
        return 0;  // Fail to load
    }
  
    return 1;
}
```

## Common Use Cases

### Input Handling

```c
typedef struct {
    int shiftPressed;
    int ctrlPressed;
    int altPressed;
} ModifierState_t;

ModifierState_t GetModifierState(XPLMKeyFlags flags) {
    ModifierState_t state = {0};
    state.shiftPressed = (flags & xplm_ShiftFlag) != 0;
    state.ctrlPressed = (flags & xplm_ControlFlag) != 0;
    state.altPressed = (flags & xplm_OptionAltFlag) != 0;
    return state;
}

void HandleKeyInput(char inKey, XPLMKeyFlags inFlags, char inVirtualKey, void* inRefcon) {
    ModifierState_t mods = GetModifierState(inFlags);
  
    if (mods.ctrlPressed && inVirtualKey == XPLM_VK_S) {
        // Ctrl+S - Save
        SaveConfiguration();
    } else if (mods.ctrlPressed && inVirtualKey == XPLM_VK_O) {
        // Ctrl+O - Open
        OpenConfiguration();
    } else if (inVirtualKey == XPLM_VK_F1) {
        // F1 - Help
        ShowHelp();
    }
}
```

### Version Checking

```c
int CheckSDKVersion() {
    // Check if we're running on compatible SDK version
    if (kXPLM_Version < 410) {  // Require SDK 4.1.0 or later
        XPLMDebugString("This plugin requires X-Plane SDK 4.1.0 or later\n");
        return 0;
    }
  
    // Check X-Plane version at runtime
    int xplaneVersion, xplmVersion;
    XPLMHostApplicationID hostID;
    XPLMGetVersions(&xplaneVersion, &xplmVersion, &hostID);
  
    if (xplmVersion < 410) {
        XPLMDebugString("This plugin requires XPLM 4.1.0 or later\n");
        return 0;
    }
  
    return 1;
}
```

### Plugin Communication

```c
// Inter-plugin communication
void SendMessageToOtherPlugin() {
    XPLMPluginID otherPlugin = XPLMFindPluginBySignature("com.example.other");
  
    if (otherPlugin != XPLM_NO_PLUGIN_ID) {
        // Send custom message (use values >= 0x01000000)
        int customMessage = 0x01000001;
        void* messageData = GetMessageData();
      
        XPLMSendMessageToPlugin(otherPlugin, customMessage, messageData);
    }
}

// Message handling
void XPluginReceiveMessage(XPLMPluginID inFrom, int inMsg, void* inParam) {
    if (inMsg >= 0x01000000) {
        // Custom plugin message
        HandleCustomMessage(inFrom, inMsg, inParam);
    } else {
        // Standard X-Plane messages
        switch (inMsg) {
            case XPLM_MSG_PLANE_LOADED:
                OnPlaneLoaded((int)(intptr_t)inParam);
                break;
            case XPLM_MSG_AIRPORT_LOADED:
                OnAirportLoaded();
                break;
        }
    }
}
```
