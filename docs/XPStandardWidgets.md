# XPStandardWidgets.h - X-Plane 12 Standard Widget Classes

## Overview

The `XPStandardWidgets.h` header defines the standard widget classes provided by the X-Plane 12 Widget Framework. These pre-built widgets handle common UI patterns like windows, buttons, text fields, and other standard interface elements with built-in behavior and styling.

## Theory of Operation

Standard widgets are created using `XPCreateWidget()` with predefined widget class constants. Each widget class has:

1. **Built-in Behavior**: Automatic handling of user interactions
2. **Customizable Properties**: Settings that control appearance and behavior
3. **Event Messages**: Notifications sent when user interactions occur
4. **Visual Styling**: Consistent appearance matching X-Plane's UI

### Widget Class Architecture

- Widget classes are identified by integer constants (e.g., `xpWidgetClass_MainWindow`)
- Each class implements a standard widget function that handles messages
- Properties control specific aspects of widget behavior and appearance
- Messages are sent up the widget hierarchy when events occur

## Main Window Widget

The main window widget creates top-level plugin windows that users can interact with.

### Widget Class

```c
#define xpWidgetClass_MainWindow 1
```

### Window Types

```c
enum {
    xpMainWindowStyle_MainWindow    = 0,    // Standard dialog window
    xpMainWindowStyle_Translucent   = 1     // Semi-transparent dark window
};
```

### Properties

| Property | ID | Type | Description |
|----------|----|----|-------------|
| `xpProperty_MainWindowType` | 1100 | int | Window visual style (see types above) |
| `xpProperty_MainWindowHasCloseBoxes` | 1200 | int | 1 = show close buttons in corners |

### Messages

| Message | ID | Parameters | Description |
|---------|----|-----------| ------------|
| `xpMessage_CloseButtonPushed` | 1200 | None | Close button was clicked |

### Usage Example

```c
// Create main window
XPWidgetID mainWindow = XPCreateWidget(
    100, 400, 500, 200,              // Geometry
    1,                               // Visible
    "My Plugin Window",              // Title
    1,                               // Root widget
    0,                               // No container
    xpWidgetClass_MainWindow         // Widget class
);

// Configure window properties
XPSetWidgetProperty(mainWindow, xpProperty_MainWindowType, xpMainWindowStyle_MainWindow);
XPSetWidgetProperty(mainWindow, xpProperty_MainWindowHasCloseBoxes, 1);

// Handle close button
int WindowHandler(XPWidgetMessage inMessage, XPWidgetID inWidget,
                  intptr_t inParam1, intptr_t inParam2) {
    if (inMessage == xpMessage_CloseButtonPushed) {
        XPHideWidget(inWidget);
        return 1;
    }
    return 0;
}

XPAddWidgetCallback(mainWindow, WindowHandler);
```

## Sub Window Widget

Sub windows create panels and sections within main windows.

### Widget Class

```c
#define xpWidgetClass_SubWindow 2
```

### Window Types

```c
enum {
    xpSubWindowStyle_SubWindow  = 0,    // Panel within main window
    xpSubWindowStyle_Screen     = 2,    // LCD screen for text display  
    xpSubWindowStyle_ListView   = 3     // Scrolling list background
};
```

### Properties

| Property | ID | Description |
|----------|----| ------------|
| `xpProperty_SubWindowType` | 1200 | Sub window visual style |

### Usage Example

```c
// Create content panel
XPWidgetID contentPanel = XPCreateWidget(
    110, 390, 490, 210,              // Inside main window
    1, "", 0, mainWindow,            // Visible, no text, child of main window
    xpWidgetClass_SubWindow
);

XPSetWidgetProperty(contentPanel, xpProperty_SubWindowType, xpSubWindowStyle_SubWindow);

// Create info screen
XPWidgetID infoScreen = XPCreateWidget(
    120, 380, 300, 220,
    1, "", 0, contentPanel,
    xpWidgetClass_SubWindow
);

XPSetWidgetProperty(infoScreen, xpProperty_SubWindowType, xpSubWindowStyle_Screen);
```

