# XPUIGraphics.h - X-Plane 12 Widget Framework UI Graphics

## Overview

The `XPUIGraphics.h` header provides low-level drawing functions for creating custom widget graphics in X-Plane 12. This API allows developers to draw standard X-Plane UI elements like windows, buttons, scrollbars, and other interface components with consistent styling.

## Theory of Operation

The UI Graphics API provides three main categories of drawable elements:

1. **Windows**: Background panels and frames for organizing interface elements
2. **Elements**: Individual UI components like buttons, checkboxes, and text fields
3. **Tracks**: Value-based controls like scrollbars, sliders, and progress bars

All drawing functions use X-Plane's built-in graphics resources and automatically adapt to the current X-Plane version's visual style.

### Coordinate System

- All coordinates are in global screen space
- Origin (0,0) is at bottom-left of screen
- Parameters specify rectangle bounds: `(x1, y1)` = bottom-left, `(x2, y2)` = top-right
- Widgets automatically scale and position elements appropriately

## Window Graphics

Windows provide background styling for dialog boxes and panels.

### XPWindowStyle

```c
typedef int XPWindowStyle;
```

Defines the visual style of window backgrounds:

| Style | Value | Description | Usage |
|-------|-------|-------------|-------|
| `xpWindow_Help` | 0 | LCD screen style for help dialogs | Help windows, information displays |
| `xpWindow_MainWindow` | 1 | Standard dialog box window | Primary dialogs, floating windows |
| `xpWindow_SubWindow` | 2 | Panel within dialog box | Sections within main windows |
| `xpWindow_Screen` | 4 | LCD screen for text displays | Text areas, data displays |
| `xpWindow_ListView` | 5 | Scrolling list background | File lists, option lists |

### XPDrawWindow

```c
WIDGET_API void XPDrawWindow(
    int                  inX1,        // Left edge
    int                  inY1,        // Bottom edge
    int                  inX2,        // Right edge
    int                  inY2,        // Top edge
    XPWindowStyle        inStyle      // Window style
);
```

Draws a window background with the specified style and dimensions.

**Usage Example:**

```c
int MyWindowCallback(XPWidgetMessage inMessage, XPWidgetID inWidget,
                     intptr_t inParam1, intptr_t inParam2) {
    if (inMessage == xpMsg_Draw) {
        int left, top, right, bottom;
        XPGetWidgetGeometry(inWidget, &left, &top, &right, &bottom);
        
        // Draw main window background
        XPDrawWindow(left, bottom, right, top, xpWindow_MainWindow);
        
        // Draw sub-panel
        XPDrawWindow(left + 10, bottom + 10, right - 10, top - 30, xpWindow_SubWindow);
        
        return 1;
    }
    return 0;
}
```

### XPGetWindowDefaultDimensions

```c
WIDGET_API void XPGetWindowDefaultDimensions(
    XPWindowStyle        inStyle,
    int *                outWidth,     // Can be NULL
    int *                outHeight     // Can be NULL
);
```

Returns the default or minimum dimensions for a window style.

**Usage Example:**

```c
void CreateOptimalWindow(XPWindowStyle style) {
    int defaultWidth, defaultHeight;
    XPGetWindowDefaultDimensions(style, &defaultWidth, &defaultHeight);
    
    // Create window with at least the default size
    int width = max(defaultWidth, 400);
    int height = max(defaultHeight, 200);
    
    XPWidgetID window = XPCreateWidget(100, 100 + height, 100 + width, 100,
                                       1, "Window", 1, 0, xpWidgetClass_MainWindow);
}
```

## UI Elements

Elements are individual interface components that can be drawn independently.

### XPElementStyle

```c
typedef int XPElementStyle;
```

Defines individual UI element types:

#### Form Controls

| Element | Value | Scalable | Background | Description |
|---------|-------|----------|------------|-------------|
| `xpElement_TextField` | 6 | X-axis | Metal | Text input field |
| `xpElement_CheckBox` | 9 | No | Metal | Unchecked checkbox |
| `xpElement_CheckBoxLit` | 10 | No | Metal | Checked checkbox |
| `xpElement_PushButton` | 16 | X-axis | Metal | Standard button |
| `xpElement_PushButtonLit` | 17 | X-axis | Metal | Highlighted button |

