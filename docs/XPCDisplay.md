# XPCDisplay - C++ Display Wrapper Classes

## Overview

The `XPCDisplay` wrapper classes provide a C++ object-oriented interface to X-Plane's display and windowing system. These classes encapsulate the low-level XPLM display APIs with automatic resource management, event handling, and modern C++ conventions. The wrapper consists of two primary classes: `XPCKeySniffer` for global keyboard interception and `XPCWindow` for window management.

## Files

- **Header**: `CHeaders/Wrappers/XPCDisplay.h`
- **Implementation**: `CHeaders/Wrappers/XPCDisplay.cpp`
- **Dependencies**: `XPLMDisplay.h`

## Architecture Overview

### Design Philosophy

The XPCDisplay wrappers follow the **RAII (Resource Acquisition Is Initialization)** pattern, ensuring proper resource cleanup and exception safety. They provide:

- **Automatic Resource Management**: Constructors register callbacks, destructors unregister them
- **Virtual Interface Pattern**: Pure virtual functions for derived class implementation
- **Static Callback Bridge**: C-style callbacks that dispatch to C++ member functions
- **Type Safety**: Strong typing over raw XPLM handles and void pointers

### Class Hierarchy

```text
XPCKeySniffer (Abstract Base)
    ↓ implements
    HandleKeyStroke() (pure virtual)

XPCWindow (Abstract Base)
    ↓ implements
    DoDraw() (pure virtual)
    HandleKey() (pure virtual)
    LoseFocus() (pure virtual)
    HandleClick() (pure virtual)
```

---

## XPCKeySniffer Class

The `XPCKeySniffer` class provides a C++ interface for intercepting keystrokes at a low level, before or after the window system processes them.

### XPCKeySniffer Declaration

```cpp
class XPCKeySniffer {
public:
    XPCKeySniffer(int inBeforeWindows);
    virtual ~XPCKeySniffer();
  
    virtual int HandleKeyStroke(char inCharKey, XPLMKeyFlags inFlags, char inVirtualKey) = 0;

private:
    int mBeforeWindows;
    static int KeySnifferCB(char inCharKey, XPLMKeyFlags inFlags, char inVirtualKey, void* inRefCon);
};
```

### XPCKeySniffer Constructor

#### XPCKeySniffer(int inBeforeWindows)

Creates a new key sniffer and automatically registers it with X-Plane's keyboard system.

**Parameters**:

- `inBeforeWindows`: Boolean flag (0 or 1)
  - `1` (true): Intercept keys **before** the window system processes them (high priority)
  - `0` (false): Intercept keys **after** the window system processes them (low priority)

**Behavior**:

1. Stores the priority setting in `mBeforeWindows`
2. Calls `XPLMRegisterKeySniffer()` with the static callback bridge
3. Passes `this` pointer as the refcon for callback routing

**Usage Guidelines**:

- **Use `inBeforeWindows = 0`** for most applications - this allows normal window keyboard handling to work first
- **Use `inBeforeWindows = 1`** only for global hotkeys that should override everything
- Constructor automatically handles registration - no manual setup required

**Example**:

```cpp
class MyGlobalHotkeys : public XPCKeySniffer {
public:
    MyGlobalHotkeys() : XPCKeySniffer(0) {  // After window system
        // Constructor automatically registers the sniffer
    }
  
    int HandleKeyStroke(char inCharKey, XPLMKeyFlags inFlags, char inVirtualKey) override {
        if (inVirtualKey == XPLM_VK_F1 && (inFlags & xplm_ShiftFlag)) {
            // Handle Shift+F1
            ShowHelp();
            return 0;  // Consume the key
        }
        return 1;  // Pass through
    }
};
```

### XPCKeySniffer Destructor

#### virtual ~XPCKeySniffer()

Automatically unregisters the key sniffer when the object is destroyed.

**Behavior**:

1. Calls `XPLMUnregisterKeySniffer()` with the same parameters used during registration
2. Ensures proper cleanup even if exceptions occur
3. Safe to call multiple times

**RAII Benefits**:

- No manual cleanup required
- Exception-safe resource management
- Prevents dangling callbacks that could crash X-Plane

### XPCKeySniffer Pure Virtual Methods

#### virtual int HandleKeyStroke(char inCharKey, XPLMKeyFlags inFlags, char inVirtualKey) = 0

**Abstract method** that derived classes must implement to handle intercepted keystrokes.

**Parameters**:

- `inCharKey`: The ASCII character (printable keys) or 0 for non-printable keys
- `inFlags`: Modifier key flags (see `XPLMKeyFlags` in XPLMDefs.h)
  - `xplm_ShiftFlag`: Shift key pressed
  - `xplm_OptionAltFlag`: Alt/Option key pressed
  - `xplm_ControlFlag`: Control key pressed
