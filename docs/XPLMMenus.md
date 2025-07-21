# XPLMMenus API Documentation

## Overview

The XPLMMenus API provides comprehensive functionality for creating and managing menus in X-Plane's menu bar. This system allows plugins to integrate seamlessly with X-Plane's user interface by creating custom menus, submenus, and menu items with callbacks or command bindings.

## Key Features

- **Sandboxed Menu System**: Each plugin's menus are isolated from other plugins
- **Unicode Support**: Full UTF-8 text support for international characters
- **Command Integration**: Menu items can trigger X-Plane commands with keyboard shortcuts
- **Visual States**: Support for checked/unchecked and enabled/disabled menu states
- **Hierarchical Structure**: Support for submenus and separators

## Architecture

### Menu Sandboxing

Menus are completely sandboxed between plugins. No plugin can access or modify menus created by another plugin. All menu indices are relative to your plugin's menus only - if your plugin creates two submenus, they will always have indices 0 and 1 regardless of what other plugins do.

### Menu Item Handling

Menu items can be handled in two ways:

1. **Callback-based**: Trigger a custom `XPLMMenuHandler_f` function
2. **Command-based**: Execute an X-Plane command (displays keyboard shortcuts)

## Data Types

### XPLMMenuID

```c
typedef void * XPLMMenuID;
```

Opaque handle representing a menu. Used to identify menus for all operations.

### XPLMMenuCheck

```c
enum {
    xplm_Menu_NoCheck    = 0,  // No check indicator
    xplm_Menu_Unchecked  = 1,  // Unchecked (unlit) indicator
    xplm_Menu_Checked    = 2   // Checked (lit) indicator
};
typedef int XPLMMenuCheck;
```

Defines the visual check state for menu items. Check marks appear as lights in X-Plane.

### XPLMMenuHandler_f

```c
typedef void (* XPLMMenuHandler_f)(
    void * inMenuRef,    // Menu reference data
    void * inItemRef     // Item reference data
);
```

Callback function type for handling menu item selections.

## Core Functions

### Menu Discovery

#### XPLMFindPluginsMenu

```c
XPLM_API XPLMMenuID XPLMFindPluginsMenu(void);
```

Returns the ID of the main "Plugins" menu that X-Plane creates automatically.

**Use Cases:**

- Adding plugin-specific menu items to the standard Plugins menu
- Creating submenus under the Plugins menu

**Example:**

```c
XPLMMenuID pluginsMenu = XPLMFindPluginsMenu();
XPLMMenuID mySubMenu = XPLMCreateMenu("My Plugin", pluginsMenu, 0, MyMenuHandler, NULL);
```

#### XPLMFindAircraftMenu (XPLM300+)

```c
XPLM_API XPLMMenuID XPLMFindAircraftMenu(void);
```

Returns the aircraft-specific menu for the currently loaded aircraft.

**Important Notes:**

- Only plugins loaded with the current aircraft can access this menu
- Returns NULL for plugins not associated with the current aircraft
- Menu remains hidden until populated with items

**Use Cases:**

- Aircraft-specific commands and settings
- Systems management for complex aircraft

### Menu Creation and Management

#### XPLMCreateMenu

```c
XPLM_API XPLMMenuID XPLMCreateMenu(
    const char *      inName,          // Menu title
    XPLMMenuID        inParentMenu,    // Parent menu (NULL for menu bar)
    int               inParentItem,    // Parent item index
    XPLMMenuHandler_f inHandler,       // Callback function (can be NULL)
    void *            inMenuRef        // Reference data for callback
);
```

Creates a new menu or submenu.

**Parameters:**

- `inName`: Menu title (required even for submenus)
- `inParentMenu`: Parent menu ID, or NULL to create in menu bar
- `inParentItem`: Index of parent item to attach submenu to
- `inHandler`: Callback function for menu items (NULL if no callbacks needed)
- `inMenuRef`: User data passed to callback function

**Returns:** Menu ID, or NULL on failure

**Implementation Guidelines:**