#### Window Controls

| Element | Value | Description |
|---------|-------|-------------|
| `xpElement_WindowCloseBox` | 14 | Close button (normal) |
| `xpElement_WindowCloseBoxPressed` | 15 | Close button (pressed) |
| `xpElement_WindowDragBar` | 61 | Window title bar |
| `xpElement_WindowDragBarSmooth` | 62 | Smooth title bar |

#### Navigation Elements

| Element | Value | Description |
|---------|-------|-------------|
| `xpElement_LittleDownArrow` | 53 | Small down arrow |
| `xpElement_LittleUpArrow` | 54 | Small up arrow |

#### Map/Navigation Icons

| Element | Value | Description |
|---------|-------|-------------|
| `xpElement_Airport` | 29 | Airport symbol |
| `xpElement_Waypoint` | 30 | Waypoint marker |
| `xpElement_NDB` | 31 | NDB station |
| `xpElement_VOR` | 32 | VOR station |
| `xpElement_VORWithCompassRose` | 49 | VOR with compass |
| `xpElement_ILSGlideScope` | 27 | ILS approach |
| `xpElement_RadioTower` | 33 | Radio tower |

#### Objects and Structures

| Element | Value | Description |
|---------|-------|-------------|
| `xpElement_Ship` | 26 | Ship icon |
| `xpElement_AircraftCarrier` | 34 | Aircraft carrier |
| `xpElement_OilPlatform` | 24 | Large oil platform |
| `xpElement_OilPlatformSmall` | 25 | Small oil platform |
| `xpElement_Building` | 40 | Building icon |
| `xpElement_CoolingTower` | 38 | Cooling tower |
| `xpElement_SmokeStack` | 39 | Smoke stack |
| `xpElement_PowerLine` | 41 | Power line |
| `xpElement_Fire` | 35 | Fire/emergency |

#### Utility Elements

| Element | Value | Description |
|---------|-------|-------------|
| `xpElement_CustomObject` | 37 | Generic custom marker |
| `xpElement_MarkerLeft` | 28 | Left-pointing marker |
| `xpElement_MarkerRight` | 36 | Right-pointing marker |

### XPDrawElement

```c
WIDGET_API void XPDrawElement(
    int                  inX1,        // Left edge
    int                  inY1,        // Bottom edge  
    int                  inX2,        // Right edge
    int                  inY2,        // Top edge
    XPElementStyle       inStyle,     // Element type
    int                  inLit        // 1 = lit/highlighted, 0 = normal
);
```

Draws a UI element with specified style and highlight state.

**Usage Example:**

```c
void DrawCustomButton(int left, int bottom, int right, int top, int highlighted) {
    // Draw button background
    XPDrawElement(left, bottom, right, top, 
                  highlighted ? xpElement_PushButtonLit : xpElement_PushButton, 0);
    
    // Draw button text
    XPLMDrawString(RGB(1,1,1), left + 10, top - 15, "Click Me", 
                   NULL, xplmFont_Proportional);
}

int ButtonCallback(XPWidgetMessage inMessage, XPWidgetID inWidget,
                   intptr_t inParam1, intptr_t inParam2) {
    static int isHighlighted = 0;
    
    switch (inMessage) {
        case xpMsg_Draw: {
            int left, top, right, bottom;
            XPGetWidgetGeometry(inWidget, &left, &top, &right, &bottom);
            DrawCustomButton(left, bottom, right, top, isHighlighted);
            return 1;
        }
        
        case xpMsg_MouseDown:
            isHighlighted = 1;
            return 1;
            
        case xpMsg_MouseUp:
            isHighlighted = 0;
            return 1;
    }
    return 0;
}
```

### XPGetElementDefaultDimensions

```c
WIDGET_API void XPGetElementDefaultDimensions(
    XPElementStyle       inStyle,
    int *                outWidth,      // Can be NULL
    int *                outHeight,     // Can be NULL  
    int *                outCanBeLit    // Can be NULL
);
```

Returns the recommended dimensions and lighting capability for an element.

**Usage Example:**