- `inVirtualKey`: Virtual key code for special keys (see XPLM_VK_* defines)

**Return Value**:

- `1`: Pass the keystroke to the next handler in the chain
- `0`: Consume the keystroke (prevent further processing)

**Implementation Guidelines**:

1. **Check Virtual Keys First**: For special keys (arrows, function keys), check `inVirtualKey`
2. **Handle Modifiers**: Use bitwise AND to check modifier flags
3. **Be Selective**: Only consume keystrokes you actually handle
4. **Performance**: Keep processing fast - this is called for every keystroke

**Virtual Key Examples**:

```cpp
int MyKeySniffer::HandleKeyStroke(char inCharKey, XPLMKeyFlags inFlags, char inVirtualKey) {
    // Function keys
    if (inVirtualKey == XPLM_VK_F1) {
        if (inFlags & xplm_ControlFlag) {
            // Ctrl+F1
            return 0; // Consumed
        }
    }
  
    // Arrow keys
    if (inVirtualKey == XPLM_VK_LEFT && (inFlags & xplm_ShiftFlag)) {
        // Shift+Left Arrow
        return 0;
    }
  
    // Regular character keys
    if (inCharKey == 'q' && (inFlags & xplm_ControlFlag)) {
        // Ctrl+Q
        return 0;
    }
  
    return 1; // Pass through unhandled keys
}
```

### XPCKeySniffer Static Callback Bridge

#### static int KeySnifferCB(char inCharKey, XPLMKeyFlags inFlags, char inVirtualKey, void* inRefCon)

**Internal implementation detail** - static function that bridges C-style callbacks to C++ member functions.

**Behavior**:

1. Casts `inRefCon` back to `XPCKeySniffer*`
2. Calls the virtual `HandleKeyStroke()` method
3. Returns the result to X-Plane

**Note**: This is a private implementation detail - plugins should not call this directly.

### XPCKeySniffer Use Cases

#### 1. Global Plugin Hotkeys

```cpp
class PluginHotkeys : public XPCKeySniffer {
private:
    enum Actions {
        ACTION_SHOW_UI = 1,
        ACTION_TOGGLE_FEATURE = 2
    };
  
public:
    PluginHotkeys() : XPCKeySniffer(0) {}
  
    int HandleKeyStroke(char inCharKey, XPLMKeyFlags inFlags, char inVirtualKey) override {
        // Ctrl+Shift+P - Show main UI
        if (inCharKey == 'p' && (inFlags & (xplm_ControlFlag | xplm_ShiftFlag)) == (xplm_ControlFlag | xplm_ShiftFlag)) {
            PluginManager::ShowMainWindow();
            return 0;
        }
      
        // F12 - Toggle debug mode
        if (inVirtualKey == XPLM_VK_F12) {
            DebugManager::ToggleDebugMode();
            return 0;
        }
      
        return 1; // Pass through other keys
    }
};
```

#### 2. Text Input Library

```cpp
class TextInputSniffer : public XPCKeySniffer {
private:
    TextInputWidget* activeWidget;
  
public:
    TextInputSniffer() : XPCKeySniffer(0), activeWidget(nullptr) {}
  
    void SetActiveWidget(TextInputWidget* widget) { activeWidget = widget; }
  
    int HandleKeyStroke(char inCharKey, XPLMKeyFlags inFlags, char inVirtualKey) override {
        if (!activeWidget || !activeWidget->HasFocus()) {
            return 1; // No active text input
        }
      
        // Handle backspace
        if (inVirtualKey == XPLM_VK_BACK) {
            activeWidget->DeleteLastChar();
            return 0;
        }
      
        // Handle printable characters
        if (inCharKey >= 32 && inCharKey <= 126) { // Printable ASCII
            activeWidget->AddChar(inCharKey);
            return 0;
        }
      
        return 1; // Pass through other keys
    }
};
```

#### 3. Accessibility Support

```cpp
class AccessibilitySniffer : public XPCKeySniffer {
public:
    AccessibilitySniffer() : XPCKeySniffer(1) {} // Before windows for accessibility priority
  
    int HandleKeyStroke(char inCharKey, XPLMKeyFlags inFlags, char inVirtualKey) override {
        // Screen reader commands
        if (inFlags & xplm_ControlFlag) {
            switch (inVirtualKey) {
                case XPLM_VK_UP:
                    ScreenReader::ReadPreviousElement();
                    return 0;
                case XPLM_VK_DOWN:
                    ScreenReader::ReadNextElement();
                    return 0;
                case XPLM_VK_RETURN:
                    ScreenReader::ActivateCurrentElement();
                    return 0;
            }
        }
      
        return 1; // Pass through other keys
    }
};
```

---

## XPCWindow Class

