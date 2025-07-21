# XPWidgetUtils.h - X-Plane 12 Widget Framework Utility Functions

## Overview

The `XPWidgetUtils.h` header provides utility functions and convenience macros that simplify widget development in X-Plane 12. This library contains pre-built behaviors, layout managers, and helper functions that reduce boilerplate code and add common functionality to widgets.

## Theory of Operation

The utility library provides two main categories of functionality:

1. **Widget Behavior Functions**: Pre-built behaviors that can be attached to widgets or called from custom widget functions
2. **Convenience Utilities**: Helper macros and functions for common widget operations

### Usage Patterns

Widget behavior functions can be used in two ways:

1. **As Callbacks**: Attach behavior functions directly to widgets using `XPAddWidgetCallback()`
2. **From Custom Functions**: Call behavior functions from your own widget callback to include their functionality

## Convenience Utilities

### Parameter Access Macros

These macros simplify extracting values from message parameters:

#### Mouse Event Macros

```c
#define MOUSE_X(param) (((XPMouseState_t *) (param))->x)
#define MOUSE_Y(param) (((XPMouseState_t *) (param))->y)
```

Extract mouse coordinates from mouse event messages.

**Usage Example:**

```c
int MyWidgetCallback(XPWidgetMessage inMessage, XPWidgetID inWidget,
                     intptr_t inParam1, intptr_t inParam2) {
    switch (inMessage) {
        case xpMsg_MouseDown: {
            int x = MOUSE_X(inParam1);  // Mouse X coordinate
            int y = MOUSE_Y(inParam1);  // Mouse Y coordinate
            // Handle click at (x, y)
            return 1;
        }
    }
    return 0;
}
```

#### Geometry Change Macros

```c
#define DELTA_X(param) (((XPWidgetGeometryChange_t *) (param))->dx)
#define DELTA_Y(param) (((XPWidgetGeometryChange_t *) (param))->dy)
#define DELTA_W(param) (((XPWidgetGeometryChange_t *) (param))->dwidth)
#define DELTA_H(param) (((XPWidgetGeometryChange_t *) (param))->dheight)
```

Extract geometry change deltas from reshape messages.

**Usage Example:**

```c
case xpMsg_Reshape: {
    int dx = DELTA_X(inParam2);        // X position change
    int dy = DELTA_Y(inParam2);        // Y position change  
    int dwidth = DELTA_W(inParam2);    // Width change
    int dheight = DELTA_H(inParam2);   // Height change
    // Handle geometry change
    return 1;
}
```

#### Keyboard Event Macros

```c
#define KEY_CHAR(param) (((XPKeyState_t *) (param))->key)
#define KEY_FLAGS(param) (((XPKeyState_t *) (param))->flags)
#define KEY_VKEY(param) (((XPKeyState_t *) (param))->vkey)
```

Extract keyboard information from key press messages.

**Usage Example:**

```c
case xpMsg_KeyPress: {
    char key = KEY_CHAR(inParam1);           // ASCII character
    XPLMKeyFlags flags = KEY_FLAGS(inParam1); // Key state flags
    char vkey = KEY_VKEY(inParam1);          // Virtual key code
    
    if (flags & xplm_DownFlag) {
        // Handle key down
        if (key == '\r' || key == '\n') {
            // Enter key pressed
        }
    }
    return 1;
}
```

### Rectangle Testing Macro

```c
#define IN_RECT(x, y, l, t, r, b) \
    (((x) >= (l)) && ((x) <= (r)) && ((y) >= (b)) && ((y) <= (t)))
```

Tests if a point is within a rectangle.

**Parameters:**
- `x`, `y`: Point coordinates
- `l`, `t`, `r`, `b`: Left, top, right, bottom of rectangle

**Usage Example:**

```c
int left, top, right, bottom;
XPGetWidgetGeometry(widget, &left, &top, &right, &bottom);

if (IN_RECT(mouseX, mouseY, left, top, right, bottom)) {
    // Mouse is inside widget bounds
}
```

## Bulk Widget Creation

### XPWidgetCreate_t Structure

