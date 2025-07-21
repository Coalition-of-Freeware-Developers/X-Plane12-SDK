# XPWidgets.h - X-Plane 12 Widget Framework Main API

## Overview

The `XPWidgets.h` header provides the primary API for creating, managing, and interacting with widgets in the X-Plane 12 Widget Framework. This is the main interface for plugin developers who want to create user interfaces.

## Theory of Operation

### Widget Architecture

Widgets are persistent view objects in X-Plane with the following characteristics:

- **Global Coordinates**: Bounding boxes defined in global screen coordinates (0,0 at bottom-left)
- **Hierarchical Structure**: Parent-child relationships form widget trees
- **Root Widgets**: Top-level widgets that create X-Plane windows
- **Message-Based Communication**: Widgets respond to messages for events and behaviors
- **Property System**: State stored via key-value properties
- **Callback-Driven**: Behavior implemented through callback functions

### Coordinate System

- Origin (0,0) is at bottom-left of screen
- +X = right, +Y = up
- All coordinates are absolute screen positions
- Widget bounds define both hit-testing and visible areas
- Visible area is intersection of widget bounds with parent bounds

### Widget Lifecycle

1. **Creation**: `XPCreateWidget()` or `XPCreateCustomWidget()`
2. **Configuration**: Set properties, attach callbacks
3. **Activation**: Place in widget hierarchy
4. **Operation**: Receive and handle messages
5. **Destruction**: `XPDestroyWidget()`

## Widget Creation and Management

### XPCreateWidget

```c
WIDGET_API XPWidgetID XPCreateWidget(
    int                  inLeft,        // Left edge (global coords)
    int                  inTop,         // Top edge (global coords)  
    int                  inRight,       // Right edge (global coords)
    int                  inBottom,      // Bottom edge (global coords)
    int                  inVisible,     // 1 = visible, 0 = hidden
    const char *         inDescriptor,  // Text string (null-terminated)
    int                  inIsRoot,      // 1 = root widget, 0 = child
    XPWidgetID           inContainer,   // Parent widget (0 for root)
    XPWidgetClass        inClass        // Widget class ID
);
```

Creates a new widget using a predefined widget class.

**Parameters:**

- **Geometry**: `inLeft`, `inTop`, `inRight`, `inBottom` define the widget's bounding rectangle
- **Visibility**: `inVisible` controls initial visibility state  
- **Descriptor**: Text string whose meaning depends on widget class (button label, field text, etc.)
- **Root Status**: `inIsRoot = 1` creates a top-level window widget
- **Container**: Parent widget ID, must be 0 for root widgets
- **Class**: Predefined widget type from `XPStandardWidgets.h`

**Returns:** Widget ID on success, `NULL` on failure

**Usage Example:**

```c
// Create a main window
XPWidgetID mainWindow = XPCreateWidget(
    100, 400, 500, 200,              // 400x200 window at (100,200)
    1,                               // Visible
    "My Plugin Window",              // Title
    1,                               // Root widget
    0,                               // No parent
    xpWidgetClass_MainWindow         // Main window class
);

// Create a button inside the window
XPWidgetID button = XPCreateWidget(
    110, 380, 200, 360,              // Button bounds
    1,                               // Visible
    "Click Me",                      // Button text
    0,                               // Not a root
    mainWindow,                      // Parent window
    xpWidgetClass_Button             // Button class
);
```

### XPCreateCustomWidget

```c
WIDGET_API XPWidgetID XPCreateCustomWidget(
    int                  inLeft,
    int                  inTop,
    int                  inRight,
    int                  inBottom,
    int                  inVisible,
    const char *         inDescriptor,
    int                  inIsRoot,
    XPWidgetID           inContainer,
    XPWidgetFunc_t       inCallback     // Custom callback function
);
```

Creates a custom widget with your own callback function instead of a predefined class.

**Usage Example:**

```c
int MyCustomWidgetCallback(XPWidgetMessage inMessage, XPWidgetID inWidget,
                           intptr_t inParam1, intptr_t inParam2) {
    switch (inMessage) {
        case xpMsg_Draw:
            // Custom drawing code
            return 1;
        case xpMsg_MouseDown:
            // Custom mouse handling
            return 1;
    }
    return 0;
}

XPWidgetID customWidget = XPCreateCustomWidget(
    10, 100, 200, 50,
    1, "Custom Widget", 0, parentWidget,
    MyCustomWidgetCallback
);
```

### XPDestroyWidget

```c
WIDGET_API void XPDestroyWidget(
    XPWidgetID           inWidget,
    int                  inDestroyChildren
);
```

Destroys a widget and optionally its children.

**Parameters:**

- **inWidget**: Widget to destroy
- **inDestroyChildren**: 
  - `1` = Recursively destroy all child widgets
  - `0` = Only destroy this widget, children become parentless

