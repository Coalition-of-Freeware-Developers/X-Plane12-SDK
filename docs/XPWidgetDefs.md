# XPWidgetDefs.h - X-Plane 12 Widget Framework Core Definitions

## Overview

The `XPWidgetDefs.h` header file provides the core definitions, types, and constants for the X-Plane 12 Widget Framework. This is the foundation header that defines the fundamental concepts and structures used throughout the widget system.

## Architecture

The Widget Framework is a callback-driven UI system that allows plugins to create persistent, interactive user interface elements within X-Plane. Widgets are organized in hierarchical trees where parent widgets contain child widgets.

### Key Concepts

- **Widget Hierarchy**: Widgets are organized in parent-child relationships forming trees
- **Message-Driven**: Widgets communicate via messages dispatched through the hierarchy
- **Global Coordinates**: All coordinates are in global screen space (0,0 at bottom-left)
- **Property System**: Widgets store state via key-value properties
- **Callback Functions**: Widget behavior is implemented through callback functions

## Core Types and Definitions

### Platform Macros

The widget system uses platform-specific export macros:

```c
// Platform detection - define exactly one
#define APL 1   // macOS
#define IBM 1   // Windows  
#define LIN 1   // Linux

// Export macro for widget functions
WIDGET_API  // Platform-appropriate DLL export/import
```

### Widget Identity

#### `XPWidgetID`

```c
typedef void * XPWidgetID;
```

An opaque handle that uniquely identifies a widget instance. Use `0` to specify "no widget". This handle is returned when creating widgets and used for all subsequent widget operations.

**Usage Guidelines:**
- Always store widget IDs returned from creation functions
- Use `0` to indicate no parent widget or invalid widget
- Widget IDs remain valid until the widget is destroyed
- Do not cast or dereference widget IDs

#### `XPWidgetClass`

```c
typedef int XPWidgetClass;
#define xpWidgetClass_None   0
```

Identifies predefined widget types. Standard widget classes are defined in `XPStandardWidgets.h`.

### Widget Properties

#### `XPWidgetPropertyID`

```c
typedef int XPWidgetPropertyID;
```

32-bit identifiers for widget properties. Properties store arbitrary values (pointer-width) associated with widgets.

**Predefined Properties:**

| Property | ID | Purpose |
|----------|----|---------| 
| `xpProperty_Refcon` | 0 | Opaque reference value for client code |
| `xpProperty_Dragging` | 1 | Used by utilities for drag implementation |
| `xpProperty_DragXOff` | 2 | X-offset during dragging |
| `xpProperty_DragYOff` | 3 | Y-offset during dragging |
| `xpProperty_Hilited` | 4 | Whether widget is highlighted |
| `xpProperty_Object` | 5 | C++ object attachment |
| `xpProperty_Clip` | 6 | Enable OpenGL clipping to widget bounds |
| `xpProperty_Enabled` | 7 | Whether widget is enabled/disabled |

**Property ID Ranges:**
- 1-999: Reserved for widgets library
- 1000-9999: Standard widget classes (1000-1099 for class 0, etc.)
- 10000+: Available for plugin use (`xpProperty_UserStart`)

### Input Event Structures

#### `XPMouseState_t`

```c
typedef struct {
    int x;              // Mouse X coordinate (global screen space)
    int y;              // Mouse Y coordinate (global screen space) 
    int button;         // Mouse button (0 = left, right not yet supported)
    int delta;          // Scroll wheel delta (XPLM200+)
} XPMouseState_t;
```

Describes mouse state for click, drag, and wheel events.

#### `XPKeyState_t`

```c
typedef struct {
    char key;           // ASCII character (may be 0 for non-ASCII)
    XPLMKeyFlags flags; // Key state flags (up/down, shift, etc.)
    char vkey;          // Virtual key code
} XPKeyState_t;
```

Describes keyboard input state. Check `flags` to distinguish key-down vs key-up events.

#### `XPWidgetGeometryChange_t`