```c
// Create main menu in menu bar
XPLMMenuID mainMenu = XPLMCreateMenu(
    "My Plugin", 
    NULL, 0,           // No parent (menu bar)
    MyMenuHandler, 
    NULL
);

// Create submenu
XPLMMenuID subMenu = XPLMCreateMenu(
    "Settings", 
    mainMenu, 0,       // Attach to first item of main menu
    MyMenuHandler, 
    NULL
);
```

#### XPLMDestroyMenu

```c
XPLM_API void XPLMDestroyMenu(XPLMMenuID inMenuID);
```

Destroys a menu and all its contents. Use sparingly - typically only needed for dynamic menu rebuilding.

#### XPLMClearAllMenuItems

```c
XPLM_API void XPLMClearAllMenuItems(XPLMMenuID inMenuID);
```

Removes all items from a menu, allowing complete reconstruction.

**Use Cases:**

- Dynamic menu content based on aircraft state
- Settings menus that change based on configuration

### Menu Item Operations

#### XPLMAppendMenuItem

```c
XPLM_API int XPLMAppendMenuItem(
    XPLMMenuID    inMenu,                    // Target menu
    const char *  inItemName,                // Item text
    void *        inItemRef,                 // Reference data for callback
    int           inDeprecatedAndIgnored     // Ignored parameter
);
```

Adds a menu item that triggers the menu's callback handler.

**Returns:** Item index (>= 0) or negative on failure

**Example:**

```c
int itemIndex = XPLMAppendMenuItem(
    myMenu, 
    "Toggle Feature", 
    (void*)FEATURE_TOGGLE_ID, 
    0
);

// In menu handler:
void MyMenuHandler(void* inMenuRef, void* inItemRef) {
    int command = (int)(intptr_t)inItemRef;
    switch(command) {
        case FEATURE_TOGGLE_ID:
            ToggleMyFeature();
            break;
    }
}
```

#### XPLMAppendMenuItemWithCommand (XPLM300+)

```c
XPLM_API int XPLMAppendMenuItemWithCommand(
    XPLMMenuID     inMenu,               // Target menu
    const char *   inItemName,           // Item text
    XPLMCommandRef inCommandToExecute    // Command to execute
);
```

Adds a menu item that executes an X-Plane command directly.

**Advantages:**

- Automatically displays keyboard shortcuts
- Integrates with X-Plane's command system
- User can assign custom key bindings

**Example:**

```c
XPLMCommandRef myCommand = XPLMCreateCommand(
    "myplugin/toggle_feature", 
    "Toggle My Feature"
);

XPLMAppendMenuItemWithCommand(
    myMenu, 
    "Toggle Feature", 
    myCommand
);
```

#### XPLMAppendMenuSeparator

```c
XPLM_API void XPLMAppendMenuSeparator(XPLMMenuID inMenu);
```

Adds a visual separator line to organize menu items.

### Menu Item Properties

#### XPLMSetMenuItemName

```c
XPLM_API void XPLMSetMenuItemName(
    XPLMMenuID    inMenu,                    // Menu containing item
    int           inIndex,                   // Item index
    const char *  inItemName,                // New item text
    int           inDeprecatedAndIgnored     // Ignored parameter
);
```

Changes the display text of an existing menu item.

#### XPLMCheckMenuItem

```c
XPLM_API void XPLMCheckMenuItem(
    XPLMMenuID    inMenu,    // Menu containing item
    int           index,     // Item index
    XPLMMenuCheck inCheck    // Check state
);
```

Sets the check mark state of a menu item.

#### XPLMCheckMenuItemState

```c
XPLM_API void XPLMCheckMenuItemState(
    XPLMMenuID    inMenu,     // Menu containing item
    int           index,      // Item index
    XPLMMenuCheck *outCheck   // Current check state
);
```

Retrieves the current check state of a menu item.

#### XPLMEnableMenuItem

```c
XPLM_API void XPLMEnableMenuItem(
    XPLMMenuID inMenu,    // Menu containing item
    int        index,     // Item index
    int        enabled    // 1 = enabled, 0 = disabled
);
```

Enables or disables a menu item. Disabled items appear grayed out.