```c
void CreateStandardButton(int x, int y, const char* text) {
    int width, height, canBeLit;
    XPGetElementDefaultDimensions(xpElement_PushButton, &width, &height, &canBeLit);
    
    // Adjust width for text
    int textWidth = XPLMGetStringWidth(text, xplmFont_Proportional);
    width = max(width, textWidth + 20); // Add padding
    
    XPWidgetID button = XPCreateWidget(x, y + height, x + width, y,
                                       1, text, 0, parentWidget, xpWidgetClass_Button);
}
```

## Track Controls

Tracks are value-based controls like scrollbars and sliders.

### XPTrackStyle

```c
typedef int XPTrackStyle;
```

Defines different types of track controls:

| Style | Value | Description | Can Light | Rotatable |
|-------|-------|-------------|-----------|-----------|
| `xpTrack_ScrollBar` | 0 | Standard scrollbar with arrows | Yes | Yes |
| `xpTrack_Slider` | 1 | Simple slider control | Yes | Yes |
| `xpTrack_Progress` | 2 | Progress indicator | No | No |

### XPDrawTrack

```c
WIDGET_API void XPDrawTrack(
    int                  inX1,          // Left edge
    int                  inY1,          // Bottom edge
    int                  inX2,          // Right edge  
    int                  inY2,          // Top edge
    int                  inMin,         // Minimum value
    int                  inMax,         // Maximum value
    int                  inValue,       // Current value
    XPTrackStyle         inTrackStyle,  // Track type
    int                  inLit          // 1 = highlighted
);
```

Draws a track control with the specified value and range.

**Usage Example:**

```c
typedef struct {
    int minValue;
    int maxValue;
    int currentValue;
    int isDragging;
} SliderData;

int SliderCallback(XPWidgetMessage inMessage, XPWidgetID inWidget,
                   intptr_t inParam1, intptr_t inParam2) {
    SliderData* data = (SliderData*)XPGetWidgetProperty(inWidget, xpProperty_Refcon, NULL);
    
    switch (inMessage) {
        case xpMsg_Create: {
            data = malloc(sizeof(SliderData));
            data->minValue = 0;
            data->maxValue = 100;
            data->currentValue = 50;
            data->isDragging = 0;
            XPSetWidgetProperty(inWidget, xpProperty_Refcon, (intptr_t)data);
            return 1;
        }
        
        case xpMsg_Draw: {
            if (!data) return 1;
            
            int left, top, right, bottom;
            XPGetWidgetGeometry(inWidget, &left, &top, &right, &bottom);
            
            // Draw slider track
            XPDrawTrack(left, bottom, right, top,
                        data->minValue, data->maxValue, data->currentValue,
                        xpTrack_Slider, data->isDragging);
            return 1;
        }
        
        case xpMsg_MouseDown: {
            data->isDragging = 1;
            // Calculate new value based on mouse position
            int mouseX = MOUSE_X(inParam1);
            int left, top, right, bottom;
            XPGetWidgetGeometry(inWidget, &left, &top, &right, &bottom);
            
            float ratio = (float)(mouseX - left) / (float)(right - left);
            data->currentValue = data->minValue + 
                                (int)(ratio * (data->maxValue - data->minValue));
            
            // Clamp to range
            if (data->currentValue < data->minValue) data->currentValue = data->minValue;
            if (data->currentValue > data->maxValue) data->currentValue = data->maxValue;
            
            return 1;
        }
        
        case xpMsg_MouseUp:
            data->isDragging = 0;
            return 1;
            
        case xpMsg_Destroy:
            if (data) free(data);
            return 1;
    }
    return 0;
}
```

### XPGetTrackDefaultDimensions

```c
WIDGET_API void XPGetTrackDefaultDimensions(
    XPTrackStyle         inStyle,
    int *                outWidth,       // Minimum width for track
    int *                outCanBeLit     // Whether track can be highlighted
);
```

Returns the minimum width and lighting capability for a track control.

### XPGetTrackMetrics

```c
WIDGET_API void XPGetTrackMetrics(
    int                  inX1,              // Left edge
    int                  inY1,              // Bottom edge
    int                  inX2,              // Right edge
    int                  inY2,              // Top edge
    int                  inMin,             // Minimum value
    int                  inMax,             // Maximum value  
    int                  inValue,           // Current value
    XPTrackStyle         inTrackStyle,      // Track type
    int *                outIsVertical,     // 1 = vertical orientation
    int *                outDownBtnSize,    // Size of decrease button
    int *                outDownPageSize,   // Size of decrease area
    int *                outThumbSize,      // Size of thumb/slider
    int *                outUpPageSize,     // Size of increase area
    int *                outUpBtnSize       // Size of increase button
);
```