The `XPCWindow` class provides a C++ interface for creating and managing windows in X-Plane. It encapsulates the legacy window system with automatic resource management and virtual method dispatch.

### XPCWindow Declaration

```cpp
class XPCWindow {
public:
    XPCWindow(int inLeft, int inTop, int inRight, int inBottom, int inIsVisible);
    virtual ~XPCWindow();

    // Pure virtual methods - must be implemented by derived classes
    virtual void DoDraw(void) = 0;
    virtual void HandleKey(char inKey, XPLMKeyFlags inFlags, char inVirtualKey) = 0;
    virtual void LoseFocus(void) = 0;
    virtual int HandleClick(int x, int y, XPLMMouseStatus inMouse) = 0;
  
    // Window management methods
    void GetWindowGeometry(int* outLeft, int* outTop, int* outRight, int* outBottom);
    void SetWindowGeometry(int inLeft, int inTop, int inRight, int inBottom);
    int GetWindowIsVisible(void);
    void SetWindowIsVisible(int inIsVisible);
    void TakeKeyboardFocus(void);
    void BringWindowToFront(void);
    int IsWindowInFront(void);

private:
    XPLMWindowID mWindow;
  
    // Static callback bridges
    static void DrawCB(XPLMWindowID inWindowID, void* inRefcon);
    static void HandleKeyCB(XPLMWindowID inWindowID, char inKey, XPLMKeyFlags inFlags, char inVirtualKey, void* inRefcon, int losingFocus);
    static int MouseClickCB(XPLMWindowID inWindowID, int x, int y, XPLMMouseStatus inMouse, void* inRefcon);
};
```

### XPCWindow Constructor

#### XPCWindow(int inLeft, int inTop, int inRight, int inBottom, int inIsVisible)

Creates a new window and automatically registers it with X-Plane's window system.

**Parameters**:

- `inLeft`, `inTop`, `inRight`, `inBottom`: Window bounds in pixels
  - Coordinates are in X-Plane's panel coordinate system
  - Origin (0,0) is at the bottom-left of the X-Plane window
  - `inLeft` < `inRight`, `inBottom` < `inTop`
- `inIsVisible`: Initial visibility (0 = hidden, 1 = visible)

**Behavior**:

1. Calls `XPLMCreateWindow()` to create the underlying XPLM window
2. Registers static callback functions for drawing, keyboard, and mouse events
3. Stores `this` pointer as the refcon for callback routing
4. Stores the returned `XPLMWindowID` in `mWindow`

**Important Notes**:

- Uses **legacy window system** (not modern XPLMCreateWindowEx)
- Coordinates are in pixels, not boxels (no high-DPI scaling)
- Window has no automatic decoration - you must draw your own frame
- No support for modern features like pop-out windows or window layers

**Example**:

```cpp
class MyPluginWindow : public XPCWindow {
public:
    MyPluginWindow() : XPCWindow(100, 400, 500, 200, 1) {
        // Creates a 400x200 pixel window at position (100, 200)
        // Window is initially visible
    }
  
    // Must implement pure virtual methods...
};
```

### XPCWindow Destructor

#### virtual ~XPCWindow()

Automatically destroys the window and cleans up resources.

**Behavior**:

1. Calls `XPLMDestroyWindow(mWindow)` to destroy the underlying window
2. Automatically removes keyboard focus if this window had it
3. Window callbacks are no longer called after destruction
4. Safe cleanup even in exception scenarios

**RAII Benefits**:

- Automatic resource management
- No manual cleanup required
- Exception-safe destruction

### XPCWindow Pure Virtual Methods

These methods **must** be implemented by derived classes to handle window events.

#### virtual void DoDraw(void) = 0

**Abstract method** for rendering the window contents.

**Context**:

- Called by X-Plane when the window needs to be redrawn
- OpenGL context is set up for 2D drawing
- Coordinate system has origin at bottom-left of the window
- Window bounds are clipped automatically

**Implementation Guidelines**:

1. **Draw Background**: Clear or fill the window background
2. **Draw Content**: Render text, graphics, UI elements
3. **Use XPLM Graphics API**: Use `XPLMDrawString()`, `XPLMDrawTranslucentDarkBox()`, etc.
4. **Handle Transparency**: Unfilled areas will be transparent
5. **Performance**: Keep drawing efficient - called every frame when visible

**Example**:

```cpp
void MyWindow::DoDraw(void) {
    int left, top, right, bottom;
    GetWindowGeometry(&left, &top, &right, &bottom);
  
    // Draw background
    XPLMDrawTranslucentDarkBox(left, top, right, bottom);
  
    // Draw title
    XPLMDrawString(RGB(1.0f, 1.0f, 1.0f), left + 10, top - 20, 
                   "My Plugin Window", NULL, xplmFont_Proportional);
  
    // Draw content based on window state
    if (isDataAvailable) {
        char buffer[256];
        snprintf(buffer, sizeof(buffer), "Value: %.2f", currentValue);
        XPLMDrawString(RGB(0.8f, 1.0f, 0.8f), left + 10, top - 40,
                       buffer, NULL, xplmFont_Proportional);
    }
}
```

