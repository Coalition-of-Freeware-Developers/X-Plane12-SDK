# XPLMGraphics.h - X-Plane 12 SDK Graphics API Documentation

## Overview

The XPLMGraphics API provides OpenGL integration capabilities for X-Plane 12 plugins, enabling coordinate system conversions, texture management, and basic drawing operations. This API serves as the foundation for 3D rendering and 2D UI drawing within the X-Plane environment.

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Coordinate Systems](#coordinate-systems)
- [Core Data Types](#core-data-types)
- [OpenGL State Management](#opengl-state-management)
- [Texture Management](#texture-management)
- [Coordinate Conversion](#coordinate-conversion)
- [Drawing Utilities](#drawing-utilities)
- [Font and Text Rendering](#font-and-text-rendering)
- [Implementation Guidelines](#implementation-guidelines)
- [Best Practices](#best-practices)
- [Common Use Cases](#common-use-cases)
- [Performance Considerations](#performance-considerations)
- [Deprecation Notes](#deprecation-notes)

## Architecture Overview

The XPLMGraphics API operates within X-Plane's OpenGL rendering context and provides:

- **Coordinate System Integration**: Conversions between X-Plane's different coordinate systems
- **OpenGL State Management**: Controlled access to OpenGL state changes
- **Texture Utilities**: Generation and binding of OpenGL textures
- **Font Rendering**: Text drawing capabilities with X-Plane's built-in fonts
- **Drawing Helpers**: Utility functions for common drawing operations

The API is designed to work seamlessly with X-Plane's rendering pipeline while maintaining compatibility across different graphics backends (OpenGL, Vulkan, Metal).

## Coordinate Systems

X-Plane uses three distinct coordinate systems, each optimized for different use cases:

### Global Coordinates

**Format**: Latitude, longitude, and elevation
**Precision**: Lower precision, stable across long distances
**Usage**: World positioning, navigation waypoints
**Units**: Degrees (lat/lon), meters MSL (elevation)

### Local/OpenGL Coordinates

**Format**: Cartesian X, Y, Z coordinates
**Precision**: High precision, suitable for 3D rendering
**Usage**: 3D OpenGL drawing, object positioning
**Origin**: Surface of Earth at sea level near the aircraft
**Axes**:

- X-axis: East-West (positive = east)
- Y-axis: Up-Down (positive = up)
- Z-axis: North-South (positive = south)

**Units**: Meters

### 2D Panel Coordinates

**Format**: 2D X, Y coordinates
**Usage**: Cockpit panels, 2D UI elements
**Origin**: Bottom-left corner (0, 0)
**Bounds**: (1024, 768) represents upper-right
**Scaling**: Automatically scaled for different resolutions

## Core Data Types

### XPLMTextureID

```c
enum {
    xplm_Tex_GeneralInterface = 0,    // UI bitmap textures
    // Deprecated texture IDs (do not use)
};
typedef int XPLMTextureID;
```

**Important**: Most texture IDs are deprecated. Use the Widgets library to access legacy UI textures if needed.

### XPLMFontID

```c
enum {
    xplmFont_Basic = 0,                // Mono-spaced UI font (recommended)
    xplmFont_Proportional = 18,        // Proportional UI font (XPLM200+)
    // Many deprecated font IDs (marked as deprecated)
};
typedef int XPLMFontID;
```

**Recommendation**: Use `xplmFont_Basic` for maximum compatibility, or `xplmFont_Proportional` for modern UI.

## OpenGL State Management

### XPLMSetGraphicsState

```c
XPLM_API void XPLMSetGraphicsState(
    int inEnableFog,              // Enable/disable fog
    int inNumberTexUnits,         // Number of texture units (0-4)
    int inEnableLighting,         // Enable/disable lighting
    int inEnableAlphaTesting,     // Enable/disable alpha testing
    int inEnableAlphaBlending,    // Enable/disable alpha blending
    int inEnableDepthTesting,     // Enable/disable depth testing
    int inEnableDepthWriting      // Enable/disable depth buffer writes
);
```

**Purpose**: Safely modifies OpenGL state while keeping X-Plane synchronized.

**Key Benefits**:

- Prevents state conflicts between plugins and X-Plane
- Optimizes state changes by caching current state
- Ensures consistent rendering behavior

**Important Notes**:

- Always call this function instead of direct OpenGL state calls
- State is not automatically restored - plugins must set desired state
- Lighting and fog are deprecated for modern usage

**Example Usage**:

```c
// Set up for 2D drawing with transparency
XPLMSetGraphicsState(
    0,    // No fog
    1,    // One texture unit
    0,    // No lighting
    1,    // Enable alpha testing
    1,    // Enable alpha blending
    0,    // No depth testing
    0     // No depth writing
);

// Draw your 2D elements here
// ...
```

**Modern Best Practices**:

```c
// For modern 3D drawing (no fog/lighting)
XPLMSetGraphicsState(0, 1, 0, 1, 1, 1, 1);

// For 2D UI drawing
XPLMSetGraphicsState(0, 1, 0, 1, 1, 0, 0);

// For simple geometry without textures
XPLMSetGraphicsState(0, 0, 0, 0, 1, 1, 1);
```

## Texture Management

### XPLMBindTexture2d

```c
XPLM_API void XPLMBindTexture2d(
    int inTextureNum,     // OpenGL texture object ID
    int inTextureUnit     // Texture unit (0-3, may expand)
);
```

**Purpose**: Efficiently binds 2D textures with caching to prevent redundant OpenGL calls.

**Parameters**:

- `inTextureNum`: OpenGL texture object ID (from glGenTextures)
- `inTextureUnit`: Zero-based texture unit index

**Usage**:

```c
GLuint myTexture;
glGenTextures(1, &myTexture);
glBindTexture(GL_TEXTURE_2D, myTexture);
// ... load texture data ...

// Later, for drawing:
XPLMSetGraphicsState(0, 1, 0, 1, 1, 1, 1);
XPLMBindTexture2d(myTexture, 0);  // Bind to unit 0
// ... draw textured geometry ...
```

### XPLMGenerateTextureNumbers

```c
XPLM_API void XPLMGenerateTextureNumbers(
    int *outTextureIDs,   // Array to receive texture IDs
    int inCount           // Number of texture IDs to generate
);
```

**Purpose**: Generates OpenGL texture object IDs safely.

**Modern Usage**: This function now simply calls `glGenTextures` but maintains API consistency.

**Example**:

```c
GLuint textures[4];
XPLMGenerateTextureNumbers((int*)textures, 4);

// Use the generated IDs
for (int i = 0; i < 4; i++) {
    glBindTexture(GL_TEXTURE_2D, textures[i]);
    // Load texture data...
}
```

## Coordinate Conversion

### XPLMWorldToLocal

```c
XPLM_API void XPLMWorldToLocal(
    double inLatitude,     // Latitude in decimal degrees
    double inLongitude,    // Longitude in decimal degrees  
    double inAltitude,     // Altitude in meters MSL
    double *outX,          // Local X coordinate (meters)
    double *outY,          // Local Y coordinate (meters)
    double *outZ           // Local Z coordinate (meters)
);
```

**Purpose**: Converts global coordinates to local OpenGL coordinates.

**Use Cases**:

- Positioning 3D objects at specific geographic locations
- Converting navigation waypoints for rendering
- Placing scenery objects

**Example**:

```c
// Position object at specific airport
double lat = 37.6213; // San Francisco International
double lon = -122.3790;
double alt = 10.0; // 10 meters above sea level

double x, y, z;
XPLMWorldToLocal(lat, lon, alt, &x, &y, &z);

// Now use x, y, z for OpenGL positioning
glTranslatef((float)x, (float)y, (float)z);
```

### XPLMLocalToWorld

```c
XPLM_API void XPLMLocalToWorld(
    double inX,            // Local X coordinate (meters)
    double inY,            // Local Y coordinate (meters)
    double inZ,            // Local Z coordinate (meters)
    double *outLatitude,   // Latitude in decimal degrees
    double *outLongitude,  // Longitude in decimal degrees
    double *outAltitude    // Altitude in meters MSL
);
```

**Purpose**: Converts local OpenGL coordinates back to global coordinates.

**Important Note**: Global coordinates have lower precision than local coordinates. Avoid round-trip conversions when possible.

**Use Cases**:

- Converting rendered object positions back to geographic coordinates
- Calculating distances and bearings
- Integration with navigation systems

**Example**:

```c
// Get aircraft position in world coordinates
double acftX = 1000.0, acftY = 500.0, acftZ = -2000.0;
double lat, lon, alt;

XPLMLocalToWorld(acftX, acftY, acftZ, &lat, &lon, &alt);

char msg[256];
sprintf(msg, "Aircraft at: %.6f, %.6f, %.1fm\n", lat, lon, alt);
XPLMDebugString(msg);
```

## Drawing Utilities

### XPLMDrawTranslucentDarkBox

```c
XPLM_API void XPLMDrawTranslucentDarkBox(
    int inLeft,     // Left edge in pixels
    int inTop,      // Top edge in pixels
    int inRight,    // Right edge in pixels
    int inBottom    // Bottom edge in pixels
);
```

**Purpose**: Draws a semi-transparent dark background box, commonly used behind text for readability.

**Coordinate System**: Uses current OpenGL coordinate system (typically screen pixels).

**Usage**:

```c
// Set up for 2D drawing
XPLMSetGraphicsState(0, 0, 0, 1, 1, 0, 0);

// Draw background box
XPLMDrawTranslucentDarkBox(100, 400, 400, 300);

// Draw text over the box
float white[] = {1.0f, 1.0f, 1.0f};
XPLMDrawString(white, 110, 380, "Information Panel", NULL, xplmFont_Basic);
```

## Font and Text Rendering

### XPLMDrawString

```c
XPLM_API void XPLMDrawString(
    float *inColorRGB,        // Color array [R, G, B] (0.0-1.0)
    int inXOffset,            // X position (pixels)
    int inYOffset,            // Y position (pixels)  
    const char *inChar,       // Null-terminated string
    int *inWordWrapWidth,     // Word wrap width (or NULL)
    XPLMFontID inFontID       // Font to use
);
```

**Purpose**: Renders text strings using X-Plane's built-in fonts.

**Parameters**:

- `inColorRGB`: RGB color values from 0.0 to 1.0
- `inXOffset`, `inYOffset`: Bottom-left corner of text
- `inChar`: Text to display
- `inWordWrapWidth`: Optional word wrapping (pass NULL to disable)
- `inFontID`: Font identifier

**Example**:

```c
// Draw white text
float white[] = {1.0f, 1.0f, 1.0f};
XPLMDrawString(white, 50, 100, "Hello World!", NULL, xplmFont_Basic);

// Draw colored text with word wrap
float green[] = {0.0f, 1.0f, 0.0f};
int wrapWidth = 200;
XPLMDrawString(green, 50, 200, "This is a long string that will wrap", 
               &wrapWidth, xplmFont_Proportional);
```

### XPLMDrawNumber

```c
XPLM_API void XPLMDrawNumber(
    float *inColorRGB,    // Color array [R, G, B]
    int inXOffset,        // X position
    int inYOffset,        // Y position
    double inValue,       // Numeric value to display
    int inDigits,         // Total digits to show
    int inDecimals,       // Decimal places
    int inShowSign,       // Show +/- sign (boolean)
    XPLMFontID inFontID   // Font to use
);
```

**Purpose**: Draws formatted numbers similar to X-Plane's built-in displays.

**Example**:

```c
// Draw altitude: "  5280" (5 digits, no decimals, no sign)
float white[] = {1.0f, 1.0f, 1.0f};
XPLMDrawNumber(white, 100, 150, 5280.0, 5, 0, 0, xplmFont_Basic);

// Draw frequency: "119.50" (5 digits, 2 decimals, no sign)
XPLMDrawNumber(white, 100, 130, 119.5, 5, 2, 0, xplmFont_Basic);

// Draw vertical speed: " +750" (4 digits, 0 decimals, show sign)
XPLMDrawNumber(white, 100, 110, 750.0, 4, 0, 1, xplmFont_Basic);
```

### XPLMGetFontDimensions

```c
XPLM_API void XPLMGetFontDimensions(
    XPLMFontID inFontID,      // Font to query
    int *outCharWidth,        // Character width (or NULL)
    int *outCharHeight,       // Character height (or NULL)
    int *outDigitsOnly        // 1 if digits-only font (or NULL)
);
```

**Purpose**: Gets font metrics for layout calculations.

**Example**:

```c
int charWidth, charHeight, digitsOnly;
XPLMGetFontDimensions(xplmFont_Basic, &charWidth, &charHeight, &digitsOnly);

// Calculate text box size
int textLength = strlen("My Text");
int boxWidth = textLength * charWidth;
int boxHeight = charHeight;
```

### XPLMMeasureString (XPLM200+)

```c
XPLM_API float XPLMMeasureString(
    XPLMFontID inFontID,     // Font to use
    const char *inChar,      // Text to measure
    int inNumChars           // Number of characters (-1 for null-terminated)
);
```

**Purpose**: Precisely measures string width for proportional fonts.

**Example**:

```c
const char* text = "Variable Width Text";
float width = XPLMMeasureString(xplmFont_Proportional, text, -1);

// Center text in a 400-pixel wide area
int xPos = 200 - (int)(width / 2.0f);
float white[] = {1.0f, 1.0f, 1.0f};
XPLMDrawString(white, xPos, 100, text, NULL, xplmFont_Proportional);
```

## Implementation Guidelines

### Plugin Integration Pattern

```c
// Graphics resources structure
typedef struct {
    GLuint backgroundTexture;
    GLuint iconTexture;
    int fontCharWidth;
    int fontCharHeight;
} GraphicsResources_t;

static GraphicsResources_t gGraphics = {0};

// Initialize graphics resources
int InitializeGraphics() {
    // Generate textures
    XPLMGenerateTextureNumbers((int*)&gGraphics.backgroundTexture, 1);
    XPLMGenerateTextureNumbers((int*)&gGraphics.iconTexture, 1);
  
    // Load texture data (implementation specific)
    LoadTextureFromFile("background.png", gGraphics.backgroundTexture);
    LoadTextureFromFile("icon.png", gGraphics.iconTexture);
  
    // Get font dimensions
    XPLMGetFontDimensions(xplmFont_Basic, 
                         &gGraphics.fontCharWidth, 
                         &gGraphics.fontCharHeight, 
                         NULL);
  
    return 1;
}

// Cleanup graphics resources
void CleanupGraphics() {
    if (gGraphics.backgroundTexture != 0) {
        glDeleteTextures(1, &gGraphics.backgroundTexture);
        gGraphics.backgroundTexture = 0;
    }
    if (gGraphics.iconTexture != 0) {
        glDeleteTextures(1, &gGraphics.iconTexture);
        gGraphics.iconTexture = 0;
    }
}
```

### Safe Drawing Function

```c
void SafeDrawPanel(int left, int top, int right, int bottom) {
    // Save current OpenGL state if needed
    GLboolean depthTest = glIsEnabled(GL_DEPTH_TEST);
    GLboolean blend = glIsEnabled(GL_BLEND);
  
    // Set up for 2D drawing
    XPLMSetGraphicsState(0, 1, 0, 1, 1, 0, 0);
  
    // Draw background
    XPLMBindTexture2d(gGraphics.backgroundTexture, 0);
    glBegin(GL_QUADS);
    glTexCoord2f(0, 0); glVertex2i(left, bottom);
    glTexCoord2f(1, 0); glVertex2i(right, bottom);
    glTexCoord2f(1, 1); glVertex2i(right, top);
    glTexCoord2f(0, 1); glVertex2i(left, top);
    glEnd();
  
    // Draw text
    float white[] = {1.0f, 1.0f, 1.0f};
    XPLMDrawString(white, left + 10, top - 20, "Panel Title", NULL, xplmFont_Basic);
  
    // Restore state if needed
    if (!depthTest) glDisable(GL_DEPTH_TEST);
    if (!blend) glDisable(GL_BLEND);
}
```

## Best Practices

### OpenGL State Management

```c
// GOOD: Always set required state explicitly
void DrawMyUI() {
    XPLMSetGraphicsState(0, 1, 0, 1, 1, 0, 0);
    // ... drawing code ...
}

// BAD: Assuming state from previous operations
void DrawMyUI() {
    // Don't assume OpenGL state - always set it!
    glEnable(GL_BLEND); // Use XPLMSetGraphicsState instead
}
```

### Coordinate System Usage

```c
// GOOD: Use appropriate coordinate system
void PositionWorldObject(double lat, double lon, double alt) {
    double x, y, z;
    XPLMWorldToLocal(lat, lon, alt, &x, &y, &z);
  
    glPushMatrix();
    glTranslated(x, y, z);
    DrawMyObject();
    glPopMatrix();
}

// BAD: Manual coordinate conversion (inaccurate)
void PositionWorldObject(double lat, double lon, double alt) {
    // Don't do manual lat/lon to meter conversion!
    double x = lon * 111320.0; // This is wrong!
    double z = lat * 110540.0;  // This is wrong!
}
```

### Font Rendering

```c
// GOOD: Measure text for proper layout
void DrawCenteredText(const char* text, int centerX, int y) {
    float width = XPLMMeasureString(xplmFont_Proportional, text, -1);
    int x = centerX - (int)(width / 2.0f);
  
    float white[] = {1.0f, 1.0f, 1.0f};
    XPLMDrawString(white, x, y, text, NULL, xplmFont_Proportional);
}

// ACCEPTABLE: Fixed-width calculations for monospace fonts
void DrawFixedText(const char* text, int x, int y) {
    // Only works reliably with xplmFont_Basic
    float white[] = {1.0f, 1.0f, 1.0f};
    XPLMDrawString(white, x, y, text, NULL, xplmFont_Basic);
}
```

## Common Use Cases

### Custom Instrument Display

```c
typedef struct {
    float altitude;
    float speed;
    float heading;
    float verticalSpeed;
} InstrumentData_t;

void DrawAltitudeIndicator(int x, int y, const InstrumentData_t* data) {
    // Draw background
    XPLMSetGraphicsState(0, 0, 0, 1, 1, 0, 0);
    XPLMDrawTranslucentDarkBox(x, y + 60, x + 100, y);
  
    // Draw altitude
    float green[] = {0.0f, 1.0f, 0.0f};
    XPLMDrawNumber(green, x + 10, y + 35, data->altitude, 5, 0, 0, xplmFont_Basic);
  
    // Draw vertical speed
    float white[] = {1.0f, 1.0f, 1.0f};
    XPLMDrawNumber(white, x + 10, y + 15, data->verticalSpeed, 4, 0, 1, xplmFont_Basic);
}
```

### World Object Positioning

```c
typedef struct {
    double latitude;
    double longitude;  
    double altitude;
    float heading;
    GLuint textureID;
} WorldObject_t;

void DrawWorldObject(const WorldObject_t* obj) {
    // Convert to local coordinates
    double x, y, z;
    XPLMWorldToLocal(obj->latitude, obj->longitude, obj->altitude, &x, &y, &z);
  
    // Set up for 3D textured drawing
    XPLMSetGraphicsState(0, 1, 0, 1, 1, 1, 1);
    XPLMBindTexture2d(obj->textureID, 0);
  
    glPushMatrix();
    glTranslated(x, y, z);
    glRotatef(obj->heading, 0, 1, 0);
  
    // Draw object geometry
    DrawObjectGeometry();
  
    glPopMatrix();
}
```

### 2D Overlay Interface

```c
void DrawStatusOverlay(int screenWidth, int screenHeight) {
    // Position overlay in screen coordinates
    int overlayWidth = 300;
    int overlayHeight = 100;
    int x = screenWidth - overlayWidth - 20;
    int y = screenHeight - overlayHeight - 20;
  
    // Draw semi-transparent background
    XPLMSetGraphicsState(0, 0, 0, 1, 1, 0, 0);
    XPLMDrawTranslucentDarkBox(x, y + overlayHeight, x + overlayWidth, y);
  
    // Draw status information
    float white[] = {1.0f, 1.0f, 1.0f};
    float yellow[] = {1.0f, 1.0f, 0.0f};
  
    XPLMDrawString(white, x + 10, y + 75, "Flight Status", NULL, xplmFont_Basic);
    XPLMDrawString(yellow, x + 10, y + 55, "Autopilot: ON", NULL, xplmFont_Basic);
    XPLMDrawString(yellow, x + 10, y + 35, "Nav Mode: GPS", NULL, xplmFont_Basic);
}
```

## Performance Considerations

### Optimization Strategies

1. **Minimize State Changes**: Group drawing operations with similar state requirements
2. **Cache Font Metrics**: Calculate text dimensions once and reuse
3. **Batch Texture Operations**: Switch textures as infrequently as possible
4. **Avoid Coordinate Conversions**: Cache converted coordinates when possible
5. **Use Appropriate Precision**: Use float for rendering, double for conversions

### Performance Anti-patterns

```c
// BAD: Excessive state changes
void DrawMultipleElements() {
    XPLMSetGraphicsState(0, 1, 0, 1, 1, 0, 0);
    DrawElement1();
    XPLMSetGraphicsState(0, 0, 0, 1, 1, 0, 0);
    DrawElement2();
    XPLMSetGraphicsState(0, 1, 0, 1, 1, 0, 0);  // Redundant!
    DrawElement3();
}

// GOOD: Batch by state requirements
void DrawMultipleElements() {
    // Draw all textured elements
    XPLMSetGraphicsState(0, 1, 0, 1, 1, 0, 0);
    DrawElement1();
    DrawElement3();
  
    // Draw all non-textured elements
    XPLMSetGraphicsState(0, 0, 0, 1, 1, 0, 0);
    DrawElement2();
}
```

## Deprecation Notes

### Deprecated Features

- **Most XPLMTextureID values**: Use custom textures or Widgets library
- **Lighting and Fog**: Modern graphics should avoid fixed-function pipeline features
- **Many XPLMFontID values**: Use xplmFont_Basic or xplmFont_Proportional only

### Migration Guidelines

```c
// OLD (deprecated)
int oldTexture = XPLMGetTexture(xplm_Tex_AircraftPaint);

// NEW (recommended)
GLuint myTexture;
XPLMGenerateTextureNumbers((int*)&myTexture, 1);
LoadMyCustomTexture(myTexture);
```

### Future Compatibility

- Avoid direct OpenGL state manipulation outside of XPLMSetGraphicsState
- Use XPLMInstance API for 3D object rendering instead of direct drawing
- Prefer modern texture management over legacy X-Plane textures