**Important Notes:**

- Widget receives `xpMsg_Destroy` message before destruction
- Children destroyed with `inDestroyChildren = 1` 
- All associated resources are cleaned up
- Widget ID becomes invalid after destruction

### XPSendMessageToWidget

```c
WIDGET_API int XPSendMessageToWidget(
    XPWidgetID           inWidget,
    XPWidgetMessage      inMessage,
    XPDispatchMode       inMode,
    intptr_t             inParam1,
    intptr_t             inParam2
);
```

Sends a message to a widget with specified dispatching behavior.

**Parameters:**

- **inWidget**: Target widget
- **inMessage**: Message type
- **inMode**: Dispatching mode (see `XPDispatchMode` in `XPWidgetDefs.h`)
- **inParam1**, **inParam2**: Message-specific parameters

**Returns:** 1 if message was handled, 0 otherwise

**Usage Example:**

```c
// Send custom message to widget
XPSendMessageToWidget(myWidget, xpMsg_UserStart + 100, 
                      xpMode_Direct, (intptr_t)data, 0);
```

## Widget Positioning and Visibility

### Hierarchy Management

#### XPPlaceWidgetWithin

```c
WIDGET_API void XPPlaceWidgetWithin(
    XPWidgetID           inSubWidget,
    XPWidgetID           inContainer
);
```

Changes a widget's parent container.

**Important:**

- Cannot be used on root widgets
- Pass `0` for `inContainer` to remove from hierarchy
- Does not change widget's global coordinates
- Widget becomes last/topmost child in new container

#### XPCountChildWidgets

```c
WIDGET_API int XPCountChildWidgets(XPWidgetID inWidget);
```

Returns the number of direct child widgets.

#### XPGetNthChildWidget

```c
WIDGET_API XPWidgetID XPGetNthChildWidget(
    XPWidgetID           inWidget,
    int                  inIndex
);
```

Returns the widget ID of the nth child (0-based indexing).

#### XPGetParentWidget

```c
WIDGET_API XPWidgetID XPGetParentWidget(XPWidgetID inWidget);
```

Returns the parent widget ID, or `0` if no parent (root widgets).

### Visibility Control

#### XPShowWidget / XPHideWidget

```c
WIDGET_API void XPShowWidget(XPWidgetID inWidget);
WIDGET_API void XPHideWidget(XPWidgetID inWidget);
```

Controls widget visibility. Note that a widget is only actually visible if:

1. The widget's visibility flag is true
2. All parent widgets are visible  
3. The widget is in a rooted hierarchy

#### XPIsWidgetVisible

```c
WIDGET_API int XPIsWidgetVisible(XPWidgetID inWidget);
```

Returns `1` if widget is actually visible to user, considering parent visibility.

### Root Widget Management

#### XPFindRootWidget

```c
WIDGET_API XPWidgetID XPFindRootWidget(XPWidgetID inWidget);
```

Returns the root widget containing this widget, or `NULL` if not in a rooted hierarchy.

#### XPBringRootWidgetToFront

```c
WIDGET_API void XPBringRootWidgetToFront(XPWidgetID inWidget);
```

Brings the widget's root hierarchy to the front.

#### XPIsWidgetInFront

```c
WIDGET_API int XPIsWidgetInFront(XPWidgetID inWidget);
```

Returns `1` if widget's hierarchy is frontmost.

### Geometry Management

#### XPGetWidgetGeometry

```c
WIDGET_API void XPGetWidgetGeometry(
    XPWidgetID           inWidget,
    int *                outLeft,      // Can be NULL
    int *                outTop,       // Can be NULL
    int *                outRight,     // Can be NULL
    int *                outBottom     // Can be NULL
);
```

Retrieves widget's bounding box in global coordinates.

#### XPSetWidgetGeometry

```c
WIDGET_API void XPSetWidgetGeometry(
    XPWidgetID           inWidget,
    int                  inLeft,
    int                  inTop,
    int                  inRight,
    int                  inBottom
);
```

Changes widget's bounding box.

#### XPGetWidgetForLocation

```c
WIDGET_API XPWidgetID XPGetWidgetForLocation(
    XPWidgetID           inContainer,
    int                  inXOffset,
    int                  inYOffset,
    int                  inRecursive,
    int                  inVisibleOnly
);
```

Finds the child widget at a specific location.

**Parameters:**

- **inRecursive**: `1` = search children of children
- **inVisibleOnly**: `1` = only consider visible widgets

**Returns:** Widget at location, `inContainer` if location is in container but not in child, or `0` if not in container

#### XPGetWidgetExposedGeometry

```c
WIDGET_API void XPGetWidgetExposedGeometry(
    XPWidgetID           inWidgetID,
    int *                outLeft,
    int *                outTop,  
    int *                outRight,
    int *                outBottom
);
```