#### virtual void HandleKey(char inKey, XPLMKeyFlags inFlags, char inVirtualKey) = 0

**Abstract method** for handling keyboard input when the window has focus.

**Parameters**:

- `inKey`: ASCII character code (0 for non-printable keys)
- `inFlags`: Modifier key flags (see `XPLMKeyFlags`)
- `inVirtualKey`: Virtual key code for special keys

**Context**:

- Only called when this window has keyboard focus
- Use `TakeKeyboardFocus()` to request focus
- Window can lose focus to other windows or X-Plane itself

**Implementation Guidelines**:

1. **Handle Text Input**: Process printable characters for text entry
2. **Handle Special Keys**: Process arrows, function keys, etc.
3. **Check Modifiers**: Handle Ctrl+, Shift+, Alt+ combinations
4. **Provide Feedback**: Update UI state based on input
5. **Navigation**: Handle Tab, Enter for UI navigation

**Example**:

```cpp
void MyWindow::HandleKey(char inKey, XPLMKeyFlags inFlags, char inVirtualKey) {
    // Handle text input
    if (inKey >= 32 && inKey <= 126) { // Printable ASCII
        if (textInputActive) {
            inputBuffer += inKey;
            return;
        }
    }
  
    // Handle special keys
    switch (inVirtualKey) {
        case XPLM_VK_RETURN:
            if (textInputActive) {
                ProcessInput(inputBuffer);
                inputBuffer.clear();
            }
            break;
          
        case XPLM_VK_BACK:
            if (textInputActive && !inputBuffer.empty()) {
                inputBuffer.pop_back();
            }
            break;
          
        case XPLM_VK_ESCAPE:
            CancelCurrentOperation();
            break;
          
        case XPLM_VK_TAB:
            FocusNextControl();
            break;
    }
}
```

#### virtual void LoseFocus(void) = 0

**Abstract method** called when the window loses keyboard focus.

**Context**:

- Called when another window takes keyboard focus
- Called when user clicks outside the window
- Called when X-Plane itself takes focus
- Called before the window is destroyed

**Implementation Guidelines**:

1. **Save State**: Commit any pending text input or changes
2. **Visual Feedback**: Update UI to show unfocused state
3. **Cleanup**: Cancel any ongoing input operations
4. **No Exceptions**: Must not throw exceptions

**Example**:

```cpp
void MyWindow::LoseFocus(void) {
    // Commit any pending input
    if (textInputActive && !inputBuffer.empty()) {
        ProcessInput(inputBuffer);
        inputBuffer.clear();
    }
  
    // Update visual state
    hasFocus = false;
    cursorBlinkTimer = 0;
  
    // Cancel any modal operations
    if (isInModalDialog) {
        CancelModalDialog();
    }
}
```

#### virtual int HandleClick(int x, int y, XPLMMouseStatus inMouse) = 0

**Abstract method** for handling mouse clicks within the window.

**Parameters**:

- `x`, `y`: Mouse coordinates in panel coordinate system
  - Origin is at bottom-left of the X-Plane window
  - Coordinates are in pixels
- `inMouse`: Mouse button and action (see `XPLMMouseStatus`)
  - `xplm_MouseDown`: Button pressed
  - `xplm_MouseDrag`: Mouse moved while button held
  - `xplm_MouseUp`: Button released

**Return Value**:

- `1`: Event handled by this window
- `0`: Event not handled (rare - usually return 1)

**Implementation Guidelines**:

1. **Convert Coordinates**: Convert from global to window-local coordinates
2. **Hit Testing**: Determine which UI element was clicked
3. **Handle Button States**: Track mouse down/drag/up sequences
4. **Take Focus**: Call `TakeKeyboardFocus()` if needed
5. **Visual Feedback**: Update UI to show interaction state

**Example**:

```cpp
int MyWindow::HandleClick(int x, int y, XPLMMouseStatus inMouse) {
    int left, top, right, bottom;
    GetWindowGeometry(&left, &top, &right, &bottom);
  
    // Convert to window-local coordinates
    int localX = x - left;
    int localY = y - bottom;
  
    // Handle close button (top-right corner)
    if (localX >= (right - left - 20) && localY >= (top - bottom - 20)) {
        if (inMouse == xplm_MouseDown) {
            closeButtonPressed = true;
        } else if (inMouse == xplm_MouseUp && closeButtonPressed) {
            SetWindowIsVisible(0);
            closeButtonPressed = false;
        }
        return 1;
    }
  
    // Handle text input area
    if (IsInTextArea(localX, localY)) {
        if (inMouse == xplm_MouseDown) {
            TakeKeyboardFocus();
            textInputActive = true;
            SetCursorPosition(localX, localY);
        }
        return 1;
    }
  
    return 1;
}
```