## Button Widget

The button widget provides clickable buttons with various visual styles and behaviors.

### Widget Class

```c
#define xpWidgetClass_Button 3
```

### Button Types (Visual Appearance)

```c
enum {
    xpPushButton        = 0,    // Standard push button
    xpRadioButton       = 1,    // Check box or radio button style
    xpWindowCloseBox    = 3,    // Window close button
    xpLittleDownArrow   = 5,    // Small down arrow
    xpLittleUpArrow     = 6     // Small up arrow
};
```

### Button Behaviors (Interaction Model)

```c
enum {
    xpButtonBehaviorPushButton   = 0,   // Highlight while pressed, send message on release
    xpButtonBehaviorCheckBox     = 1,   // Toggle state immediately on click
    xpButtonBehaviorRadioButton  = 2    // Set to 1 on click (manual radio group management)
};
```

### Properties

| Property | ID | Description |
|----------|----| ------------|
| `xpProperty_ButtonType` | 1300 | Visual appearance (see types above) |
| `xpProperty_ButtonBehavior` | 1301 | Interaction behavior (see behaviors above) |
| `xpProperty_ButtonState` | 1302 | For check/radio buttons: 0 = unchecked, 1 = checked |

### Messages

| Message | ID | Param1 | Param2 | Description |
|---------|----| -------|--------| ------------|
| `xpMsg_PushButtonPressed` | 1300 | Widget ID | - | Push button was clicked and released |
| `xpMsg_ButtonStateChanged` | 1301 | Widget ID | New state | Check/radio button state changed |

### Usage Examples

#### Push Button

```c
XPWidgetID okButton = XPCreateWidget(
    200, 250, 280, 230,
    1, "OK", 0, parentWidget,
    xpWidgetClass_Button
);

XPSetWidgetProperty(okButton, xpProperty_ButtonType, xpPushButton);
XPSetWidgetProperty(okButton, xpProperty_ButtonBehavior, xpButtonBehaviorPushButton);

int ButtonHandler(XPWidgetMessage inMessage, XPWidgetID inWidget,
                  intptr_t inParam1, intptr_t inParam2) {
    if (inMessage == xpMsg_PushButtonPressed) {
        if (inWidget == okButton) {
            // Handle OK button click
            ProcessOKAction();
        }
        return 1;
    }
    return 0;
}

XPAddWidgetCallback(okButton, ButtonHandler);
```

#### Check Box

```c
XPWidgetID enableCheckbox = XPCreateWidget(
    120, 300, 135, 285,
    1, "Enable Feature", 0, parentWidget,
    xpWidgetClass_Button
);

XPSetWidgetProperty(enableCheckbox, xpProperty_ButtonType, xpRadioButton);
XPSetWidgetProperty(enableCheckbox, xpProperty_ButtonBehavior, xpButtonBehaviorCheckBox);
XPSetWidgetProperty(enableCheckbox, xpProperty_ButtonState, 1); // Initially checked

int CheckboxHandler(XPWidgetMessage inMessage, XPWidgetID inWidget,
                    intptr_t inParam1, intptr_t inParam2) {
    if (inMessage == xpMsg_ButtonStateChanged) {
        XPWidgetID buttonWidget = (XPWidgetID)inParam1;
        int newState = (int)inParam2;
        
        if (buttonWidget == enableCheckbox) {
            // Handle checkbox state change
            SetFeatureEnabled(newState);
        }
        return 1;
    }
    return 0;
}
```

#### Radio Button Group