Returns detailed metrics about how a track control is laid out, useful for implementing custom track interaction.

**Usage Example:**

```c
void HandleScrollBarClick(XPWidgetID scrollBar, int mouseX, int mouseY) {
    int left, top, right, bottom;
    XPGetWidgetGeometry(scrollBar, &left, &top, &right, &bottom);
    
    int min = 0, max = 100, value = 50; // Get current values
    int isVertical, downBtnSize, downPageSize, thumbSize, upPageSize, upBtnSize;
    
    XPGetTrackMetrics(left, bottom, right, top, min, max, value, xpTrack_ScrollBar,
                      &isVertical, &downBtnSize, &downPageSize, 
                      &thumbSize, &upPageSize, &upBtnSize);
    
    if (isVertical) {
        int clickPos = mouseY - bottom;
        if (clickPos < downBtnSize) {
            // Clicked down button
            value = max(min, value - 1);
        } else if (clickPos < downBtnSize + downPageSize) {
            // Clicked in down page area
            value = max(min, value - 10);
        } else if (clickPos < downBtnSize + downPageSize + thumbSize) {
            // Clicked on thumb - start dragging
        } else if (clickPos < downBtnSize + downPageSize + thumbSize + upPageSize) {
            // Clicked in up page area
            value = min(max, value + 10);
        } else {
            // Clicked up button
            value = min(max, value + 1);
        }
    } else {
        // Handle horizontal scrollbar similarly
        int clickPos = mouseX - left;
        // ... similar logic for horizontal orientation
    }
}
```

## Complete Implementation Examples

### Custom Window with Mixed Elements

```c
typedef struct {
    int progressValue;
    int sliderValue;
    int checkboxChecked;
} CustomWindowData;

int CustomWindowCallback(XPWidgetMessage inMessage, XPWidgetID inWidget,
                         intptr_t inParam1, intptr_t inParam2) {
    CustomWindowData* data = (CustomWindowData*)XPGetWidgetProperty(inWidget, 
                                                 xpProperty_Refcon, NULL);
    
    switch (inMessage) {
        case xpMsg_Create: {
            data = malloc(sizeof(CustomWindowData));
            data->progressValue = 0;
            data->sliderValue = 50;
            data->checkboxChecked = 0;
            XPSetWidgetProperty(inWidget, xpProperty_Refcon, (intptr_t)data);
            return 1;
        }
        
        case xpMsg_Draw: {
            if (!data) return 1;
            
            int left, top, right, bottom;
            XPGetWidgetGeometry(inWidget, &left, &top, &right, &bottom);
            
            // Draw main window background
            XPDrawWindow(left, bottom, right, top, xpWindow_MainWindow);
            
            // Draw title bar
            XPDrawElement(left, top - 25, right, top, xpElement_WindowDragBar, 0);
            XPLMDrawString(RGB(1,1,1), left + 10, top - 15, "Custom Window", 
                           NULL, xplmFont_Proportional);
            
            // Draw close button
            XPDrawElement(right - 20, top - 20, right - 5, top - 5, 
                          xpElement_WindowCloseBox, 0);
            
            // Draw content panel
            XPDrawWindow(left + 10, bottom + 10, right - 10, top - 35, 
                         xpWindow_SubWindow);
            
            // Draw progress bar
            XPDrawTrack(left + 20, top - 60, right - 20, top - 40,
                        0, 100, data->progressValue, xpTrack_Progress, 0);
            XPLMDrawString(RGB(0,0,0), left + 20, top - 35, "Progress:", 
                           NULL, xplmFont_Proportional);
            
            // Draw slider
            XPDrawTrack(left + 20, top - 100, right - 20, top - 80,
                        0, 100, data->sliderValue, xpTrack_Slider, 0);
            XPLMDrawString(RGB(0,0,0), left + 20, top - 75, "Slider:", 
                           NULL, xplmFont_Proportional);
            
            // Draw checkbox
            XPDrawElement(left + 20, top - 130, left + 35, top - 115,
                          data->checkboxChecked ? xpElement_CheckBoxLit : xpElement_CheckBox, 0);
            XPLMDrawString(RGB(0,0,0), left + 45, top - 125, "Enable Feature", 
                           NULL, xplmFont_Proportional);
            
            // Draw some navigation icons
            XPDrawElement(left + 20, bottom + 20, left + 35, bottom + 35, 
                          xpElement_Airport, 0);
            XPDrawElement(left + 40, bottom + 20, left + 55, bottom + 35, 
                          xpElement_VOR, 0);
            XPDrawElement(left + 60, bottom + 20, left + 75, bottom + 35, 
                          xpElement_Waypoint, 0);
            
            return 1;
        }
        
        case xpMsg_MouseDown: {
            if (!data) return 0;
            
            int mouseX = MOUSE_X(inParam1);
            int mouseY = MOUSE_Y(inParam1);
            int left, top, right, bottom;
            XPGetWidgetGeometry(inWidget, &left, &top, &right, &bottom);
            
            // Check close button
            if (IN_RECT(mouseX, mouseY, right - 20, top - 5, right - 5, top - 20)) {
                XPHideWidget(inWidget);
                return 1;
            }
            
            // Check checkbox
            if (IN_RECT(mouseX, mouseY, left + 20, top - 115, left + 35, top - 130)) {
                data->checkboxChecked = !data->checkboxChecked;
                return 1;
            }
            
            // Check slider
            if (IN_RECT(mouseX, mouseY, left + 20, top - 80, right - 20, top - 100)) {
                float ratio = (float)(mouseX - (left + 20)) / (float)((right - 20) - (left + 20));
                data->sliderValue = (int)(ratio * 100);
                if (data->sliderValue < 0) data->sliderValue = 0;
                if (data->sliderValue > 100) data->sliderValue = 100;
                return 1;
            }
            
            return 0;
        }
        
        case xpMsg_Destroy:
            if (data) free(data);
            return 1;
    }
    
    return 0;
}
```