Returns the portion of widget that's actually visible (clipped by parents).

## Widget Data Access

### Descriptor Management

#### XPSetWidgetDescriptor

```c
WIDGET_API void XPSetWidgetDescriptor(
    XPWidgetID           inWidget,
    const char *         inDescriptor
);
```

Sets the widget's descriptor text string.

#### XPGetWidgetDescriptor

```c
WIDGET_API int XPGetWidgetDescriptor(
    XPWidgetID           inWidget,
    char *               outDescriptor,
    int                  inMaxDescLength
);
```

Retrieves the widget's descriptor text.

**Returns:** Actual descriptor length. Pass `NULL` for `outDescriptor` to get length only.

### Window Integration

#### XPGetWidgetUnderlyingWindow

```c
WIDGET_API XPLMWindowID XPGetWidgetUnderlyingWindow(XPWidgetID inWidget);
```

Returns the XPLM window backing the widget window. Requires modern windows feature enabled:

```c
XPLMEnableFeature("XPLM_USE_NATIVE_WIDGET_WINDOWS", 1);
```

### Property Management

#### XPSetWidgetProperty

```c
WIDGET_API void XPSetWidgetProperty(
    XPWidgetID           inWidget,
    XPWidgetPropertyID   inProperty,
    intptr_t             inValue
);
```

Sets a widget property value.

#### XPGetWidgetProperty

```c
WIDGET_API intptr_t XPGetWidgetProperty(
    XPWidgetID           inWidget,
    XPWidgetPropertyID   inProperty,
    int *                inExists      // Can be NULL
);
```

Gets a widget property value. Set `inExists` to check if property is defined.

## Keyboard Management

### XPSetKeyboardFocus

```c
WIDGET_API XPWidgetID XPSetKeyboardFocus(XPWidgetID inWidget);
```

Sets keyboard focus to a widget. Pass `0` to give focus to X-Plane.

**Returns:** Widget that actually received focus, or `0` for X-Plane.

### XPLoseKeyboardFocus

```c
WIDGET_API void XPLoseKeyboardFocus(XPWidgetID inWidget);
```

Causes widget to lose keyboard focus (passed to parent).

### XPGetWidgetWithFocus

```c
WIDGET_API XPWidgetID XPGetWidgetWithFocus(void);
```

Returns the widget with keyboard focus, or `0` if X-Plane has focus.

## Custom Widget Development

### XPAddWidgetCallback

```c
WIDGET_API void XPAddWidgetCallback(
    XPWidgetID           inWidget,
    XPWidgetFunc_t       inNewCallback
);
```

Adds a callback function to an existing widget for customization/subclassing.

**Key Points:**

- New callback receives messages first
- Multiple callbacks can be attached
- `xpMsg_Create` sent immediately to new callback
- `xpMsg_Destroy` sent before widget destruction
- Enables extending/customizing standard widget behavior

### XPGetWidgetClassFunc

```c
WIDGET_API XPWidgetFunc_t XPGetWidgetClassFunc(XPWidgetClass inWidgetClass);
```

Returns the callback function for a standard widget class.

## Implementation Examples

### Complete Widget Window Example

```c
static XPWidgetID gMainWindow = NULL;
static XPWidgetID gOKButton = NULL;
static XPWidgetID gTextField = NULL;

int ButtonHandler(XPWidgetMessage inMessage, XPWidgetID inWidget,
                  intptr_t inParam1, intptr_t inParam2) {
    if (inMessage == xpMsg_PushButtonPressed) {
        if (inWidget == gOKButton) {
            // Get text field contents
            char text[256];
            XPGetWidgetDescriptor(gTextField, text, sizeof(text));
            XPLMDebugString("Text field contains: ");
            XPLMDebugString(text);
            XPLMDebugString("\n");
        }
        return 1;
    }
    return 0;
}

void CreateWindow() {
    // Create main window
    gMainWindow = XPCreateWidget(
        100, 400, 400, 200,
        1, "Example Dialog", 1, 0,
        xpWidgetClass_MainWindow);
    
    // Create text field
    gTextField = XPCreateWidget(
        110, 380, 380, 360,
        1, "Enter text here", 0, gMainWindow,
        xpWidgetClass_TextField);
    
    // Create OK button
    gOKButton = XPCreateWidget(
        310, 230, 380, 210,
        1, "OK", 0, gMainWindow,
        xpWidgetClass_Button);
    
    // Set button type and behavior
    XPSetWidgetProperty(gOKButton, xpProperty_ButtonType, xpPushButton);
    XPSetWidgetProperty(gOKButton, xpProperty_ButtonBehavior, xpButtonBehaviorPushButton);
    
    // Add button handler
    XPAddWidgetCallback(gOKButton, ButtonHandler);
}

void DestroyWindow() {
    if (gMainWindow) {
        XPDestroyWidget(gMainWindow, 1); // Destroy children too
        gMainWindow = NULL;
        gOKButton = NULL;
        gTextField = NULL;
    }
}
```