```c
static XPWidgetID radioButtons[3];

void CreateRadioGroup() {
    const char* labels[] = {"Option A", "Option B", "Option C"};
    
    for (int i = 0; i < 3; i++) {
        radioButtons[i] = XPCreateWidget(
            120, 350 - (i * 25), 135, 335 - (i * 25),
            1, labels[i], 0, parentWidget,
            xpWidgetClass_Button
        );
        
        XPSetWidgetProperty(radioButtons[i], xpProperty_ButtonType, xpRadioButton);
        XPSetWidgetProperty(radioButtons[i], xpProperty_ButtonBehavior, xpButtonBehaviorRadioButton);
        XPAddWidgetCallback(radioButtons[i], RadioGroupHandler);
    }
    
    // Set first option as selected
    XPSetWidgetProperty(radioButtons[0], xpProperty_ButtonState, 1);
}

int RadioGroupHandler(XPWidgetMessage inMessage, XPWidgetID inWidget,
                      intptr_t inParam1, intptr_t inParam2) {
    if (inMessage == xpMsg_ButtonStateChanged) {
        XPWidgetID buttonWidget = (XPWidgetID)inParam1;
        int newState = (int)inParam2;
        
        if (newState == 1) {
            // Clear other radio buttons in group
            for (int i = 0; i < 3; i++) {
                if (radioButtons[i] != buttonWidget) {
                    XPSetWidgetProperty(radioButtons[i], xpProperty_ButtonState, 0);
                }
            }
            
            // Handle selection change
            for (int i = 0; i < 3; i++) {
                if (radioButtons[i] == buttonWidget) {
                    HandleOptionSelected(i);
                    break;
                }
            }
        }
        return 1;
    }
    return 0;
}
```

## Text Field Widget

The text field widget provides editable text input with selection and keyboard navigation.

### Widget Class

```c
#define xpWidgetClass_TextField 4
```

### Text Field Types

```c
enum {
    xpTextEntryField    = 0,    // Standard text input field
    xpTextTransparent   = 3,    // Transparent text (no background)
    xpTextTranslucent   = 4     // Semi-transparent dark background
};
```

### Properties

| Property | ID | Description |
|----------|----| ------------|
| `xpProperty_EditFieldSelStart` | 1400 | Selection start position (character index) |
| `xpProperty_EditFieldSelEnd` | 1401 | Selection end position |
| `xpProperty_EditFieldSelDragStart` | 1402 | Drag start position (-1 if not dragging) |
| `xpProperty_TextFieldType` | 1403 | Visual style (see types above) |
| `xpProperty_PasswordMode` | 1404 | 1 = show asterisks, 0 = show actual text |
| `xpProperty_MaxCharacters` | 1405 | Maximum length (0 = unlimited) |
| `xpProperty_ScrollPosition` | 1406 | First visible character (horizontal scroll) |
| `xpProperty_Font` | 1407 | Font ID (XPLMFontID) |
| `xpProperty_ActiveEditSide` | 1408 | Active side of selection (internal) |

### Messages

| Message | ID | Param1 | Description |
|---------|----| -------| ------------|
| `xpMsg_TextFieldChanged` | 1400 | Widget ID | Text content was modified |

### Usage Examples

#### Basic Text Field

```c
XPWidgetID nameField = XPCreateWidget(
    120, 350, 400, 330,
    1, "Enter your name", 0, parentWidget,
    xpWidgetClass_TextField
);

XPSetWidgetProperty(nameField, xpProperty_TextFieldType, xpTextEntryField);
XPSetWidgetProperty(nameField, xpProperty_MaxCharacters, 50);

int TextFieldHandler(XPWidgetMessage inMessage, XPWidgetID inWidget,
                     intptr_t inParam1, intptr_t inParam2) {
    if (inMessage == xpMsg_TextFieldChanged) {
        if (inWidget == nameField) {
            char text[256];
            XPGetWidgetDescriptor(inWidget, text, sizeof(text));
            // Handle text change
            ValidateInput(text);
        }
        return 1;
    }
    return 0;
}

XPAddWidgetCallback(nameField, TextFieldHandler);
```

#### Password Field