```c
typedef struct {
    int                left;
    int                top;
    int                right;
    int                bottom;
    int                visible;
    const char *       descriptor;
    int                isRoot;
    int                containerIndex;   // Index into widget array or special value
    XPWidgetClass      widgetClass;
} XPWidgetCreate_t;
```

Describes a widget to be created in bulk operations.

**Special Container Index Values:**

```c
#define NO_PARENT            -1    // Widget has no parent
#define PARAM_PARENT         -2    // Use parameter parent passed to XPUCreateWidgets
```

### XPUCreateWidgets

```c
WIDGET_API void XPUCreateWidgets(
    const XPWidgetCreate_t * inWidgetDefs,   // Array of widget definitions
    int                      inCount,        // Number of widgets to create
    XPWidgetID               inParamParent,  // Parent for PARAM_PARENT widgets
    XPWidgetID *             ioWidgets       // Output array for widget IDs
);
```

Creates multiple widgets from a table definition.

**Usage Example:**

```c
static XPWidgetCreate_t widgetDefs[] = {
    // Main window (index 0)
    { 100, 400, 500, 200, 1, "Dialog", 1, NO_PARENT, xpWidgetClass_MainWindow },
    
    // Sub window (index 1, parent is widget 0)
    { 110, 390, 490, 210, 1, "", 0, 0, xpWidgetClass_SubWindow },
    
    // Button (index 2, parent is widget 1)
    { 120, 380, 200, 360, 1, "OK", 0, 1, xpWidgetClass_Button },
    
    // Text field (index 3, parent is widget 1)  
    { 220, 380, 480, 360, 1, "Enter text", 0, 1, xpWidgetClass_TextField }
};

XPWidgetID widgets[4];
XPUCreateWidgets(widgetDefs, 4, NULL, widgets);

// Access created widgets
XPWidgetID mainWindow = widgets[0];
XPWidgetID subWindow = widgets[1];
XPWidgetID button = widgets[2];
XPWidgetID textField = widgets[3];
```

### Helper Macros

```c
#define WIDGET_COUNT(x) ((sizeof(x) / sizeof(XPWidgetCreate_t)))
```

Calculates the number of elements in a widget definition array.

**Usage Example:**

```c
static XPWidgetCreate_t myWidgets[] = { /* ... */ };
XPWidgetID widgets[WIDGET_COUNT(myWidgets)];
XPUCreateWidgets(myWidgets, WIDGET_COUNT(myWidgets), NULL, widgets);
```

### XPUMoveWidgetBy

```c
WIDGET_API void XPUMoveWidgetBy(
    XPWidgetID           inWidget,
    int                  inDeltaX,    // +X = right
    int                  inDeltaY     // +Y = up
);
```

Moves a widget by the specified offset without resizing.

**Usage Example:**

```c
// Move widget 50 pixels right and 30 pixels up
XPUMoveWidgetBy(myWidget, 50, 30);
```

## Layout Managers

Layout managers are behavior functions that automatically manage widget positioning.

### XPUFixedLayout

```c
WIDGET_API int XPUFixedLayout(
    XPWidgetMessage      inMessage,
    XPWidgetID           inWidget,
    intptr_t             inParam1,
    intptr_t             inParam2
);
```

Maintains children in fixed positions relative to the parent widget as it is resized.

**Usage Patterns:**

**As Attached Callback:**

```c
// Attach to a window widget
XPAddWidgetCallback(mainWindow, XPUFixedLayout);
```

**From Custom Callback:**

```c
int MyWindowCallback(XPWidgetMessage inMessage, XPWidgetID inWidget,
                     intptr_t inParam1, intptr_t inParam2) {
    // Let fixed layout handle geometry first
    if (XPUFixedLayout(inMessage, inWidget, inParam1, inParam2)) {
        return 1;
    }
    
    // Handle other messages
    switch (inMessage) {
        case xpMsg_Create:
            // Custom initialization
            return 1;
    }
    return 0;
}
```

**Implementation Details:**
- Children maintain relative positions when parent is resized
- Use on top-level window widgets
- Preserves aspect ratios and relative spacing
- Essential for resizable dialog boxes