#### XPLMRemoveMenuItem (XPLM210+)

```c
XPLM_API void XPLMRemoveMenuItem(
    XPLMMenuID inMenu,    // Menu containing item
    int        inIndex    // Item index to remove
);
```

Removes a single menu item. All subsequent items shift up by one index.

## Implementation Patterns

### Basic Menu Setup

```c
static XPLMMenuID gMainMenu = NULL;

// In XPluginStart
void CreateMenus() {
    // Find the plugins menu
    XPLMMenuID pluginsMenu = XPLMFindPluginsMenu();
  
    // Create submenu
    gMainMenu = XPLMCreateMenu("My Plugin", pluginsMenu, 0, MenuHandler, NULL);
  
    // Add menu items
    XPLMAppendMenuItem(gMainMenu, "Settings...", (void*)MENU_SETTINGS, 0);
    XPLMAppendMenuSeparator(gMainMenu);
    XPLMAppendMenuItem(gMainMenu, "About", (void*)MENU_ABOUT, 0);
}

void MenuHandler(void* inMenuRef, void* inItemRef) {
    int item = (int)(intptr_t)inItemRef;
  
    switch(item) {
        case MENU_SETTINGS:
            ShowSettingsDialog();
            break;
        case MENU_ABOUT:
            ShowAboutDialog();
            break;
    }
}
```

### Dynamic Menu Updates

```c
void UpdateMenuBasedOnAircraftState() {
    // Clear all existing items
    XPLMClearAllMenuItems(gMainMenu);
  
    // Check aircraft type
    int engineCount = XPLMGetDatai(gEngineCountRef);
  
    // Add appropriate menu items
    XPLMAppendMenuItem(gMainMenu, "Engine Controls", (void*)MENU_ENGINES, 0);
  
    if (engineCount > 1) {
        XPLMAppendMenuItem(gMainMenu, "Engine Sync", (void*)MENU_SYNC, 0);
    }
  
    // Update check states
    int featureEnabled = GetFeatureState();
    XPLMCheckMenuItem(gMainMenu, 0, 
        featureEnabled ? xplm_Menu_Checked : xplm_Menu_Unchecked);
}
```

### Command-Based Menu Items

```c
// Create commands first
XPLMCommandRef gToggleCommand = XPLMCreateCommand(
    "myplugin/toggle_autopilot", 
    "Toggle Autopilot"
);

// Register command handler
XPLMRegisterCommandHandler(gToggleCommand, CommandHandler, 1, NULL);

// Add to menu with automatic keyboard shortcut display
XPLMAppendMenuItemWithCommand(gMainMenu, "Toggle Autopilot", gToggleCommand);

int CommandHandler(XPLMCommandRef cmd, XPLMCommandPhase phase, void* refcon) {
    if (phase == xplm_CommandBegin) {
        ToggleAutopilot();
    }
    return 1;
}
```

## Best Practices

### Menu Organization

- Use clear, descriptive menu names
- Group related functionality with separators
- Keep menu hierarchies shallow (2-3 levels maximum)
- Use consistent naming conventions

### Performance Considerations

- Create menus once during plugin initialization
- Avoid frequent menu rebuilding
- Cache menu item indices for updates
- Use command-based items when possible for better integration

### User Experience

- Provide visual feedback with check marks for toggle states
- Disable menu items when functionality is unavailable
- Use standard menu conventions (About at bottom, etc.)
- Support both mouse and keyboard access

### Memory Management

- Store menu IDs in global variables for access across functions
- Clean up menus in XPluginDisable if needed
- Don't destroy menus unnecessarily

### Error Handling

```c
XPLMMenuID menu = XPLMCreateMenu("Test", parent, 0, handler, NULL);
if (menu == NULL) {
    XPLMDebugString("Failed to create menu\n");
    return 0;
}

int itemIndex = XPLMAppendMenuItem(menu, "Test Item", NULL, 0);
if (itemIndex < 0) {
    XPLMDebugString("Failed to add menu item\n");
}
```

## Integration with Other APIs

### Command System Integration