```c
XPWidgetID passwordField = XPCreateWidget(
    120, 300, 400, 280,
    1, "", 0, parentWidget,
    xpWidgetClass_TextField
);

XPSetWidgetProperty(passwordField, xpProperty_TextFieldType, xpTextEntryField);
XPSetWidgetProperty(passwordField, xpProperty_PasswordMode, 1);
XPSetWidgetProperty(passwordField, xpProperty_MaxCharacters, 32);
```

#### Custom Text Validation

```c
int ValidatingTextFieldHandler(XPWidgetMessage inMessage, XPWidgetID inWidget,
                               intptr_t inParam1, intptr_t inParam2) {
    if (inMessage == xpMsg_KeyPress) {
        XPKeyState_t* key = (XPKeyState_t*)inParam1;
        
        if (key->flags & xplm_DownFlag) {
            // Allow only numeric input
            if (key->key >= '0' && key->key <= '9') {
                return 0; // Allow the key
            } else if (key->key == '\b' || key->key == '\t' || key->key == '\r') {
                return 0; // Allow backspace, tab, enter
            } else {
                return 1; // Block other keys
            }
        }
    }
    
    if (inMessage == xpMsg_TextFieldChanged) {
        // Validate complete input
        char text[256];
        XPGetWidgetDescriptor(inWidget, text, sizeof(text));
        
        int value = atoi(text);
        if (value < 0 || value > 100) {
            // Reset to valid range
            snprintf(text, sizeof(text), "%d", (value < 0) ? 0 : 100);
            XPSetWidgetDescriptor(inWidget, text);
        }
        return 1;
    }
    
    return 0;
}
```

## Scroll Bar Widget

The scroll bar widget provides value selection through draggable sliders.

### Widget Class

```c
#define xpWidgetClass_ScrollBar 5
```

### Scroll Bar Types

```c
enum {
    xpScrollBarTypeScrollBar = 0,   // Full scrollbar with arrows
    xpScrollBarTypeSlider    = 1    // Simple slider without arrows
};
```

### Properties

| Property | ID | Description |
|----------|----| ------------|
| `xpProperty_ScrollBarSliderPosition` | 1500 | Current value |
| `xpProperty_ScrollBarMin` | 1501 | Minimum value |
| `xpProperty_ScrollBarMax` | 1502 | Maximum value |
| `xpProperty_ScrollBarPageAmount` | 1503 | Amount to scroll when clicking beside thumb |
| `xpProperty_ScrollBarType` | 1504 | Visual style (see types above) |
| `xpProperty_ScrollBarSlop` | 1505 | Internal use |

### Messages

| Message | ID | Param1 | Description |
|---------|----| -------| ------------|
| `xpMsg_ScrollBarSliderPositionChanged` | 1500 | Widget ID | Slider value changed |

### Usage Examples

#### Horizontal Slider

```c
static int gSliderValue = 50;

XPWidgetID volumeSlider = XPCreateWidget(
    120, 280, 400, 260,
    1, "", 0, parentWidget,
    xpWidgetClass_ScrollBar
);

XPSetWidgetProperty(volumeSlider, xpProperty_ScrollBarType, xpScrollBarTypeSlider);
XPSetWidgetProperty(volumeSlider, xpProperty_ScrollBarMin, 0);
XPSetWidgetProperty(volumeSlider, xpProperty_ScrollBarMax, 100);
XPSetWidgetProperty(volumeSlider, xpProperty_ScrollBarSliderPosition, gSliderValue);
XPSetWidgetProperty(volumeSlider, xpProperty_ScrollBarPageAmount, 10);

int SliderHandler(XPWidgetMessage inMessage, XPWidgetID inWidget,
                  intptr_t inParam1, intptr_t inParam2) {
    if (inMessage == xpMsg_ScrollBarSliderPositionChanged) {
        if (inWidget == volumeSlider) {
            gSliderValue = XPGetWidgetProperty(inWidget, xpProperty_ScrollBarSliderPosition, NULL);
            
            // Update volume
            SetAudioVolume(gSliderValue / 100.0f);
            
            // Update display
            char valueText[32];
            snprintf(valueText, sizeof(valueText), "Volume: %d%%", gSliderValue);
            XPSetWidgetDescriptor(volumeLabel, valueText);
        }
        return 1;
    }
    return 0;
}

XPAddWidgetCallback(volumeSlider, SliderHandler);
```