### XPCWindow Management Methods

These methods provide direct access to the underlying window properties and state.

#### void GetWindowGeometry(int\* outLeft, int\* outTop, int\* outRight, int\* outBottom)

Retrieves the current window bounds.

**Parameters**:

- `outLeft`, `outTop`, `outRight`, `outBottom`: Pointers to receive coordinate values

**Usage**:

```cpp
int left, top, right, bottom;
GetWindowGeometry(&left, &top, &right, &bottom);

int width = right - left;
int height = top - bottom;
```

#### void SetWindowGeometry(int inLeft, int inTop, int inRight, int inBottom)

Changes the window position and size.

**Parameters**:

- Window bounds in panel coordinate system
- Changes take effect immediately
- Window is automatically redrawn

**Usage**:

```cpp
// Resize window to 300x200 at position (150, 250)
SetWindowGeometry(150, 450, 450, 250);
```

#### int GetWindowIsVisible(void)

Returns the current visibility state.

**Return Value**:

- `1`: Window is visible
- `0`: Window is hidden

#### void SetWindowIsVisible(int inIsVisible)

Shows or hides the window.

**Parameters**:

- `inIsVisible`: 1 to show, 0 to hide

**Effects**:

- Hidden windows don't receive draw callbacks
- Hidden windows don't receive mouse clicks
- Hidden windows can still have keyboard focus

#### void TakeKeyboardFocus(void)

Requests keyboard focus for this window.

**Effects**:

- This window will receive all keyboard input
- Other windows lose focus (their `LoseFocus()` is called)
- User typing will be sent to this window's `HandleKey()` method

**Usage**:

```cpp
// When user clicks in a text field
void MyWindow::OnTextFieldClick() {
    TakeKeyboardFocus();
    textInputActive = true;
    // Now HandleKey() will receive keyboard input
}
```

#### void BringWindowToFront(void)

Brings the window to the front of the Z-order.

**Effects**:

- Window draws on top of other windows at the same layer
- Window receives mouse clicks before windows behind it
- Only affects windows in the same layer (legacy windows are all in one layer)

#### int IsWindowInFront(void)

Checks if this window is at the front of its layer.

**Return Value**:

- `1`: Window is frontmost in its layer
- `0`: Other windows are in front

### XPCWindow Static Callback Bridges

These are internal implementation details that bridge C-style callbacks to C++ member functions.

#### static void DrawCB(XPLMWindowID inWindowID, void* inRefcon)

**Internal method** - bridges X-Plane draw callbacks to the virtual `DoDraw()` method.

#### static void HandleKeyCB(XPLMWindowID inWindowID, char inKey, XPLMKeyFlags inFlags, char inVirtualKey, void* inRefcon, int losingFocus)

**Internal method** - bridges X-Plane keyboard callbacks to virtual methods:

- Calls `LoseFocus()` if `losingFocus` is true
- Calls `HandleKey()` if `losingFocus` is false

#### static int MouseClickCB(XPLMWindowID inWindowID, int x, int y, XPLMMouseStatus inMouse, void* inRefcon)

**Internal method** - bridges X-Plane mouse callbacks to the virtual `HandleClick()` method.

### XPCWindow Complete Implementation Example