### Animated Progress Dialog

```c
static XPWidgetID gProgressDialog = NULL;
static float gProgressValue = 0.0f;
static int gAnimationActive = 0;

float ProgressAnimationCallback(float inElapsedSinceLastCall, 
                                float inElapsedTimeSinceLastFlightLoop,
                                int inCounter, void* inRefcon) {
    if (!gAnimationActive) return 0; // Stop animation
    
    gProgressValue += inElapsedSinceLastCall * 20.0f; // 20 units per second
    if (gProgressValue >= 100.0f) {
        gProgressValue = 100.0f;
        gAnimationActive = 0; // Stop when complete
    }
    
    return 0.1f; // Call again in 0.1 seconds
}

int ProgressDialogCallback(XPWidgetMessage inMessage, XPWidgetID inWidget,
                           intptr_t inParam1, intptr_t inParam2) {
    switch (inMessage) {
        case xpMsg_Draw: {
            int left, top, right, bottom;
            XPGetWidgetGeometry(inWidget, &left, &top, &right, &bottom);
            
            // Draw dialog background
            XPDrawWindow(left, bottom, right, top, xpWindow_MainWindow);
            
            // Draw title
            XPLMDrawString(RGB(1,1,1), left + 10, top - 20, "Processing...", 
                           NULL, xplmFont_Proportional);
            
            // Draw progress bar
            XPDrawTrack(left + 10, bottom + 20, right - 10, bottom + 40,
                        0, 100, (int)gProgressValue, xpTrack_Progress, 0);
            
            // Draw percentage text
            char progressText[32];
            sprintf(progressText, "%.0f%%", gProgressValue);
            int textWidth = XPLMGetStringWidth(progressText, xplmFont_Proportional);
            int centerX = (left + right) / 2 - textWidth / 2;
            XPLMDrawString(RGB(0,0,0), centerX, bottom + 50, progressText, 
                           NULL, xplmFont_Proportional);
            
            return 1;
        }
    }
    return 0;
}

void StartProgressDialog() {
    if (!gProgressDialog) {
        gProgressDialog = XPCreateWidget(200, 300, 500, 200, 1, "Progress", 1, 0,
                                         xpWidgetClass_MainWindow);
        XPAddWidgetCallback(gProgressDialog, ProgressDialogCallback);
    }
    
    gProgressValue = 0.0f;
    gAnimationActive = 1;
    XPShowWidget(gProgressDialog);
    
    // Start animation
    XPLMRegisterFlightLoopCallback(ProgressAnimationCallback, 0.1f, NULL);
}
```