### Custom Widget with Drawing

```c
typedef struct {
    float red, green, blue;
    int highlighted;
} ColorBoxData;

int ColorBoxCallback(XPWidgetMessage inMessage, XPWidgetID inWidget,
                     intptr_t inParam1, intptr_t inParam2) {
    switch (inMessage) {
        case xpMsg_Create: {
            // Initialize custom data
            ColorBoxData* data = malloc(sizeof(ColorBoxData));
            data->red = 0.5f;
            data->green = 0.7f; 
            data->blue = 0.9f;
            data->highlighted = 0;
            XPSetWidgetProperty(inWidget, xpProperty_Refcon, (intptr_t)data);
            return 1;
        }
        
        case xpMsg_Destroy: {
            // Clean up custom data
            ColorBoxData* data = (ColorBoxData*)XPGetWidgetProperty(inWidget, xpProperty_Refcon, NULL);
            if (data) free(data);
            return 1;
        }
        
        case xpMsg_Draw: {
            // Custom drawing
            ColorBoxData* data = (ColorBoxData*)XPGetWidgetProperty(inWidget, xpProperty_Refcon, NULL);
            if (!data) return 1;
            
            int left, top, right, bottom;
            XPGetWidgetGeometry(inWidget, &left, &top, &right, &bottom);
            
            XPLMSetGraphicsState(0, 0, 0, 1, 1, 0, 0);
            if (data->highlighted) {
                glColor3f(1.0f, 1.0f, 1.0f); // White when highlighted
            } else {
                glColor3f(data->red, data->green, data->blue);
            }
            
            glBegin(GL_QUADS);
            glVertex2i(left, bottom);
            glVertex2i(right, bottom);
            glVertex2i(right, top);
            glVertex2i(left, top);
            glEnd();
            
            return 1;
        }
        
        case xpMsg_MouseDown: {
            // Toggle highlight on click
            ColorBoxData* data = (ColorBoxData*)XPGetWidgetProperty(inWidget, xpProperty_Refcon, NULL);
            if (data) {
                data->highlighted = !data->highlighted;
            }
            return 1;
        }
    }
    return 0;
}
```

### Layout Management Example

```c
void ResizeChildWidgets(XPWidgetID parent) {
    int left, top, right, bottom;
    XPGetWidgetGeometry(parent, &left, &top, &right, &bottom);
    
    int width = right - left;
    int height = top - bottom;
    
    int childCount = XPCountChildWidgets(parent);
    int childHeight = (height - 20) / childCount; // 10px margins
    
    for (int i = 0; i < childCount; i++) {
        XPWidgetID child = XPGetNthChildWidget(parent, i);
        if (child) {
            int childTop = top - 10 - (i * childHeight);
            int childBottom = childTop - childHeight + 10;
            XPSetWidgetGeometry(child, left + 10, childTop, 
                               right - 10, childBottom);
        }
    }
}
```

## Best Practices

### Widget Creation

1. **Create hierarchies top-down**: Create parent widgets before children
2. **Use appropriate classes**: Choose the right standard widget class for your needs
3. **Set properties immediately**: Configure widget properties right after creation
4. **Handle resource cleanup**: Always clean up in `xpMsg_Destroy`

### Coordinate Management

1. **Think global**: All coordinates are global screen coordinates
2. **Consider parent bounds**: Children outside parent bounds won't receive input
3. **Use exposed geometry**: For drawing, use exposed area not full bounds
4. **Handle resizing**: Implement layout management for resizable windows

### Message Handling

1. **Return appropriate values**: 1 for handled, 0 for not handled
2. **Chain calls appropriately**: Let standard behavior run first when subclassing
3. **Validate parameters**: Check message parameters for validity
4. **Use proper dispatching**: Choose correct dispatch mode for custom messages

### Memory Management

1. **Use refcon property**: Store custom data using `xpProperty_Refcon`
2. **Clean up in destroy**: Free allocated memory in `xpMsg_Destroy`
3. **Null check properties**: Always check if properties exist before using
4. **Avoid dangling references**: Don't store widget IDs beyond widget lifetime

## Related Headers

- `XPWidgetDefs.h` - Core definitions and types
- `XPWidgetUtils.h` - Utility functions and convenience macros
- `XPStandardWidgets.h` - Standard widget classes and their specific APIs
- `XPUIGraphics.h` - Low-level drawing functions for custom widgets
- `XPLMDisplay.h` - Underlying window system integration
