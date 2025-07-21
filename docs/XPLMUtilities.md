# XPLMUtilities.h - X-Plane 12 SDK Utilities Documentation

## Overview

The XPLMUtilities.h header provides essential utility functions for X-Plane 12 plugin development, including file system operations, command management, debugging utilities, system information access, and various helper functions. This module serves as the Swiss Army knife of the X-Plane SDK, offering frequently needed functionality for plugin development.

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [File System Operations](#file-system-operations)
- [Command System](#command-system)
- [Debugging and Logging](#debugging-and-logging)
- [System Information](#system-information)
- [Host Application Info](#host-application-info)
- [Language and Localization](#language-and-localization)
- [Feature Detection](#feature-detection)
- [Error Handling](#error-handling)
- [Key and Mouse Input](#key-and-mouse-input)
- [Implementation Guidelines](#implementation-guidelines)
- [Best Practices](#best-practices)
- [Common Use Cases](#common-use-cases)

## Architecture Overview

XPLMUtilities.h provides foundational utilities that support all aspects of plugin development:

- **File System Integration**: Cross-platform file operations with X-Plane path conventions
- **Command Infrastructure**: Create and execute X-Plane commands programmatically
- **Development Tools**: Debugging, logging, and diagnostic capabilities
- **System Introspection**: Runtime information about X-Plane and host system
- **Cross-Platform Services**: Unified APIs that abstract platform differences

## File System Operations

### Path Types and Constants

```c
enum {
    xplm_File_Unknown = 0,    // Unknown file type
    xplm_File_TypeCode = 1,   // File type code
    xplm_File_CreatorCode = 2 // File creator code (macOS)
};
typedef int XPLMFileType;
```

### Directory Navigation

#### XPLMGetSystemPath

```c
void XPLMGetSystemPath(char* outSystemPath);
```

**Purpose**: Retrieves the absolute path to X-Plane's installation directory.

**Parameters**:

- `outSystemPath`: Buffer to receive the system path (minimum 256 characters)

**Path Format**: Uses platform-native path separators and follows the `XPLM_USE_NATIVE_PATHS` feature setting.

**Implementation Example**:

```c
char xplaneRoot[512];
XPLMGetSystemPath(xplaneRoot);

// Build paths to X-Plane resources
char aircraftPath[768];
sprintf(aircraftPath, "%s%sAircraft%s", xplaneRoot, 
        XPLMGetDirectorySeparator(), XPLMGetDirectorySeparator());

// Access custom scenery
char sceneryPath[768];
sprintf(sceneryPath, "%s%sCustom Scenery%s", xplaneRoot,
        XPLMGetDirectorySeparator(), XPLMGetDirectorySeparator());
```

#### XPLMGetPrefsPath

```c
void XPLMGetPrefsPath(char* outPrefsPath);
```

**Purpose**: Returns the path to X-Plane's preferences directory where plugins should store configuration files.

**Usage Pattern**:

```c
char prefsPath[512];
XPLMGetPrefsPath(prefsPath);

// Create plugin-specific preferences file
char pluginPrefs[768];
sprintf(pluginPrefs, "%s%smyplugin_config.txt", prefsPath, XPLMGetDirectorySeparator());

// Save configuration
FILE* configFile = fopen(pluginPrefs, "w");
if (configFile) {
    fprintf(configFile, "setting1=value1\n");
    fprintf(configFile, "setting2=value2\n");
    fclose(configFile);
}
```

#### XPLMGetDirectorySeparator

```c
const char* XPLMGetDirectorySeparator(void);
```

**Purpose**: Returns the platform-appropriate directory separator character.

**Return Values**:

- Windows: `"\\"` (backslash)
- macOS/Linux: `"/"` (forward slash)

**Best Practice**: Always use this function instead of hardcoding separators.

### File Operations

#### XPLMExtractFileAndPath

```c
void XPLMExtractFileAndPath(const char* inFullPath, char* outFileName, char* outPath);
```

**Purpose**: Separates a full file path into filename and directory components.

**Parameters**:

- `inFullPath`: Full path to separate
- `outFileName`: Buffer for filename (can be NULL)
- `outPath`: Buffer for directory path (can be NULL)

**Implementation**:

```c
const char* fullPath = "/Users/pilot/X-Plane 12/Aircraft/Laminar Research/Cessna 172SP/Cessna_172SP.acf";
char filename[256];
char directory[512];

XPLMExtractFileAndPath(fullPath, filename, directory);
// filename = "Cessna_172SP.acf"
// directory = "/Users/pilot/X-Plane 12/Aircraft/Laminar Research/Cessna 172SP/"

// Use just filename
XPLMExtractFileAndPath(fullPath, filename, NULL);

// Use just directory
XPLMExtractFileAndPath(fullPath, NULL, directory);
```

#### XPLMGetFileTypeAndCreator (macOS Only)

```c
int XPLMGetFileTypeAndCreator(const char* inFilePath, XPLMFileType inFileType, void* outTypeCode);
```

**Purpose**: Retrieves macOS file type and creator codes (legacy functionality).

**Note**: Primarily for backward compatibility. Modern plugins should use file extensions.

## Command System

The command system allows plugins to create, find, and execute X-Plane commands programmatically.

### Command References

```c
typedef void* XPLMCommandRef;  // Opaque command handle
```

### Command Discovery

#### XPLMFindCommand

```c
XPLMCommandRef XPLMFindCommand(const char* inName);
```

**Purpose**: Locates an existing X-Plane or plugin command by name.

**Parameters**:

- `inName`: Command name string (e.g., "sim/lights/landing_lights_toggle")

**Return Value**: Command reference, or NULL if command doesn't exist.

**Common X-Plane Commands**:

```c
// Aircraft systems
XPLMCommandRef landingLights = XPLMFindCommand("sim/lights/landing_lights_toggle");
XPLMCommandRef parkingBrake = XPLMFindCommand("sim/flight_controls/brakes_toggle_max");
XPLMCommandRef mixtureFull = XPLMFindCommand("sim/engines/mixture_max");

// Flight controls
XPLMCommandRef flapUp = XPLMFindCommand("sim/flight_controls/flaps_up");
XPLMCommandRef flapDown = XPLMFindCommand("sim/flight_controls/flaps_down");
XPLMCommandRef gearToggle = XPLMFindCommand("sim/flight_controls/landing_gear_toggle");

// Autopilot
XPLMCommandRef apToggle = XPLMFindCommand("sim/autopilot/servos_toggle");
XPLMCommandRef hdgHold = XPLMFindCommand("sim/autopilot/heading_hold");
```

### Command Creation

#### XPLMCreateCommand

```c
XPLMCommandRef XPLMCreateCommand(const char* inName, const char* inDescription);
```

**Purpose**: Creates a new command that other plugins and X-Plane can execute.

**Parameters**:

- `inName`: Unique command identifier (use reverse DNS notation)
- `inDescription`: Human-readable description for UI display

**Best Practices**:

```c
// GOOD: Use company/plugin namespace
XPLMCommandRef myCommand = XPLMCreateCommand("myplugin/systems/toggle_custom_system",
                                           "Toggle Custom System On/Off");

// GOOD: Descriptive command names
XPLMCommandRef saveState = XPLMCreateCommand("myplugin/file/save_aircraft_state",
                                           "Save Current Aircraft State");

// BAD: Generic names that could conflict
XPLMCommandRef badCommand = XPLMCreateCommand("toggle", "Toggle something");
```

### Command Execution

#### XPLMCommandOnce

```c
void XPLMCommandOnce(XPLMCommandRef inCommand);
```

**Purpose**: Executes a command once (begin immediately followed by end).

**Usage**:

```c
// Toggle landing lights
XPLMCommandRef landingLights = XPLMFindCommand("sim/lights/landing_lights_toggle");
if (landingLights) {
    XPLMCommandOnce(landingLights);
}

// Execute custom command
XPLMCommandOnce(myCustomCommand);
```

#### XPLMCommandBegin

```c
void XPLMCommandBegin(XPLMCommandRef inCommand);
```

**Purpose**: Starts command execution (press and hold behavior).

#### XPLMCommandEnd

```c
void XPLMCommandEnd(XPLMCommandRef inCommand);
```

**Purpose**: Ends command execution.

**Important**: Always balance `XPLMCommandBegin()` and `XPLMCommandEnd()` calls.

**Hold-Type Commands**:

```c
// Start holding a command (e.g., elevator input)
XPLMCommandRef elevatorUp = XPLMFindCommand("sim/flight_controls/pitch_up");
XPLMCommandBegin(elevatorUp);

// ... command remains active ...

// Stop holding the command
XPLMCommandEnd(elevatorUp);
```

### Command Handlers

#### XPLMCommandCallback_f

```c
typedef int (*XPLMCommandCallback_f)(XPLMCommandRef inCommand,
                                    XPLMCommandPhase inPhase,
                                    void* inRefcon);

enum {
    xplm_CommandBegin = 0,    // Command started
    xplm_CommandContinue = 1, // Command continuing (held)
    xplm_CommandEnd = 2       // Command ended
};
typedef int XPLMCommandPhase;
```

#### XPLMRegisterCommandHandler

```c
void XPLMRegisterCommandHandler(XPLMCommandRef inCommand,
                              XPLMCommandCallback_f inHandler,
                              int inBefore,
                              void* inRefcon);
```

**Purpose**: Registers a callback function to handle command execution.

**Parameters**:

- `inCommand`: Command to handle
- `inHandler`: Callback function
- `inBefore`: 1 = call before other handlers, 0 = call after
- `inRefcon`: Reference pointer passed to callback

**Command Handler Implementation**:

```c
int MyCommandHandler(XPLMCommandRef inCommand, XPLMCommandPhase inPhase, void* inRefcon) {
    switch (inPhase) {
        case xplm_CommandBegin:
            XPLMDebugString("Command started\n");
            StartCustomAction();
            return 1; // Continue processing (let other handlers run)
          
        case xplm_CommandContinue:
            ContinueCustomAction();
            return 1;
          
        case xplm_CommandEnd:
            XPLMDebugString("Command ended\n");
            EndCustomAction();
            return 0; // Stop processing (don't let other handlers run)
    }
    return 1;
}

// Register handler during plugin enable
void XPluginEnable(void) {
    XPLMCommandRef myCmd = XPLMCreateCommand("myplugin/test_command", "Test Command");
    XPLMRegisterCommandHandler(myCmd, MyCommandHandler, 1, NULL);
}
```

#### XPLMUnregisterCommandHandler

```c
void XPLMUnregisterCommandHandler(XPLMCommandRef inCommand,
                                XPLMCommandCallback_f inHandler,
                                int inBefore,
                                void* inRefcon);
```

**Purpose**: Removes a previously registered command handler.

**Critical**: Always unregister handlers during plugin disable to prevent crashes.

```c
void XPluginDisable(void) {
    XPLMUnregisterCommandHandler(myCommand, MyCommandHandler, 1, NULL);
}
```

## Debugging and Logging

### Debug Output

#### XPLMDebugString

```c
void XPLMDebugString(const char* inString);
```

**Purpose**: Outputs debug messages to X-Plane's log file and console.

**Output Destinations**:

- `Log.txt` file in X-Plane directory
- Console window (if visible)
- IDE output (during development)

**Usage Patterns**:

```c
// Simple messages
XPLMDebugString("Plugin initialized successfully\n");

// Formatted output
char debugMsg[512];
sprintf(debugMsg, "Dataref value: %.2f\n", currentValue);
XPLMDebugString(debugMsg);

// Error reporting
XPLMDebugString("ERROR: Failed to load configuration file\n");

// Performance timing
clock_t start = clock();
PerformExpensiveOperation();
clock_t end = clock();
sprintf(debugMsg, "Operation took %.2f ms\n", 
        ((double)(end - start) / CLOCKS_PER_SEC) * 1000);
XPLMDebugString(debugMsg);
```

### Error Callbacks (XPLM200+)

#### XPLMErrorCallback_f

```c
typedef void (*XPLMErrorCallback_f)(const char* inMessage);
```

#### XPLMSetErrorCallback

```c
void XPLMSetErrorCallback(XPLMErrorCallback_f inCallback);
```

**Purpose**: Registers a callback to handle X-Plane SDK errors.

**Implementation**:

```c
void MyErrorHandler(const char* inMessage) {
    char errorMsg[1024];
    sprintf(errorMsg, "XPLM ERROR: %s\n", inMessage);
    XPLMDebugString(errorMsg);
  
    // Log to file
    FILE* errorLog = fopen("plugin_errors.txt", "a");
    if (errorLog) {
        fprintf(errorLog, "[%s] %s\n", GetCurrentTimestamp(), inMessage);
        fclose(errorLog);
    }
}

// Register during plugin start (remove in production)
#ifdef DEBUG
XPLMSetErrorCallback(MyErrorHandler);
#endif
```

## System Information

### Version Information

#### XPLMGetVersions

```c
void XPLMGetVersions(int* outXPlaneVersion, int* outXPLMVersion, XPLMHostApplicationID* outHostID);
```

**Purpose**: Retrieves version information for X-Plane and the SDK.

**Parameters**:

- `outXPlaneVersion`: X-Plane version (e.g., 12100 for version 12.1.0)
- `outXPLMVersion`: SDK version (e.g., 410 for SDK 4.1.0)
- `outHostID`: Host application identifier

**Host Application IDs**:

```c
enum {
    xplm_Host_Unknown = 0,
    xplm_Host_XPlane = 1      // Standard X-Plane
};
typedef int XPLMHostApplicationID;
```

**Version Check Example**:

```c
int CheckCompatibility(void) {
    int xplaneVer, sdkVer;
    XPLMHostApplicationID hostID;
  
    XPLMGetVersions(&xplaneVer, &sdkVer, &hostID);
  
    // Require X-Plane 12.0.0 or later
    if (xplaneVer < 12000) {
        XPLMDebugString("ERROR: This plugin requires X-Plane 12.0.0 or later\n");
        return 0;
    }
  
    // Require SDK 4.0.0 or later
    if (sdkVer < 400) {
        XPLMDebugString("ERROR: This plugin requires SDK 4.0.0 or later\n");
        return 0;
    }
  
    char versionMsg[256];
    sprintf(versionMsg, "Running on X-Plane %d.%d.%d with SDK %d.%d.%d\n",
            xplaneVer / 10000, (xplaneVer / 100) % 100, xplaneVer % 100,
            sdkVer / 100, (sdkVer / 10) % 10, sdkVer % 10);
    XPLMDebugString(versionMsg);
  
    return 1;
}
```

## Language and Localization

### XPLMGetLanguage

```c
XPLMLanguageCode XPLMGetLanguage(void);
```

**Purpose**: Returns the currently selected language in X-Plane.

**Language Codes**:

```c
enum {
    xplm_Language_Unknown = 0,
    xplm_Language_English = 1,
    xplm_Language_French = 2,
    xplm_Language_German = 3,
    xplm_Language_Italian = 4,
    xplm_Language_Spanish = 5,
    xplm_Language_Korean = 6,
    xplm_Language_Russian = 7,
    xplm_Language_Greek = 8,
    xplm_Language_Japanese = 9,
    xplm_Language_Chinese = 10
};
typedef int XPLMLanguageCode;
```

**Localization Implementation**:

```c
const char* GetLocalizedString(const char* key) {
    XPLMLanguageCode lang = XPLMGetLanguage();
  
    if (strcmp(key, "AUTOPILOT_ON") == 0) {
        switch (lang) {
            case xplm_Language_French:  return "Pilote automatique activé";
            case xplm_Language_German:  return "Autopilot eingeschaltet";
            case xplm_Language_Spanish: return "Piloto automático activado";
            default:                    return "Autopilot On";
        }
    }
  
    // Default to English
    return key;
}
```

## Feature Detection

### XPLMHasFeature

```c
int XPLMHasFeature(const char* inFeature);
```

**Purpose**: Checks if a specific SDK feature is available in the current X-Plane version.

**Common Features**:

```c
// Check for modern features before using them
if (XPLMHasFeature("XPLM_USE_NATIVE_PATHS")) {
    XPLMEnableFeature("XPLM_USE_NATIVE_PATHS", 1);
}

if (XPLMHasFeature("XPLM_WANTS_REFLECTIONS")) {
    XPLMEnableFeature("XPLM_WANTS_REFLECTIONS", 1);
}

// Check for dataref features (SDK 4.0+)
if (XPLMHasFeature("XPLM400")) {
    // Use SDK 4.0 features like dataref introspection
    UseModernDatarefAPI();
}
```

### XPLMEnableFeature

```c
void XPLMEnableFeature(const char* inFeature, int inEnable);
```

**Purpose**: Enables or disables SDK features.

**Feature Management**:

```c
void ConfigureSDKFeatures(void) {
    // Enable native paths for consistent cross-platform behavior
    if (XPLMHasFeature("XPLM_USE_NATIVE_PATHS")) {
        XPLMEnableFeature("XPLM_USE_NATIVE_PATHS", 1);
        XPLMDebugString("Native paths enabled\n");
    }
  
    // Enable modern widgets
    if (XPLMHasFeature("XPLM_USE_NATIVE_WIDGET_WINDOWS")) {
        XPLMEnableFeature("XPLM_USE_NATIVE_WIDGET_WINDOWS", 1);
        XPLMDebugString("Native widget windows enabled\n");
    }
  
    // Enable reflection support if available
    if (XPLMHasFeature("XPLM_WANTS_REFLECTIONS")) {
        XPLMEnableFeature("XPLM_WANTS_REFLECTIONS", 1);
    }
}
```

## Key and Mouse Input

### Hot Key Registration

#### XPLMHotKeyCallback_f

```c
typedef void (*XPLMHotKeyCallback_f)(void* inRefcon);
```

#### XPLMRegisterHotKey

```c
XPLMHotKeyID XPLMRegisterHotKey(char inVirtualKey,
                               XPLMKeyFlags inFlags,
                               const char* inDescription,
                               XPLMHotKeyCallback_f inCallback,
                               void* inRefcon);
```

**Purpose**: Registers a global hot key that works regardless of X-Plane's current focus.

**Hot Key Implementation**:

```c
typedef struct {
    int configWindowVisible;
} PluginState_t;

static PluginState_t gPluginState = {0};

void ConfigHotKeyCallback(void* inRefcon) {
    PluginState_t* state = (PluginState_t*)inRefcon;
  
    // Toggle configuration window
    state->configWindowVisible = !state->configWindowVisible;
  
    if (state->configWindowVisible) {
        ShowConfigurationWindow();
    } else {
        HideConfigurationWindow();
    }
}

// Register F12 as configuration hotkey
XPLMHotKeyID gConfigHotKey = XPLMRegisterHotKey(XPLM_VK_F12, 
                                                xplm_DownFlag,
                                                "Open Plugin Configuration",
                                                ConfigHotKeyCallback,
                                                &gPluginState);
```

#### XPLMUnregisterHotKey

```c
void XPLMUnregisterHotKey(XPLMHotKeyID inHotKey);
```

**Cleanup**:

```c
void XPluginDisable(void) {
    if (gConfigHotKey) {
        XPLMUnregisterHotKey(gConfigHotKey);
        gConfigHotKey = NULL;
    }
}
```

### Mouse and Key Sniffer

#### XPLMKeySniffer_f

```c
typedef int (*XPLMKeySniffer_f)(char inChar,
                               XPLMKeyFlags inFlags,
                               char inVirtualKey,
                               void* inRefcon);
```

#### XPLMRegisterKeySniffer

```c
int XPLMRegisterKeySniffer(XPLMKeySniffer_f inCallback,
                          int inBeforeWindows,
                          void* inRefcon);
```

**Purpose**: Intercepts all keyboard input before it reaches X-Plane or other plugins.

**Key Sniffer Implementation**:

```c
int MyKeySniffer(char inChar, XPLMKeyFlags inFlags, char inVirtualKey, void* inRefcon) {
    // Log all keystrokes for debugging
    if (inFlags & xplm_DownFlag) {
        char logMsg[256];
        sprintf(logMsg, "Key pressed: char='%c' vkey=%d flags=%d\n", 
                inChar, inVirtualKey, inFlags);
        XPLMDebugString(logMsg);
    }
  
    // Check for special key combinations
    if ((inFlags & xplm_ControlFlag) && (inVirtualKey == XPLM_VK_Q)) {
        // Ctrl+Q - emergency shutdown
        EmergencyShutdown();
        return 1; // Consume the keystroke
    }
  
    return 0; // Let other handlers process
}

// Register key sniffer
int XPluginEnable(void) {
    XPLMRegisterKeySniffer(MyKeySniffer, 1, NULL);
    return 1;
}
```

#### XPLMUnregisterKeySniffer

```c
int XPLMUnregisterKeySniffer(XPLMKeySniffer_f inCallback,
                            int inBeforeWindows,
                            void* inRefcon);
```

## Implementation Guidelines

### Plugin Lifecycle Management

```c
// Global state
static XPLMCommandRef gMyCommands[10];
static XPLMHotKeyID gHotKeys[5];
static int gNumCommands = 0;
static int gNumHotKeys = 0;

int XPluginStart(char* outName, char* outSig, char* outDesc) {
    // Set basic info
    strcpy(outName, "My Utility Plugin");
    strcpy(outSig, "com.example.utilplugin");
    strcpy(outDesc, "Demonstrates X-Plane utilities");
  
    // Check compatibility
    if (!CheckCompatibility()) {
        return 0;
    }
  
    // Configure SDK features
    ConfigureSDKFeatures();
  
    // Create commands
    gMyCommands[gNumCommands++] = XPLMCreateCommand("myplugin/toggle_feature",
                                                   "Toggle Main Feature");
  
    return 1;
}

int XPluginEnable(void) {
    // Register callbacks
    for (int i = 0; i < gNumCommands; i++) {
        XPLMRegisterCommandHandler(gMyCommands[i], MyCommandHandler, 1, NULL);
    }
  
    // Register hot keys
    gHotKeys[gNumHotKeys++] = XPLMRegisterHotKey(XPLM_VK_F10, xplm_DownFlag,
                                                "Plugin Menu", MenuHotKey, NULL);
  
    return 1;
}

void XPluginDisable(void) {
    // Unregister callbacks
    for (int i = 0; i < gNumCommands; i++) {
        XPLMUnregisterCommandHandler(gMyCommands[i], MyCommandHandler, 1, NULL);
    }
  
    // Unregister hot keys
    for (int i = 0; i < gNumHotKeys; i++) {
        if (gHotKeys[i]) {
            XPLMUnregisterHotKey(gHotKeys[i]);
            gHotKeys[i] = NULL;
        }
    }
    gNumHotKeys = 0;
}

void XPluginStop(void) {
    // Final cleanup
    gNumCommands = 0;
}
```

## Best Practices

### Error Handling and Robustness

```c
// Safe command execution
void SafeExecuteCommand(const char* commandName) {
    XPLMCommandRef cmd = XPLMFindCommand(commandName);
    if (cmd) {
        XPLMCommandOnce(cmd);
    } else {
        char errorMsg[512];
        sprintf(errorMsg, "Command not found: %s\n", commandName);
        XPLMDebugString(errorMsg);
    }
}

// Safe file operations
int SavePluginState(const char* filename) {
    char prefsPath[512];
    char fullPath[768];
  
    XPLMGetPrefsPath(prefsPath);
    sprintf(fullPath, "%s%s%s", prefsPath, XPLMGetDirectorySeparator(), filename);
  
    FILE* file = fopen(fullPath, "w");
    if (!file) {
        char errorMsg[1024];
        sprintf(errorMsg, "Failed to save state to: %s\n", fullPath);
        XPLMDebugString(errorMsg);
        return 0;
    }
  
    // Write state data
    fprintf(file, "version=1.0\n");
    fprintf(file, "enabled=%d\n", gPluginEnabled);
  
    fclose(file);
    return 1;
}
```

### Resource Management

```c
// Command reference management
typedef struct {
    XPLMCommandRef ref;
    const char* name;
    const char* description;
    XPLMCommandCallback_f handler;
} CommandInfo_t;

static CommandInfo_t gCommandRegistry[] = {
    {NULL, "myplugin/systems/power_on", "Power On Systems", PowerOnHandler},
    {NULL, "myplugin/systems/power_off", "Power Off Systems", PowerOffHandler},
    {NULL, "myplugin/config/show", "Show Configuration", ConfigShowHandler},
    {NULL, NULL, NULL, NULL} // Sentinel
};

void RegisterAllCommands(void) {
    for (int i = 0; gCommandRegistry[i].name != NULL; i++) {
        gCommandRegistry[i].ref = XPLMCreateCommand(gCommandRegistry[i].name,
                                                   gCommandRegistry[i].description);
        if (gCommandRegistry[i].ref) {
            XPLMRegisterCommandHandler(gCommandRegistry[i].ref,
                                     gCommandRegistry[i].handler, 1, NULL);
        }
    }
}

void UnregisterAllCommands(void) {
    for (int i = 0; gCommandRegistry[i].name != NULL; i++) {
        if (gCommandRegistry[i].ref) {
            XPLMUnregisterCommandHandler(gCommandRegistry[i].ref,
                                       gCommandRegistry[i].handler, 1, NULL);
        }
    }
}
```

## Common Use Cases

### Configuration Management

```c
typedef struct {
    int version;
    int enabled;
    float sensitivity;
    char lastAircraft[256];
} PluginConfig_t;

int LoadConfiguration(PluginConfig_t* config) {
    char prefsPath[512];
    char configPath[768];
  
    XPLMGetPrefsPath(prefsPath);
    sprintf(configPath, "%s%smyplugin.cfg", prefsPath, XPLMGetDirectorySeparator());
  
    FILE* file = fopen(configPath, "r");
    if (!file) {
        // Create default configuration
        config->version = 1;
        config->enabled = 1;
        config->sensitivity = 1.0f;
        strcpy(config->lastAircraft, "");
        return SaveConfiguration(config);
    }
  
    char line[256];
    while (fgets(line, sizeof(line), file)) {
        if (strncmp(line, "version=", 8) == 0) {
            config->version = atoi(line + 8);
        } else if (strncmp(line, "enabled=", 8) == 0) {
            config->enabled = atoi(line + 8);
        } else if (strncmp(line, "sensitivity=", 12) == 0) {
            config->sensitivity = atof(line + 12);
        } else if (strncmp(line, "lastAircraft=", 13) == 0) {
            strncpy(config->lastAircraft, line + 13, 255);
            // Remove newline
            config->lastAircraft[strcspn(config->lastAircraft, "\n")] = '\0';
        }
    }
  
    fclose(file);
    return 1;
}

int SaveConfiguration(const PluginConfig_t* config) {
    char prefsPath[512];
    char configPath[768];
  
    XPLMGetPrefsPath(prefsPath);
    sprintf(configPath, "%s%smyplugin.cfg", prefsPath, XPLMGetDirectorySeparator());
  
    FILE* file = fopen(configPath, "w");
    if (!file) {
        return 0;
    }
  
    fprintf(file, "version=%d\n", config->version);
    fprintf(file, "enabled=%d\n", config->enabled);
    fprintf(file, "sensitivity=%.2f\n", config->sensitivity);
    fprintf(file, "lastAircraft=%s\n", config->lastAircraft);
  
    fclose(file);
    return 1;
}
```

### Command System Integration

```c
// Create context-sensitive commands
void SetupAircraftCommands(void) {
    // Get aircraft information
    XPLMDataRef aircraftName = XPLMFindDataRef("sim/aircraft/view/acf_descrip");
    if (aircraftName) {
        char name[256];
        XPLMGetDatab(aircraftName, name, 0, sizeof(name));
      
        // Create aircraft-specific commands
        char commandName[512];
        char commandDesc[512];
      
        sprintf(commandName, "myplugin/%s/quick_start", name);
        sprintf(commandDesc, "Quick Start for %s", name);
      
        XPLMCommandRef quickStart = XPLMCreateCommand(commandName, commandDesc);
        XPLMRegisterCommandHandler(quickStart, QuickStartHandler, 1, name);
    }
}

// Universal command handler with context
int UniversalCommandHandler(XPLMCommandRef inCommand, XPLMCommandPhase inPhase, void* inRefcon) {
    if (inPhase != xplm_CommandBegin) return 1;
  
    const char* context = (const char*)inRefcon;
  
    if (context && strcmp(context, "emergency") == 0) {
        HandleEmergencyProcedure();
    } else if (context && strcmp(context, "startup") == 0) {
        HandleStartupProcedure();
    } else {
        HandleGenericCommand();
    }
  
    return 1;
}
```