```c
typedef struct {
    int dx;             // X position delta
    int dy;             // Y position delta (+Y = up)
    int dwidth;         // Width change delta
    int dheight;        // Height change delta
} XPWidgetGeometryChange_t;
```

Describes changes to widget geometry during reshape operations.

## Message System

### Message Types

#### `XPWidgetMessage`

```c
typedef int XPWidgetMessage;
```

32-bit message identifiers for widget communication.

**Core Widget Messages:**

| Message | ID | Purpose | Dispatching |
|---------|----|---------| ------------|
| `xpMsg_Create` | 1 | Widget created/callback added | Direct |
| `xpMsg_Destroy` | 2 | Widget being destroyed | Direct |
| `xpMsg_Paint` | 3 | Low-level paint (handle everything) | Direct |
| `xpMsg_Draw` | 4 | High-level draw (OpenGL setup done) | Direct |
| `xpMsg_KeyPress` | 5 | Key pressed | Up Chain |
| `xpMsg_KeyTakeFocus` | 6 | Gaining keyboard focus | Direct |
| `xpMsg_KeyLoseFocus` | 7 | Losing keyboard focus | Direct |
| `xpMsg_MouseDown` | 8 | Mouse button pressed | Up Chain* |
| `xpMsg_MouseDrag` | 9 | Mouse dragged after mousedown | Direct |
| `xpMsg_MouseUp` | 10 | Mouse button released | Direct |
| `xpMsg_Reshape` | 11 | Geometry changed | Up Chain |
| `xpMsg_ExposedChanged` | 12 | Visible area changed | Direct |
| `xpMsg_AcceptChild` | 13 | Child widget added | Direct |
| `xpMsg_LoseChild` | 14 | Child widget removed | Direct |
| `xpMsg_AcceptParent` | 15 | Parent changed | Direct |
| `xpMsg_Shown` | 16 | Widget shown | Up Chain |
| `xpMsg_Hidden` | 17 | Widget hidden | Up Chain |
| `xpMsg_DescriptorChanged` | 18 | Descriptor text changed | Direct |
| `xpMsg_PropertyChanged` | 19 | Property value changed | Direct |
| `xpMsg_MouseWheel` | 20 | Mouse wheel moved (XPLM200+) | Up Chain |
| `xpMsg_CursorAdjust` | 21 | Cursor over widget (XPLM200+) | Up Chain |

*MouseDown is technically direct but library forwards until consumed

**Message ID Ranges:**
- 1000-9999: Standard widget classes
- 10000+: Plugin use (`xpMsg_UserStart`)

### Message Dispatching

#### `XPDispatchMode`

```c
typedef int XPDispatchMode;
```

Controls how messages are sent through the widget hierarchy:

| Mode | Value | Behavior |
|------|-------|----------|
| `xpMode_Direct` | 0 | Send only to target widget |
| `xpMode_UpChain` | 1 | Send to target, then up parent chain until handled |
| `xpMode_Recursive` | 2 | Send to target and all children depth-first |
| `xpMode_DirectAllCallbacks` | 3 | Send to target but all callbacks, even if handled |
| `xpMode_Once` | 4 | Send only to first handler even if not handled |

## Widget Callback Function

### `XPWidgetFunc_t`

```c
typedef int (* XPWidgetFunc_t)(
    XPWidgetMessage inMessage,    // Message type
    XPWidgetID      inWidget,     // Target widget ID
    intptr_t        inParam1,     // Message-specific parameter 1
    intptr_t        inParam2      // Message-specific parameter 2
);
```

The widget callback function signature. This function implements widget behavior by responding to messages.

**Return Values:**
- `1` - Message handled/consumed
- `0` - Message not handled (pass to next handler/parent)

**Implementation Guidelines:**
- Return `1` for messages you handle to prevent further processing
- Return `0` for unrecognized messages
- Parameters vary by message type - see individual message documentation
- Multiple callbacks can be attached to one widget (most recent first)

## Implementation Examples

### Basic Widget Callback