## Widget Behavior Functions

These functions add specific behaviors to widgets and must be called from widget callbacks.

### XPUSelectIfNeeded

```c
WIDGET_API int XPUSelectIfNeeded(
    XPWidgetMessage      inMessage,
    XPWidgetID           inWidget,
    intptr_t             inParam1,
    intptr_t             inParam2,
    int                  inEatClick     // 1 = consume background clicks
);
```

Brings widget window to foreground if it's not already active.

**Parameters:**
- **inEatClick**: Whether background clicks are consumed by the bring-to-front action

**Usage Example:**

```c
int MyWindowCallback(XPWidgetMessage inMessage, XPWidgetID inWidget,
                     intptr_t inParam1, intptr_t inParam2) {
    // Handle window activation first
    if (XPUSelectIfNeeded(inMessage, inWidget, inParam1, inParam2, 1)) {
        return 1; // Click was consumed by bringing window to front
    }
    
    // Handle other messages normally
    switch (inMessage) {
        case xpMsg_MouseDown:
            // This only executes if window was already in front
            return 1;
    }
    return 0;
}
```

### XPUDefocusKeyboard

```c
WIDGET_API int XPUDefocusKeyboard(
    XPWidgetMessage      inMessage,
    XPWidgetID           inWidget,
    intptr_t             inParam1,
    intptr_t             inParam2,
    int                  inEatClick
);
```

Returns keyboard focus to X-Plane, stopping text editing in fields.

**Usage Example:**

```c
int MyCloseButtonCallback(XPWidgetMessage inMessage, XPWidgetID inWidget,
                          intptr_t inParam1, intptr_t inParam2) {
    if (inMessage == xpMsg_PushButtonPressed) {
        // Return focus to X-Plane before closing dialog
        XPUDefocusKeyboard(inMessage, inWidget, inParam1, inParam2, 1);
        // Close dialog...
        return 1;
    }
    return 0;
}
```

### XPUDragWidget

```c
WIDGET_API int XPUDragWidget(
    XPWidgetMessage      inMessage,
    XPWidgetID           inWidget,
    intptr_t             inParam1,
    intptr_t             inParam2,
    int                  inLeft,       // Drag region left
    int                  inTop,        // Drag region top
    int                  inRight,      // Drag region right
    int                  inBottom      // Drag region bottom
);
```

Implements dragging behavior for widgets. The drag region defines the area that responds to drag operations (typically a title bar).

**Usage Example:**

```c
int MyWindowCallback(XPWidgetMessage inMessage, XPWidgetID inWidget,
                     intptr_t inParam1, intptr_t inParam2) {
    // Get window bounds
    int left, top, right, bottom;
    XPGetWidgetGeometry(inWidget, &left, &top, &right, &bottom);
    
    // Define title bar as drag region (top 20 pixels)
    int titleLeft = left;
    int titleTop = top;
    int titleRight = right;
    int titleBottom = top - 20;
    
    // Handle dragging in title bar area
    if (XPUDragWidget(inMessage, inWidget, inParam1, inParam2,
                      titleLeft, titleTop, titleRight, titleBottom)) {
        return 1; // Drag was handled
    }
    
    // Handle other window messages
    return 0;
}
```

## Complete Implementation Examples

### Simple Dialog with Utilities