#### Vertical Scroll Bar

```c
XPWidgetID listScrollBar = XPCreateWidget(
    480, 400, 495, 200,  // Tall and narrow for vertical
    1, "", 0, parentWidget,
    xpWidgetClass_ScrollBar
);

XPSetWidgetProperty(listScrollBar, xpProperty_ScrollBarType, xpScrollBarTypeScrollBar);
XPSetWidgetProperty(listScrollBar, xpProperty_ScrollBarMin, 0);
XPSetWidgetProperty(listScrollBar, xpProperty_ScrollBarMax, 50); // 50 items
XPSetWidgetProperty(listScrollBar, xpProperty_ScrollBarSliderPosition, 0);
XPSetWidgetProperty(listScrollBar, xpProperty_ScrollBarPageAmount, 5); // 5 items per page
```

## Caption Widget

The caption widget displays static text labels.

### Widget Class

```c
#define xpWidgetClass_Caption 6
```

### Properties

| Property | ID | Description |
|----------|----| ------------|
| `xpProperty_CaptionLit` | 1600 | 1 = bright text for dark backgrounds |

### Usage Example

```c
// Create label for text field
XPWidgetID nameLabel = XPCreateWidget(
    20, 350, 115, 330,
    1, "Name:", 0, parentWidget,
    xpWidgetClass_Caption
);

// Create label for screen display
XPWidgetID statusLabel = XPCreateWidget(
    20, 200, 300, 180,
    1, "Status: Ready", 0, screenWidget,
    xpWidgetClass_Caption
);

XPSetWidgetProperty(statusLabel, xpProperty_CaptionLit, 1); // Bright text for screen
```

## General Graphics Widget

The general graphics widget displays icons and symbols.

### Widget Class

```c
#define xpWidgetClass_GeneralGraphics 7
```

### Graphic Types

```c
enum {
    xpShip                   = 4,
    xpILSGlideScope         = 5,
    xpMarkerLeft            = 6,
    xp_Airport              = 7,
    xpNDB                   = 8,
    xpVOR                   = 9,
    xpRadioTower            = 10,
    xpAircraftCarrier       = 11,
    xpFire                  = 12,
    xpMarkerRight           = 13,
    xpCustomObject          = 14,
    xpCoolingTower          = 15,
    xpSmokeStack            = 16,
    xpBuilding              = 17,
    xpPowerLine             = 18,
    xpVORWithCompassRose    = 19,
    xpOilPlatform           = 21,
    xpOilPlatformSmall      = 22,
    xpWayPoint              = 23
};
```

### Properties

| Property | ID | Description |
|----------|----| ------------|
| `xpProperty_GeneralGraphicsType` | 1700 | Icon type (see types above) |

### Usage Example

```c
// Create navigation icons
XPWidgetID airportIcon = XPCreateWidget(
    50, 100, 70, 80,
    1, "", 0, mapWidget,
    xpWidgetClass_GeneralGraphics
);

XPSetWidgetProperty(airportIcon, xpProperty_GeneralGraphicsType, xp_Airport);

XPWidgetID vorIcon = XPCreateWidget(
    150, 120, 170, 100,
    1, "", 0, mapWidget,
    xpWidgetClass_GeneralGraphics
);

XPSetWidgetProperty(vorIcon, xpProperty_GeneralGraphicsType, xpVOR);
```

## Progress Indicator Widget

The progress indicator widget shows task completion status.

### Widget Class

```c
#define xpWidgetClass_Progress 8
```

### Properties

| Property | ID | Description |
|----------|----| ------------|
| `xpProperty_ProgressPosition` | 1800 | Current progress value |
| `xpProperty_ProgressMin` | 1801 | Minimum value (typically 0) |
| `xpProperty_ProgressMax` | 1802 | Maximum value (typically 100) |