```cpp
class StatusWindow : public XPCWindow {
private:
    std::string title;
    std::vector<std::string> statusLines;
    bool hasFocus;
    int selectedLine;
  
public:
    StatusWindow() : XPCWindow(50, 300, 350, 100, 1), hasFocus(false), selectedLine(0) {
        title = "Flight Status";
        statusLines.push_back("Altitude: 5000 ft");
        statusLines.push_back("Speed: 120 kts");
        statusLines.push_back("Heading: 090°");
    }
  
    void DoDraw(void) override {
        int left, top, right, bottom;
        GetWindowGeometry(&left, &top, &right, &bottom);
      
        // Background
        XPLMDrawTranslucentDarkBox(left, top, right, bottom);
      
        // Title
        float titleColor[] = {1.0f, 1.0f, 1.0f};
        XPLMDrawString(titleColor, left + 5, top - 15, title.c_str(), NULL, xplmFont_Proportional);
      
        // Status lines
        for (size_t i = 0; i < statusLines.size(); ++i) {
            float color[] = {0.8f, 0.8f, 0.8f};
            if (hasFocus && i == selectedLine) {
                color[0] = color[1] = 1.0f; // Highlight selected line
            }
          
            XPLMDrawString(color, left + 10, top - 35 - (i * 15), 
                          statusLines[i].c_str(), NULL, xplmFont_Proportional);
        }
    }
  
    void HandleKey(char inKey, XPLMKeyFlags inFlags, char inVirtualKey) override {
        switch (inVirtualKey) {
            case XPLM_VK_UP:
                if (selectedLine > 0) selectedLine--;
                break;
            case XPLM_VK_DOWN:
                if (selectedLine < statusLines.size() - 1) selectedLine++;
                break;
            case XPLM_VK_RETURN:
                // Do something with selected line
                OnLineSelected(selectedLine);
                break;
        }
    }
  
    void LoseFocus(void) override {
        hasFocus = false;
    }
  
    int HandleClick(int x, int y, XPLMMouseStatus inMouse) override {
        if (inMouse == xplm_MouseDown) {
            TakeKeyboardFocus();
            hasFocus = true;
          
            // Calculate which line was clicked
            int left, top, right, bottom;
            GetWindowGeometry(&left, &top, &right, &bottom);
          
            int localY = y - bottom;
            int lineHeight = 15;
            int firstLineY = (top - bottom) - 35;
          
            if (localY < firstLineY) {
                int clickedLine = (firstLineY - localY) / lineHeight;
                if (clickedLine < statusLines.size()) {
                    selectedLine = clickedLine;
                }
            }
        }
        return 1;
    }
  
    // Custom methods
    void UpdateStatus(const std::string& line, int index) {
        if (index < statusLines.size()) {
            statusLines[index] = line;
        }
    }
  
private:
    void OnLineSelected(int line) {
        // Handle line selection - could open detail window, etc.
    }
};
```

### XPCWindow Common Use Cases

#### 1. Simple Information Display

```cpp
class InfoPanel : public XPCWindow {
private:
    struct InfoItem {
        std::string label;
        std::string value;
        float color[3];
    };
    std::vector<InfoItem> items;
  
public:
    InfoPanel() : XPCWindow(10, 200, 200, 50, 1) {}
  
    void AddInfo(const std::string& label, const std::string& value, bool highlight = false) {
        InfoItem item;
        item.label = label;
        item.value = value;
        item.color[0] = highlight ? 1.0f : 0.8f;
        item.color[1] = highlight ? 1.0f : 0.8f;
        item.color[2] = highlight ? 0.8f : 0.8f;
        items.push_back(item);
    }
  
    void DoDraw(void) override {
        int left, top, right, bottom;
        GetWindowGeometry(&left, &top, &right, &bottom);
      
        XPLMDrawTranslucentDarkBox(left, top, right, bottom);
      
        int yPos = top - 15;
        for (const auto& item : items) {
            std::string line = item.label + ": " + item.value;
            XPLMDrawString((float*)item.color, left + 5, yPos, line.c_str(), NULL, xplmFont_Proportional);
            yPos -= 15;
        }
    }
  
    // Minimal implementations for required virtuals
    void HandleKey(char, XPLMKeyFlags, char) override {}
    void LoseFocus(void) override {}
    int HandleClick(int, int, XPLMMouseStatus) override { return 1; }
};
```

#### 2. Interactive Control Panel

```cpp
class ControlPanel : public XPCWindow {
private:
    struct Button {
        int x, y, width, height;
        std::string label;
        std::function<void()> action;
        bool pressed;
    };
    std::vector<Button> buttons;
  
public:
    ControlPanel() : XPCWindow(300, 250, 500, 100, 1) {
        AddButton(10, 10, 80, 25, "Start", []() { /* Start action */ });
        AddButton(100, 10, 80, 25, "Stop", []() { /* Stop action */ });
    }
  
    void AddButton(int x, int y, int w, int h, const std::string& label, std::function<void()> action) {
        Button btn = {x, y, w, h, label, action, false};
        buttons.push_back(btn);
    }
  
    void DoDraw(void) override {
        int left, top, right, bottom;
        GetWindowGeometry(&left, &top, &right, &bottom);
      
        XPLMDrawTranslucentDarkBox(left, top, right, bottom);
      
        for (const auto& btn : buttons) {
            // Draw button background
            float btnColor[] = {0.5f, 0.5f, 0.5f};
            if (btn.pressed) {
                btnColor[0] = btnColor[1] = btnColor[2] = 0.3f;
            }
          
            // Draw button text
            XPLMDrawString(RGB(1,1,1), left + btn.x + 5, top - btn.y - 15,
                          btn.label.c_str(), NULL, xplmFont_Proportional);
        }
    }
  
    int HandleClick(int x, int y, XPLMMouseStatus inMouse) override {
        int left, top, right, bottom;
        GetWindowGeometry(&left, &top, &right, &bottom);
      
        int localX = x - left;
        int localY = (top - y); // Convert to window coords
      
        for (auto& btn : buttons) {
            if (localX >= btn.x && localX <= btn.x + btn.width &&
                localY >= btn.y && localY <= btn.y + btn.height) {
              
                if (inMouse == xplm_MouseDown) {
                    btn.pressed = true;
                } else if (inMouse == xplm_MouseUp && btn.pressed) {
                    btn.pressed = false;
                    btn.action(); // Execute button action
                }
                return 1;
            }
        }
      
        return 1;
    }
  
    void HandleKey(char, XPLMKeyFlags, char) override {}
    void LoseFocus(void) override {
        // Reset all button states
        for (auto& btn : buttons) {
            btn.pressed = false;
        }
    }
};
```