## Platform and Version Considerations

### X-Plane Version Differences

- **X-Plane 6**: Limited shadow compositing, specific layering rules
- **X-Plane 7+**: Full compositing support, elements can be placed anywhere
- Some elements have different appearances between versions
- Always test with multiple X-Plane versions if supporting older releases

### Scaling and Resolution

- Elements automatically scale for different screen resolutions
- Use `XPGetElementDefaultDimensions()` to get appropriate sizes
- Consider high-DPI displays when calculating dimensions
- Test UI on different monitor configurations

## Best Practices

### Visual Design

1. **Follow X-Plane conventions**: Use standard element styles for consistency
2. **Maintain proper hierarchies**: Layer windows correctly (sub over main, etc.)
3. **Use appropriate backgrounds**: Match element requirements (metal, any, etc.)
4. **Provide visual feedback**: Use lit variants for interactive states

### Performance Considerations

1. **Cache dimensions**: Store element dimensions rather than querying repeatedly
2. **Minimize overdraw**: Don't draw hidden elements unnecessarily
3. **Use appropriate drawing calls**: Don't over-engineer simple interfaces
4. **Batch similar operations**: Group related drawing operations together

### Implementation Guidelines

1. **Handle all message types**: Properly implement draw, create, and destroy messages
2. **Validate coordinates**: Ensure drawing bounds are reasonable
3. **Test edge cases**: Verify behavior with minimal and maximal dimensions
4. **Clean up resources**: Free allocated memory in destroy messages

### Accessibility and Usability

1. **Provide adequate spacing**: Don't crowd elements together
2. **Use readable fonts**: Choose appropriate font sizes for text
3. **Support standard interactions**: Implement expected mouse and keyboard behaviors
4. **Consider color-blind users**: Don't rely solely on color for information

## Common Patterns and Idioms

### Custom Button Implementation

```c
void DrawCustomButton(int left, int bottom, int right, int top, 
                      const char* text, int highlighted, int pressed) {
    // Choose appropriate button state
    XPElementStyle buttonStyle = xpElement_PushButton;
    if (highlighted || pressed) {
        buttonStyle = xpElement_PushButtonLit;
    }
    
    // Draw button background
    XPDrawElement(left, bottom, right, top, buttonStyle, 0);
    
    // Draw button text centered
    int textWidth = XPLMGetStringWidth(text, xplmFont_Proportional);
    int textX = left + (right - left) / 2 - textWidth / 2;
    int textY = bottom + (top - bottom) / 2 - 6; // Rough font height / 2
    
    // Offset text slightly when pressed
    if (pressed) {
        textX += 1;
        textY -= 1;
    }
    
    XPLMDrawString(RGB(0,0,0), textX, textY, text, NULL, xplmFont_Proportional);
}
```

### Window with Resizable Content

```c
void DrawResizableWindow(XPWidgetID widget) {
    int left, top, right, bottom;
    XPGetWidgetGeometry(widget, &left, &top, &right, &bottom);
    
    // Main window background
    XPDrawWindow(left, bottom, right, top, xpWindow_MainWindow);
    
    // Title bar (fixed height)
    int titleHeight = 25;
    XPDrawElement(left, top - titleHeight, right, top, 
                  xpElement_WindowDragBar, 0);
    
    // Content area (scalable)
    int contentMargin = 10;
    XPDrawWindow(left + contentMargin, bottom + contentMargin,
                 right - contentMargin, top - titleHeight - contentMargin,
                 xpWindow_SubWindow);
    
    // Resize corner indicator (optional)
    XPDrawElement(right - 15, bottom, right, bottom + 15,
                  xpElement_CustomObject, 0);
}
```

## Related Headers

- `XPWidgetDefs.h` - Core widget definitions and message types
- `XPWidgets.h` - Main widget creation and management API  
- `XPWidgetUtils.h` - Utility functions and convenience macros
- `XPStandardWidgets.h` - Standard widget classes that use these graphics functions
- `XPLMGraphics.h` - Lower-level OpenGL graphics functions
- `XPLMDisplay.h` - Window and display management