```c
int MyWidgetCallback(XPWidgetMessage inMessage, XPWidgetID inWidget, 
                     intptr_t inParam1, intptr_t inParam2) {
    switch (inMessage) {
        case xpMsg_Create:
            // Initialize widget-specific data
            XPSetWidgetProperty(inWidget, xpProperty_Refcon, (intptr_t)my_data);
            return 1;
            
        case xpMsg_Draw:
            // Custom drawing code here
            return 1;
            
        case xpMsg_MouseDown: {
            XPMouseState_t* mouse = (XPMouseState_t*)inParam1;
            if (mouse->button == 0) { // Left button
                // Handle click at mouse->x, mouse->y
                return 1; // Consume the click
            }
            break;
        }
        
        case xpMsg_KeyPress: {
            XPKeyState_t* key = (XPKeyState_t*)inParam1;
            if (key->flags & xplm_DownFlag) {
                // Handle key down: key->key or key->vkey
                return 1; // Consume the keystroke
            }
            break;
        }
    }
    return 0; // Not handled
}
```

### Property Usage

```c
// Set custom data
typedef struct {
    int my_value;
    float my_float;
} MyWidgetData;

MyWidgetData* data = malloc(sizeof(MyWidgetData));
XPSetWidgetProperty(widget, xpProperty_Refcon, (intptr_t)data);

// Retrieve custom data
MyWidgetData* data = (MyWidgetData*)XPGetWidgetProperty(widget, xpProperty_Refcon, NULL);

// Enable clipping
XPSetWidgetProperty(widget, xpProperty_Clip, 1);

// Disable widget
XPSetWidgetProperty(widget, xpProperty_Enabled, 0);
```

### Message Parameter Access

```c
// Use utility macros from XPWidgetUtils.h for cleaner code
int MyCallback(XPWidgetMessage inMessage, XPWidgetID inWidget, 
               intptr_t inParam1, intptr_t inParam2) {
    switch (inMessage) {
        case xpMsg_MouseDown:
            int mouse_x = MOUSE_X(inParam1);
            int mouse_y = MOUSE_Y(inParam1);
            break;
            
        case xpMsg_KeyPress:
            char key_char = KEY_CHAR(inParam1);
            XPLMKeyFlags flags = KEY_FLAGS(inParam1);
            break;
            
        case xpMsg_Reshape:
            int dx = DELTA_X(inParam2);
            int dy = DELTA_Y(inParam2);
            break;
    }
    return 0;
}
```

## Best Practices

### Memory Management
- Clean up custom data in `xpMsg_Destroy` message
- Use `xpProperty_Refcon` to store pointers to allocated data
- Don't store widget IDs beyond widget lifetime

### Message Handling
- Always return appropriate values (0 or 1)
- Handle `xpMsg_Create` and `xpMsg_Destroy` for initialization/cleanup
- Use `xpMsg_Draw` for custom rendering, `xpMsg_Paint` for complete control

### Coordinate System
- All coordinates are in global screen space
- Origin (0,0) is at bottom-left of screen
- +X = right, +Y = up
- Widget bounds define hit-testing and drawing areas

### Property Usage
- Use predefined properties appropriately
- Define custom properties starting from `xpProperty_UserStart`
- Properties persist for widget lifetime
- Property values are pointer-width integers

## Common Pitfalls

1. **Coordinate Confusion**: Remember global coordinates, not relative
2. **Message Return Values**: Always return 0 or 1 appropriately  
3. **Memory Leaks**: Clean up in `xpMsg_Destroy`
4. **Focus Handling**: Properly implement focus accept/reject in callbacks
5. **Property Types**: Properties are integers, not arbitrary pointers without casting

## Related Headers

- `XPWidgets.h` - Main widget creation and management API
- `XPWidgetUtils.h` - Utility functions and convenience macros
- `XPStandardWidgets.h` - Standard widget classes and their messages
- `XPUIGraphics.h` - Low-level drawing functions for custom widgets