---

## Implementation Guidelines

### Best Practices

#### 1. Resource Management

```cpp
class MyPlugin {
private:
    std::unique_ptr<MyWindow> mainWindow;
    std::unique_ptr<MyKeySniffer> keySniffer;
  
public:
    void Initialize() {
        // Create wrappers - they automatically register with X-Plane
        mainWindow = std::make_unique<MyWindow>();
        keySniffer = std::make_unique<MyKeySniffer>();
    }
  
    void Shutdown() {
        // Destroy wrappers - they automatically unregister
        mainWindow.reset();
        keySniffer.reset();
    }
};
```

#### 2. Exception Safety

```cpp
class SafeWindow : public XPCWindow {
public:
    void DoDraw(void) override {
        try {
            // Drawing code that might throw
            RenderContent();
        } catch (const std::exception& e) {
            // Log error but don't propagate to X-Plane
            XPLMDebugString("Drawing error: ");
            XPLMDebugString(e.what());
            XPLMDebugString("\n");
        }
    }
  
private:
    void RenderContent() {
        // Actual drawing implementation
    }
};
```

#### 3. Performance Optimization

```cpp
class OptimizedWindow : public XPCWindow {
private:
    bool needsRedraw;
    mutable int cachedLeft, cachedTop, cachedRight, cachedBottom;
  
public:
    void DoDraw(void) override {
        if (!needsRedraw) return;
      
        // Cache geometry to avoid repeated API calls
        GetWindowGeometry(&cachedLeft, &cachedTop, &cachedRight, &cachedBottom);
      
        // Fast drawing path
        DrawCachedContent();
        needsRedraw = false;
    }
  
    void MarkDirty() { needsRedraw = true; }
};
```

### Integration with X-Plane Plugin Lifecycle

#### Plugin Startup

```cpp
PLUGIN_API int XPluginStart(char* outName, char* outSig, char* outDesc) {
    strcpy(outName, "XPCDisplay Example");
    strcpy(outSig, "com.example.xpcdisplay");
    strcpy(outDesc, "Example plugin using XPCDisplay wrappers");
  
    // Don't create wrappers here - wait for Enable
    return 1;
}

PLUGIN_API int XPluginEnable(void) {
    try {
        // Create display wrappers during enable
        gMainWindow = std::make_unique<MainWindow>();
        gHotkeySniffer = std::make_unique<HotkeySniffer>();
        return 1;
    } catch (const std::exception& e) {
        XPLMDebugString("Failed to enable plugin: ");
        XPLMDebugString(e.what());
        return 0;
    }
}

PLUGIN_API void XPluginDisable(void) {
    // Destroy wrappers during disable - automatic cleanup
    gMainWindow.reset();
    gHotkeySniffer.reset();
}

PLUGIN_API void XPluginStop(void) {
    // Nothing to do - wrappers already cleaned up
}
```

### Migration from Raw XPLM APIs

#### Before (Raw XPLM)

```cpp
static XPLMWindowID gWindow = NULL;

void DrawCallback(XPLMWindowID inWindowID, void* inRefcon) {
    // Drawing code
}

int KeyCallback(XPLMWindowID inWindowID, char inKey, XPLMKeyFlags inFlags, 
                char inVirtualKey, void* inRefcon, int losingFocus) {
    // Key handling
    return 1;
}

int MouseCallback(XPLMWindowID inWindowID, int x, int y, 
                  XPLMMouseStatus inMouse, void* inRefcon) {
    // Mouse handling
    return 1;
}

void CreateWindow() {
    gWindow = XPLMCreateWindow(100, 300, 400, 200, 1,
                               DrawCallback, KeyCallback, MouseCallback, NULL);
}

void DestroyWindow() {
    if (gWindow) {
        XPLMDestroyWindow(gWindow);
        gWindow = NULL;
    }
}
```

#### After (XPCDisplay Wrapper)