Menu items work seamlessly with XPLMUtilities command system:

```c
// Create command
XPLMCommandRef cmd = XPLMCreateCommand("plugin/action", "My Action");

// Add to menu - keyboard shortcuts automatically displayed
XPLMAppendMenuItemWithCommand(menu, "My Action", cmd);

// Handle command
XPLMRegisterCommandHandler(cmd, MyCommandHandler, 1, NULL);
```

### Dataref Integration

Update menu states based on simulator data:

```c
void UpdateMenuFromDatarefs() {
    int gearDown = XPLMGetDatai(gGearDeployRef);
    XPLMCheckMenuItem(gMainMenu, gearItemIndex, 
        gearDown ? xplm_Menu_Checked : xplm_Menu_Unchecked);
      
    int engineRunning = XPLMGetDatai(gEngineRunningRef);
    XPLMEnableMenuItem(gMainMenu, engineItemIndex, engineRunning);
}
```

## Version Compatibility

- **Base API**: Available in all SDK versions
- **XPLMFindAircraftMenu**: XPLM300+ (X-Plane 11+)
- **XPLMAppendMenuItemWithCommand**: XPLM300+ (X-Plane 11+)
- **XPLMRemoveMenuItem**: XPLM210+ (X-Plane 10+)

## Common Pitfalls

1. **Menu Index Tracking**: Remember that removing items shifts subsequent indices
2. **NULL Pointer Checks**: Always verify menu creation succeeded before using
3. **Handler Registration**: Ensure menu handlers are registered before menu creation
4. **Memory Leaks**: Don't repeatedly create menus without destroying old ones
5. **Case Sensitivity**: Menu names are case-sensitive for comparison

## Example: Complete Menu System

```c
// Global variables
static XPLMMenuID gMainMenu = NULL;
static XPLMCommandRef gToggleCmd = NULL;

// Menu item constants
#define MENU_SETTINGS 1
#define MENU_ABOUT 2

PLUGIN_API int XPluginStart(char* name, char* sig, char* desc) {
    // Initialize plugin
    strcpy(name, "Menu Example Plugin");
    strcpy(sig, "com.example.menu");
    strcpy(desc, "Demonstrates menu system usage");
  
    // Create command
    gToggleCmd = XPLMCreateCommand("example/toggle", "Toggle Feature");
  
    return 1;
}

PLUGIN_API int XPluginEnable(void) {
    // Create menu system
    XPLMMenuID pluginsMenu = XPLMFindPluginsMenu();
    gMainMenu = XPLMCreateMenu("Menu Example", pluginsMenu, 0, MenuHandler, NULL);
  
    if (gMainMenu != NULL) {
        // Add items
        XPLMAppendMenuItemWithCommand(gMainMenu, "Toggle Feature", gToggleCmd);
        XPLMAppendMenuSeparator(gMainMenu);
        XPLMAppendMenuItem(gMainMenu, "Settings...", (void*)MENU_SETTINGS, 0);
        XPLMAppendMenuItem(gMainMenu, "About", (void*)MENU_ABOUT, 0);
    }
  
    // Register command handler
    XPLMRegisterCommandHandler(gToggleCmd, CommandHandler, 1, NULL);
  
    return 1;
}

PLUGIN_API void XPluginDisable(void) {
    // Cleanup
    XPLMUnregisterCommandHandler(gToggleCmd, CommandHandler, 1, NULL);
}

PLUGIN_API void XPluginStop(void) {
    // Final cleanup
}

void MenuHandler(void* menuRef, void* itemRef) {
    int item = (int)(intptr_t)itemRef;
  
    switch(item) {
        case MENU_SETTINGS:
            XPLMDebugString("Settings menu selected\n");
            break;
        case MENU_ABOUT:
            XPLMDebugString("About menu selected\n");
            break;
    }
}

int CommandHandler(XPLMCommandRef cmd, XPLMCommandPhase phase, void* refcon) {
    if (phase == xplm_CommandBegin) {
        XPLMDebugString("Toggle command executed\n");
    }
    return 1;
}
```