### Usage Example

```c
static XPWidgetID gProgressBar = NULL;
static int gProgressValue = 0;

void CreateProgressDialog() {
    // Create progress window
    XPWidgetID progressWindow = XPCreateWidget(
        200, 300, 500, 200,
        1, "Loading...", 1, 0,
        xpWidgetClass_MainWindow
    );
    
    // Create progress bar
    gProgressBar = XPCreateWidget(
        210, 260, 490, 240,
        1, "", 0, progressWindow,
        xpWidgetClass_Progress
    );
    
    XPSetWidgetProperty(gProgressBar, xpProperty_ProgressMin, 0);
    XPSetWidgetProperty(gProgressBar, xpProperty_ProgressMax, 100);
    XPSetWidgetProperty(gProgressBar, xpProperty_ProgressPosition, 0);
    
    // Create status label
    XPWidgetID statusLabel = XPCreateWidget(
        210, 230, 490, 210,
        1, "Initializing...", 0, progressWindow,
        xpWidgetClass_Caption
    );
}

void UpdateProgress(int value, const char* status) {
    if (gProgressBar) {
        XPSetWidgetProperty(gProgressBar, xpProperty_ProgressPosition, value);
        
        // Update status text if provided
        if (status) {
            // Find status label (implementation dependent)
            XPSetWidgetDescriptor(statusLabel, status);
        }
    }
}

// Usage in loading process
void LoadData() {
    CreateProgressDialog();
    
    UpdateProgress(10, "Loading configuration...");
    // Load config...
    
    UpdateProgress(30, "Loading aircraft data...");
    // Load aircraft...
    
    UpdateProgress(60, "Loading scenery...");
    // Load scenery...
    
    UpdateProgress(90, "Finalizing...");
    // Finalize...
    
    UpdateProgress(100, "Complete!");
    
    // Hide progress dialog after a delay
    // Or let user close it
}
```

## Complete Dialog Example

Here's a comprehensive example showing multiple standard widgets working together:

```c
typedef struct {
    XPWidgetID mainWindow;
    XPWidgetID nameField;
    XPWidgetID emailField;
    XPWidgetID enableNotifications;
    XPWidgetID frequencySlider;
    XPWidgetID okButton;
    XPWidgetID cancelButton;
    XPWidgetID statusLabel;
    
    int notificationsEnabled;
    int frequency;
} SettingsDialog;

static SettingsDialog gDialog = {0};

int SettingsDialogHandler(XPWidgetMessage inMessage, XPWidgetID inWidget,
                          intptr_t inParam1, intptr_t inParam2) {
    switch (inMessage) {
        case xpMessage_CloseButtonPushed:
            XPHideWidget(gDialog.mainWindow);
            return 1;
            
        case xpMsg_PushButtonPressed: {
            if (inWidget == gDialog.okButton) {
                // Save settings
                char name[256], email[256];
                XPGetWidgetDescriptor(gDialog.nameField, name, sizeof(name));
                XPGetWidgetDescriptor(gDialog.emailField, email, sizeof(email));
                
                SaveSettings(name, email, gDialog.notificationsEnabled, gDialog.frequency);
                XPSetWidgetDescriptor(gDialog.statusLabel, "Settings saved successfully!");
                XPHideWidget(gDialog.mainWindow);
                
            } else if (inWidget == gDialog.cancelButton) {
                XPHideWidget(gDialog.mainWindow);
            }
            return 1;
        }
        
        case xpMsg_ButtonStateChanged: {
            if (inWidget == gDialog.enableNotifications) {
                gDialog.notificationsEnabled = (int)inParam2;
                
                // Enable/disable frequency slider based on checkbox
                XPSetWidgetProperty(gDialog.frequencySlider, xpProperty_Enabled, 
                                  gDialog.notificationsEnabled);
                
                const char* status = gDialog.notificationsEnabled ? 
                                   "Notifications enabled" : "Notifications disabled";
                XPSetWidgetDescriptor(gDialog.statusLabel, status);
            }
            return 1;
        }
        
        case xpMsg_ScrollBarSliderPositionChanged: {
            if (inWidget == gDialog.frequencySlider) {
                gDialog.frequency = XPGetWidgetProperty(inWidget, 
                                                      xpProperty_ScrollBarSliderPosition, NULL);
                
                char freqText[64];
                snprintf(freqText, sizeof(freqText), "Update frequency: %d minutes", 
                         gDialog.frequency);
                XPSetWidgetDescriptor(gDialog.statusLabel, freqText);
            }
            return 1;
        }
        
        case xpMsg_TextFieldChanged: {
            // Real-time validation
            if (inWidget == gDialog.emailField) {
                char email[256];
                XPGetWidgetDescriptor(inWidget, email, sizeof(email));
                
                if (strstr(email, "@") && strstr(email, ".")) {
                    XPSetWidgetDescriptor(gDialog.statusLabel, "Email format valid");
                } else {
                    XPSetWidgetDescriptor(gDialog.statusLabel, "Please enter a valid email");
                }
            }
            return 1;
        }
    }
    
    return 0;
}

void CreateSettingsDialog() {
    // Main window
    gDialog.mainWindow = XPCreateWidget(
        100, 500, 600, 200,
        1, "Settings", 1, 0,
        xpWidgetClass_MainWindow
    );
    
    XPSetWidgetProperty(gDialog.mainWindow, xpProperty_MainWindowHasCloseBoxes, 1);
    
    // Content panel
    XPWidgetID contentPanel = XPCreateWidget(
        110, 490, 590, 210,
        1, "", 0, gDialog.mainWindow,
        xpWidgetClass_SubWindow
    );
    
    // Name field
    XPCreateWidget(120, 480, 180, 460, 1, "Name:", 0, contentPanel, xpWidgetClass_Caption);
    gDialog.nameField = XPCreateWidget(
        190, 480, 400, 460,
        1, "Enter your name", 0, contentPanel,
        xpWidgetClass_TextField
    );
    
    // Email field  
    XPCreateWidget(120, 450, 180, 430, 1, "Email:", 0, contentPanel, xpWidgetClass_Caption);
    gDialog.emailField = XPCreateWidget(
        190, 450, 400, 430,
        1, "user@example.com", 0, contentPanel,
        xpWidgetClass_TextField
    );
    
    // Notifications checkbox
    gDialog.enableNotifications = XPCreateWidget(
        120, 420, 135, 405,
        1, "Enable Notifications", 0, contentPanel,
        xpWidgetClass_Button
    );
    
    XPSetWidgetProperty(gDialog.enableNotifications, xpProperty_ButtonType, xpRadioButton);
    XPSetWidgetProperty(gDialog.enableNotifications, xpProperty_ButtonBehavior, xpButtonBehaviorCheckBox);
    XPSetWidgetProperty(gDialog.enableNotifications, xpProperty_ButtonState, 1);
    gDialog.notificationsEnabled = 1;
    
    // Frequency slider
    XPCreateWidget(120, 390, 220, 370, 1, "Frequency:", 0, contentPanel, xpWidgetClass_Caption);
    gDialog.frequencySlider = XPCreateWidget(
        230, 390, 400, 370,
        1, "", 0, contentPanel,
        xpWidgetClass_ScrollBar
    );
    
    XPSetWidgetProperty(gDialog.frequencySlider, xpProperty_ScrollBarType, xpScrollBarTypeSlider);
    XPSetWidgetProperty(gDialog.frequencySlider, xpProperty_ScrollBarMin, 1);
    XPSetWidgetProperty(gDialog.frequencySlider, xpProperty_ScrollBarMax, 60);
    XPSetWidgetProperty(gDialog.frequencySlider, xpProperty_ScrollBarSliderPosition, 15);
    gDialog.frequency = 15;
    
    // Status label
    gDialog.statusLabel = XPCreateWidget(
        120, 350, 580, 330,
        1, "Ready", 0, contentPanel,
        xpWidgetClass_Caption
    );
    
    // Buttons
    gDialog.okButton = XPCreateWidget(
        420, 250, 480, 230,
        1, "OK", 0, contentPanel,
        xpWidgetClass_Button
    );
    
    XPSetWidgetProperty(gDialog.okButton, xpProperty_ButtonType, xpPushButton);
    XPSetWidgetProperty(gDialog.okButton, xpProperty_ButtonBehavior, xpButtonBehaviorPushButton);
    
    gDialog.cancelButton = XPCreateWidget(
        490, 250, 560, 230,
        1, "Cancel", 0, contentPanel,
        xpWidgetClass_Button
    );
    
    XPSetWidgetProperty(gDialog.cancelButton, xpProperty_ButtonType, xpPushButton);
    XPSetWidgetProperty(gDialog.cancelButton, xpProperty_ButtonBehavior, xpButtonBehaviorPushButton);
    
    // Add main handler
    XPAddWidgetCallback(gDialog.mainWindow, SettingsDialogHandler);
}
```