```cpp
class MyWindow : public XPCWindow {
public:
    MyWindow() : XPCWindow(100, 300, 400, 200, 1) {}
  
    void DoDraw(void) override {
        // Same drawing code, but as a member function
    }
  
    void HandleKey(char inKey, XPLMKeyFlags inFlags, char inVirtualKey) override {
        // Key handling as member function
    }
  
    void LoseFocus(void) override {
        // Focus handling
    }
  
    int HandleClick(int x, int y, XPLMMouseStatus inMouse) override {
        // Mouse handling as member function
        return 1;
    }
};

// Usage
std::unique_ptr<MyWindow> gWindow;

void CreateWindow() {
    gWindow = std::make_unique<MyWindow>(); // Automatic creation and registration
}

void DestroyWindow() {
    gWindow.reset(); // Automatic cleanup
}
```

### Thread Safety Considerations

The XPCDisplay wrappers inherit the thread safety characteristics of the underlying XPLM API:

#### Single-Thread Design

- All callbacks occur on X-Plane's main thread
- No synchronization primitives needed within callback methods
- Safe to access shared plugin state from callbacks

#### Multi-Threading Guidelines

```cpp
class ThreadSafeWindow : public XPCWindow {
private:
    std::mutex dataMutex;
    std::string sharedData;
  
public:
    void DoDraw(void) override {
        std::string data;
        {
            std::lock_guard<std::mutex> lock(dataMutex);
            data = sharedData; // Copy while locked
        }
      
        // Use local copy for drawing (no lock needed)
        DrawData(data);
    }
  
    // Thread-safe method for background threads
    void UpdateData(const std::string& newData) {
        std::lock_guard<std::mutex> lock(dataMutex);
        sharedData = newData;
    }
};
```

---

## Limitations and Considerations

### Legacy Window System

The `XPCWindow` class wraps the **legacy** X-Plane window system, which has several limitations compared to modern windows:

#### Missing Modern Features

- **No High-DPI Support**: Uses pixels instead of boxels
- **No Pop-out Windows**: Cannot be moved to separate OS windows
- **No Window Layers**: All legacy windows are in the same layer
- **No Modern Styling**: Must draw your own window decorations
- **Limited Positioning**: Uses panel coordinates, not global desktop coordinates

#### Migration Path

For new plugins, consider using the modern XPLM API directly:

```cpp
// Instead of XPCWindow, use modern XPLM API
XPLMCreateWindow_t params = {};
params.structSize = sizeof(params);
params.left = 100; params.top = 400; params.right = 500; params.bottom = 200;
params.visible = 1;
params.drawWindowFunc = DrawCallback;
params.handleKeyFunc = KeyCallback;
params.handleMouseClickFunc = MouseCallback;
params.layer = xplm_WindowLayerFloatingWindows;
params.decorateAsFloatingWindow = xplm_WindowDecorationRoundRectangle;

XPLMWindowID window = XPLMCreateWindowEx(&params);
```

### Performance Considerations

#### Drawing Performance

- Drawing callbacks are called every frame when visible
- Keep `DoDraw()` implementations fast and efficient
- Cache expensive calculations outside of drawing
- Use X-Plane's built-in drawing functions when possible

#### Memory Management

- Window objects should have stable lifetime (avoid frequent create/destroy)
- Large windows with complex content may impact framerate
- Consider using visibility toggling instead of window destruction

### Virtual Key Handling

#### Platform Differences

Virtual key codes may vary between platforms:

```cpp
void HandleKey(char inKey, XPLMKeyFlags inFlags, char inVirtualKey) override {
    // Be careful with virtual key comparisons
    if (inVirtualKey == XPLM_VK_RETURN) {
        // This works across platforms
    }
  
    // Avoid hardcoded key codes
    // if (inVirtualKey == 13) { // Don't do this
}
```

#### Character Encoding

- Character input may depend on user's keyboard layout
- Non-ASCII characters may not be handled consistently
- Consider using commands instead of direct keyboard handling for important features

---

## Conclusion

The XPCDisplay wrapper classes provide C++ X-Plane plugin UI development, offering:

- **RAII Resource Management**: Automatic cleanup and exception safety
- **Object-Oriented Design**: Clean separation between different windows and input handlers
- **Type Safety**: Strong typing over raw XPLM handles
- **Event-Driven Architecture**: Virtual methods for handling user interaction

While the wrappers use the legacy window system, they remain valuable for:

- **Simple Information Displays**: Status windows, debug panels
- **Basic User Interaction**: Button panels, simple dialogs
- **Rapid Prototyping**: Quick UI development and testing
- **Learning**: Understanding X-Plane UI concepts before moving to modern APIs

For production plugins requiring modern features like high-DPI support, pop-out windows, or advanced styling, consider using the modern `XPLMCreateWindowEx()` API directly while applying the same RAII and object-oriented principles demonstrated by these wrapper classes.