```c
static XPWidgetID gDialog = NULL;
static XPWidgetID gOKButton = NULL;
static XPWidgetID gCancelButton = NULL;

// Widget definitions for bulk creation
static XPWidgetCreate_t dialogWidgets[] = {
    // Main window
    { 100, 300, 400, 150, 1, "Confirmation", 1, NO_PARENT, xpWidgetClass_MainWindow },
    // Sub window  
    { 110, 290, 390, 160, 1, "", 0, 0, xpWidgetClass_SubWindow },
    // Message caption
    { 120, 280, 380, 260, 1, "Are you sure?", 0, 1, xpWidgetClass_Caption },
    // OK button
    { 220, 190, 280, 170, 1, "OK", 0, 1, xpWidgetClass_Button },
    // Cancel button  
    { 300, 190, 370, 170, 1, "Cancel", 0, 1, xpWidgetClass_Button }
};

static XPWidgetID dialogWidgetIDs[WIDGET_COUNT(dialogWidgets)];

int DialogCallback(XPWidgetMessage inMessage, XPWidgetID inWidget,
                   intptr_t inParam1, intptr_t inParam2) {
    // Handle window selection and dragging
    if (XPUSelectIfNeeded(inMessage, inWidget, inParam1, inParam2, 1)) {
        return 1;
    }
    
    // Handle dragging by title bar
    int left, top, right, bottom;
    XPGetWidgetGeometry(inWidget, &left, &top, &right, &bottom);
    if (XPUDragWidget(inMessage, inWidget, inParam1, inParam2,
                      left, top, right, top - 20)) { // Title bar area
        return 1;
    }
    
    // Handle button clicks
    if (inMessage == xpMsg_PushButtonPressed) {
        if (inWidget == gOKButton) {
            XPLMDebugString("OK button pressed\n");
            XPHideWidget(gDialog);
        } else if (inWidget == gCancelButton) {
            XPLMDebugString("Cancel button pressed\n");
            XPHideWidget(gDialog);
        }
        return 1;
    }
    
    return 0;
}

void CreateDialog() {
    // Create all widgets at once
    XPUCreateWidgets(dialogWidgets, WIDGET_COUNT(dialogWidgets), NULL, dialogWidgetIDs);
    
    // Store widget references
    gDialog = dialogWidgetIDs[0];
    gOKButton = dialogWidgetIDs[3];
    gCancelButton = dialogWidgetIDs[4];
    
    // Configure button properties
    XPSetWidgetProperty(gOKButton, xpProperty_ButtonType, xpPushButton);
    XPSetWidgetProperty(gOKButton, xpProperty_ButtonBehavior, xpButtonBehaviorPushButton);
    XPSetWidgetProperty(gCancelButton, xpProperty_ButtonType, xpPushButton);
    XPSetWidgetProperty(gCancelButton, xpProperty_ButtonBehavior, xpButtonBehaviorPushButton);
    
    // Add behavior callbacks
    XPAddWidgetCallback(gDialog, DialogCallback);
    XPAddWidgetCallback(gDialog, XPUFixedLayout); // Maintain layout on resize
}
```

### Resizable Window with Custom Layout

```c
static XPWidgetID gMainWindow = NULL;
static XPWidgetID gTextArea = NULL;
static XPWidgetID gStatusBar = NULL;

int CustomLayoutCallback(XPWidgetMessage inMessage, XPWidgetID inWidget,
                         intptr_t inParam1, intptr_t inParam2) {
    if (inMessage == xpMsg_Reshape) {
        // Custom layout management
        int left, top, right, bottom;
        XPGetWidgetGeometry(inWidget, &left, &top, &right, &bottom);
        
        // Status bar: 20 pixels high at bottom
        XPSetWidgetGeometry(gStatusBar, left + 5, bottom + 25, right - 5, bottom + 5);
        
        // Text area: fill remaining space
        XPSetWidgetGeometry(gTextArea, left + 5, top - 25, right - 5, bottom + 30);
        
        return 1;
    }
    
    return 0;
}

int WindowCallback(XPWidgetMessage inMessage, XPWidgetID inWidget,
                   intptr_t inParam1, intptr_t inParam2) {
    // Handle window behaviors first
    if (XPUSelectIfNeeded(inMessage, inWidget, inParam1, inParam2, 1)) {
        return 1;
    }
    
    // Handle dragging
    int left, top, right, bottom;
    XPGetWidgetGeometry(inWidget, &left, &top, &right, &bottom);
    if (XPUDragWidget(inMessage, inWidget, inParam1, inParam2,
                      left, top, right, top - 25)) { // Title bar
        return 1;
    }
    
    // Custom layout management
    if (CustomLayoutCallback(inMessage, inWidget, inParam1, inParam2)) {
        return 1;
    }
    
    return 0;
}

void CreateResizableWindow() {
    // Create main window
    gMainWindow = XPCreateWidget(100, 400, 500, 200, 1, "Resizable Window", 
                                 1, 0, xpWidgetClass_MainWindow);
    
    // Create text area
    gTextArea = XPCreateWidget(105, 375, 495, 225, 1, "Content area", 
                               0, gMainWindow, xpWidgetClass_TextField);
    
    // Create status bar
    gStatusBar = XPCreateWidget(105, 225, 495, 205, 1, "Status: Ready",
                                0, gMainWindow, xpWidgetClass_Caption);
    
    // Add callbacks
    XPAddWidgetCallback(gMainWindow, WindowCallback);
    
    // Initial layout
    CustomLayoutCallback(xpMsg_Reshape, gMainWindow, 0, 0);
}
```