## Best Practices

### Widget Selection and Usage

1. **Choose appropriate widget classes**: Use the right widget for the task (buttons for actions, text fields for input)
2. **Follow UI conventions**: Use standard behaviors (push buttons for actions, checkboxes for toggles)
3. **Maintain consistency**: Use similar patterns throughout your plugin interface
4. **Provide feedback**: Update status displays and labels to reflect user actions

### Property Management

1. **Set properties immediately**: Configure widget properties right after creation
2. **Validate property values**: Ensure property values are within valid ranges
3. **Use meaningful defaults**: Set sensible initial values for all properties
4. **Document custom properties**: Clearly document any plugin-specific properties

### Message Handling

1. **Handle all relevant messages**: Implement handlers for messages your widget should respond to
2. **Return appropriate values**: Return 1 when handling messages, 0 when passing through
3. **Check widget identity**: Verify which widget sent a message when handling multiple widgets
4. **Validate parameters**: Check message parameters for validity before using

### Layout and Appearance

1. **Use consistent spacing**: Maintain regular margins and padding between elements
2. **Align elements properly**: Line up related widgets for a clean appearance
3. **Size widgets appropriately**: Use reasonable dimensions for readability and usability
4. **Group related elements**: Use sub-windows to organize related controls

## Common Patterns and Idioms

### Modal Dialog Pattern

```c
void ShowModalDialog(void (*onOK)(void), void (*onCancel)(void)) {
    // Disable parent window interaction
    // Show dialog
    // Handle OK/Cancel callbacks
    // Re-enable parent window
}
```

### Validation Pattern

```c
int ValidateAndProcess(XPWidgetID textField) {
    char text[256];
    XPGetWidgetDescriptor(textField, text, sizeof(text));
    
    if (!IsValidInput(text)) {
        ShowErrorMessage("Invalid input");
        return 0;
    }
    
    ProcessInput(text);
    return 1;
}
```

### Dynamic Content Pattern

```c
void UpdateListContent(XPWidgetID listParent, const char** items, int count) {
    // Remove existing items
    int childCount = XPCountChildWidgets(listParent);
    for (int i = 0; i < childCount; i++) {
        XPWidgetID child = XPGetNthChildWidget(listParent, 0);
        XPDestroyWidget(child, 0);
    }
    
    // Add new items
    for (int i = 0; i < count; i++) {
        CreateListItem(listParent, items[i], i);
    }
}
```

## Related Headers

- `XPWidgetDefs.h` - Core widget definitions and message system
- `XPWidgets.h` - Main widget creation and management API
- `XPWidgetUtils.h` - Utility functions and behavior helpers
- `XPUIGraphics.h` - Low-level graphics functions for custom drawing
- `XPLMDisplay.h` - Underlying window and display system integration