### Multi-Step Dialog with Navigation

```c
typedef struct {
    int currentStep;
    int totalSteps;
    XPWidgetID stepWidgets[3];
} WizardData;

int WizardCallback(XPWidgetMessage inMessage, XPWidgetID inWidget,
                   intptr_t inParam1, intptr_t inParam2) {
    WizardData* data = (WizardData*)XPGetWidgetProperty(inWidget, xpProperty_Refcon, NULL);
    
    switch (inMessage) {
        case xpMsg_Create: {
            // Initialize wizard data
            data = malloc(sizeof(WizardData));
            data->currentStep = 0;
            data->totalSteps = 3;
            XPSetWidgetProperty(inWidget, xpProperty_Refcon, (intptr_t)data);
            return 1;
        }
        
        case xpMsg_Destroy:
            if (data) free(data);
            return 1;
            
        case xpMsg_PushButtonPressed: {
            XPWidgetID button = (XPWidgetID)inParam1;
            char desc[256];
            XPGetWidgetDescriptor(button, desc, sizeof(desc));
            
            if (strcmp(desc, "Next") == 0 && data->currentStep < data->totalSteps - 1) {
                // Hide current step
                XPHideWidget(data->stepWidgets[data->currentStep]);
                // Show next step
                data->currentStep++;
                XPShowWidget(data->stepWidgets[data->currentStep]);
            } else if (strcmp(desc, "Previous") == 0 && data->currentStep > 0) {
                // Hide current step
                XPHideWidget(data->stepWidgets[data->currentStep]);
                // Show previous step  
                data->currentStep--;
                XPShowWidget(data->stepWidgets[data->currentStep]);
            }
            return 1;
        }
    }
    
    // Handle standard window behaviors
    if (XPUSelectIfNeeded(inMessage, inWidget, inParam1, inParam2, 1)) {
        return 1;
    }
    
    return 0;
}
```

## Best Practices

### Using Behavior Functions

1. **Call behaviors first**: Let behavior functions handle events before custom processing
2. **Check return values**: Behavior functions return 1 if they handle the message
3. **Combine behaviors**: Multiple behavior functions can work together
4. **Order matters**: The order of callback additions affects message processing

### Bulk Widget Creation

1. **Plan hierarchies**: Design parent-child relationships before creating widget arrays
2. **Use indices correctly**: Container indices must reference earlier array positions
3. **Validate definitions**: Check widget definitions for correct bounds and properties
4. **Store references**: Keep references to important widgets from the created array

### Layout Management

1. **Handle reshape messages**: Implement custom layout in response to geometry changes
2. **Maintain proportions**: Preserve visual relationships during resize operations
3. **Set minimum sizes**: Prevent widgets from becoming unusably small
4. **Test edge cases**: Verify layout with extreme window sizes

### Coordinate Calculations

1. **Use helper macros**: Leverage provided macros for cleaner code
2. **Validate bounds**: Check that calculated coordinates are reasonable
3. **Consider margins**: Leave space between adjacent widgets
4. **Account for borders**: Remember window frames and borders in calculations

## Related Headers

- `XPWidgetDefs.h` - Core widget definitions and message types
- `XPWidgets.h` - Main widget creation and management API
- `XPStandardWidgets.h` - Standard widget classes and their properties
- `XPUIGraphics.h` - Low-level drawing functions for custom rendering
