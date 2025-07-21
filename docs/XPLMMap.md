# XPLMMap.h - X-Plane 12 SDK Map Layer API Documentation

## Overview

The XPLMMap API provides a comprehensive system for creating custom map layers within X-Plane's built-in map displays. This API allows plugins to integrate seamlessly with X-Plane's cartographic rendering system by providing OpenGL drawing capabilities, icon management, text labeling, and true cartographic projections. Unlike simple overlay systems, the Map API respects X-Plane's sophisticated layering model and projection system, ensuring that plugin-created content integrates naturally with built-in map elements.

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Map Layer System](#map-layer-system)
- [Projection System](#projection-system)
- [Data Types and Handles](#data-types-and-handles)
- [Layer Management](#layer-management)
- [Drawing System](#drawing-system)
- [Map Projections](#map-projections)
- [Implementation Guidelines](#implementation-guidelines)
- [Use Cases](#use-cases)
- [Performance Considerations](#performance-considerations)
- [Best Practices](#best-practices)
- [Advanced Techniques](#advanced-techniques)

## Architecture Overview

### Three-Stage Rendering Pipeline

X-Plane 11+ map rendering follows a strict three-stage pipeline that ensures consistent visual layering:

**Stage 1: Background/Fill Drawing**

- Terrain, weather patterns, large area coverage
- Plugin fill layers drawn above native X-Plane fill
- OpenGL drawing callbacks executed

**Stage 2: Icon Drawing**

- Navigation aids, airports, waypoints
- Plugin icons drawn above native X-Plane icons
- Icon drawing callbacks executed

**Stage 3: Label Drawing**

- Text labels, identifiers, annotations
- Plugin labels drawn above all other content
- Label drawing callbacks executed

### Layer Type Hierarchy

The API distinguishes between two fundamental layer types:

**Fill Layers** (`xplm_MapLayer_Fill`):

- Cover large portions of the map area
- Examples: weather systems, terrain overlays, traffic patterns
- Always render beneath markings layers
- Ideal for continuous data visualization

**Markings Layers** (`xplm_MapLayer_Markings`):

- Provide specific point-based information
- Examples: airports, navigation aids, custom waypoints
- Always render above fill layers
- Optimized for discrete feature representation

### Multi-Map Support

The system supports multiple map instances simultaneously:

- **Main UI Map** (`XPLM_MAP_USER_INTERFACE`): Primary user map interface
- **Instructor Station** (`XPLM_MAP_IOS`): Instructor Operator Station display
- **Custom Maps**: Third-party map implementations

Each map instance maintains its own layer stack and projection system, allowing for specialized content per display type.

## Map Layer System

### XPLMMapLayerID

```c
typedef void * XPLMMapLayerID;
```

**Purpose**: Opaque handle representing a plugin-created map layer.

**Characteristics**:

- Unique per layer instance
- Tied to specific map instance lifetime
- Required for all layer-specific operations
- Automatically invalidated when parent map is destroyed

### Map Styles

```c
enum {
    xplm_MapStyle_VFR_Sectional              = 0,
    xplm_MapStyle_IFR_LowEnroute             = 1,
    xplm_MapStyle_IFR_HighEnroute            = 2,
};
typedef int XPLMMapStyle;
```

**Style-Responsive Rendering**: Different map styles may require different visual representations of the same data. For example, navigation aids might be shown differently between VFR sectional and IFR enroute charts.

**Implementation Strategy**:

```c
void MyDrawingCallback(XPLMMapLayerID layer,
                      const float* bounds,
                      float zoomRatio,
                      float mapUnitsPerUIUnit,
                      XPLMMapStyle mapStyle,
                      XPLMMapProjectionID projection,
                      void* refcon) {
  
    switch (mapStyle) {
        case xplm_MapStyle_VFR_Sectional:
            // Draw VFR-appropriate symbols and colors
            DrawVFRNavaids(projection);
            break;
          
        case xplm_MapStyle_IFR_LowEnroute:
        case xplm_MapStyle_IFR_HighEnroute:
            // Draw IFR-appropriate symbols and colors
            DrawIFRFeatures(projection, mapStyle);
            break;
    }
}
```

## Drawing System

### Drawing Callback Types

#### XPLMMapDrawingCallback_f

```c
typedef void (* XPLMMapDrawingCallback_f)(
    XPLMMapLayerID       inLayer,
    const float *        inMapBoundsLeftTopRightBottom,
    float                zoomRatio,
    float                mapUnitsPerUserInterfaceUnit,
    XPLMMapStyle         mapStyle,
    XPLMMapProjectionID  projection,
    void *               inRefcon);
```

**Purpose**: Enables arbitrary OpenGL drawing within the map coordinate system.

**Parameters**:

- `inLayer`: Handle to your layer (for identification)
- `inMapBoundsLeftTopRightBottom`: Array [left, top, right, bottom] in map coordinates
- `zoomRatio`: Current zoom level (1.0 = default zoom)
- `mapUnitsPerUserInterfaceUnit`: Scale factor for UI consistency
- `mapStyle`: Current map visual style
- `projection`: Handle for coordinate transformations
- `inRefcon`: User-provided reference data

**Critical Restrictions**:

- **No Z-buffer modifications** - will cause rendering errors
- Must use provided projection for coordinate conversions
- Drawing appears beneath all icons and labels

**Example Implementation**:

```c
void DrawWeatherLayer(XPLMMapLayerID layer,
                     const float* bounds,
                     float zoomRatio,
                     float mapUnitsPerUIUnit,
                     XPLMMapStyle mapStyle,
                     XPLMMapProjectionID projection,
                     void* refcon) {
  
    WeatherSystem* weather = (WeatherSystem*)refcon;
  
    // Set OpenGL state for transparent rendering
    glEnable(GL_BLEND);
    glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA);
  
    // Draw weather cells
    for (int i = 0; i < weather->cellCount; i++) {
        WeatherCell* cell = &weather->cells[i];
      
        // Project lat/lon to map coordinates
        float mapX, mapY;
        XPLMMapProject(projection, cell->latitude, cell->longitude, &mapX, &mapY);
      
        // Skip cells outside visible area
        if (mapX < bounds[0] || mapX > bounds[2] || 
            mapY < bounds[3] || mapY > bounds[1]) {
            continue;
        }
      
        // Calculate visual radius based on intensity and zoom
        float radius = cell->intensity * 5000.0f * XPLMMapScaleMeter(projection, mapX, mapY);
        radius *= zoomRatio;  // Scale with zoom level
      
        // Set color based on weather type and intensity
        SetWeatherColor(cell->type, cell->intensity);
      
        // Draw circular weather cell
        glBegin(GL_TRIANGLE_FAN);
        glVertex2f(mapX, mapY);  // Center
      
        for (int angle = 0; angle <= 360; angle += 10) {
            float rad = angle * M_PI / 180.0f;
            glVertex2f(mapX + radius * cos(rad), mapY + radius * sin(rad));
        }
        glEnd();
    }
  
    glDisable(GL_BLEND);
}
```

#### XPLMMapIconDrawingCallback_f

```c
typedef void (* XPLMMapIconDrawingCallback_f)(
    XPLMMapLayerID       inLayer,
    const float *        inMapBoundsLeftTopRightBottom,
    float                zoomRatio,
    float                mapUnitsPerUserInterfaceUnit,
    XPLMMapStyle         mapStyle,
    XPLMMapProjectionID  projection,
    void *               inRefcon);
```

**Purpose**: Enables drawing of PNG icons using X-Plane's built-in icon system.

**Restrictions**:

- **No OpenGL drawing allowed** - icons only
- Use `XPLMDrawMapIconFromSheet()` exclusively
- Icons appear above all OpenGL drawing

**Example Implementation**:

```c
void DrawAirportIcons(XPLMMapLayerID layer,
                     const float* bounds, 
                     float zoomRatio,
                     float mapUnitsPerUIUnit,
                     XPLMMapStyle mapStyle,
                     XPLMMapProjectionID projection,
                     void* refcon) {
  
    AirportDatabase* airports = (AirportDatabase*)refcon;
  
    // Calculate appropriate icon size based on zoom
    float iconSize = 32.0f * mapUnitsPerUIUnit;
    if (zoomRatio < 0.5f) {
        iconSize *= 0.7f;  // Smaller icons when zoomed out
    }
  
    // Draw airport icons
    for (int i = 0; i < airports->count; i++) {
        Airport* airport = &airports->data[i];
      
        // Project airport location
        float mapX, mapY;
        XPLMMapProject(projection, airport->latitude, airport->longitude, &mapX, &mapY);
      
        // Cull airports outside visible bounds
        if (mapX < bounds[0] - iconSize || mapX > bounds[2] + iconSize ||
            mapY < bounds[3] - iconSize || mapY > bounds[1] + iconSize) {
            continue;
        }
      
        // Select icon based on airport type and map style
        int iconS = 0, iconT = 0;
        if (airport->hasControlTower) {
            iconS = 0; iconT = 0;  // Towered airport icon
        } else if (airport->longestRunway > 3000) {
            iconS = 1; iconT = 0;  // Large non-towered airport
        } else {
            iconS = 2; iconT = 0;  // Small airport
        }
      
        // Different icons for different map styles
        if (mapStyle == xplm_MapStyle_IFR_LowEnroute || 
            mapStyle == xplm_MapStyle_IFR_HighEnroute) {
            iconT = 1;  // Use IFR-style icons (second row)
        }
      
        // Draw the icon
        XPLMDrawMapIconFromSheet(layer,
            "Resources/plugins/MyPlugin/icons/airports.png",
            iconS, iconT,        // Source coordinates in icon sheet
            3, 2,               // Sheet dimensions (3x2 grid)
            mapX, mapY,         // Map position
            xplm_MapOrientation_Map,  // Rotate with map
            0.0f,               // No additional rotation
            iconSize);          // Icon size in map units
    }
}
```

#### XPLMMapLabelDrawingCallback_f

```c
typedef void (* XPLMMapLabelDrawingCallback_f)(
    XPLMMapLayerID       inLayer,
    const float *        inMapBoundsLeftTopRightBottom, 
    float                zoomRatio,
    float                mapUnitsPerUserInterfaceUnit,
    XPLMMapStyle         mapStyle,
    XPLMMapProjectionID  projection,
    void *               inRefcon);
```

**Purpose**: Enables drawing of text labels using X-Plane's built-in labeling system.

**Restrictions**:

- **No OpenGL drawing allowed** - labels only
- Use `XPLMDrawMapLabel()` exclusively
- Labels appear above all other content

**Example Implementation**:

```c
void DrawNavaidLabels(XPLMMapLayerID layer,
                     const float* bounds,
                     float zoomRatio, 
                     float mapUnitsPerUIUnit,
                     XPLMMapStyle mapStyle,
                     XPLMMapProjectionID projection,
                     void* refcon) {
  
    NavaidDatabase* navaids = (NavaidDatabase*)refcon;
  
    // Only show labels when sufficiently zoomed in
    if (zoomRatio < 0.3f) return;
  
    for (int i = 0; i < navaids->count; i++) {
        Navaid* nav = &navaids->data[i];
      
        // Project navaid location
        float mapX, mapY;
        XPLMMapProject(projection, nav->latitude, nav->longitude, &mapX, &mapY);
      
        // Cull labels outside bounds (with padding for text)
        float padding = 100.0f * mapUnitsPerUIUnit;
        if (mapX < bounds[0] - padding || mapX > bounds[2] + padding ||
            mapY < bounds[3] - padding || mapY > bounds[1] + padding) {
            continue;
        }
      
        // Create label text based on zoom level and navaid type
        char labelText[64];
        if (zoomRatio > 1.0f) {
            // High zoom: show full information
            sprintf(labelText, "%s\n%.2f", nav->identifier, nav->frequency);
        } else {
            // Medium zoom: just identifier
            strcpy(labelText, nav->identifier);
        }
      
        // Position label slightly offset from navaid position
        float labelX = mapX + (20.0f * mapUnitsPerUIUnit);
        float labelY = mapY + (10.0f * mapUnitsPerUIUnit);
      
        // Calculate label rotation to match north
        float northHeading = XPLMMapGetNorthHeading(projection, mapX, mapY);
      
        XPLMDrawMapLabel(layer,
            labelText,
            labelX, labelY,
            xplm_MapOrientation_UI,  // Keep labels upright relative to UI
            0.0f);                   // No rotation
    }
}
```

## Layer Management

### Layer Creation

#### XPLMCreateMapLayer_t

```c
typedef struct {
    int                       structSize;
    const char *              mapToCreateLayerIn;
    XPLMMapLayerType          layerType;
    XPLMMapWillBeDeletedCallback_f willBeDeletedCallback;
    XPLMMapPrepareCacheCallback_f prepCacheCallback;
    XPLMMapDrawingCallback_f  drawCallback;
    XPLMMapIconDrawingCallback_f iconCallback;
    XPLMMapLabelDrawingCallback_f labelCallback;
    int                       showUiToggle;
    const char *              layerName;
    void *                    refcon;
} XPLMCreateMapLayer_t;
```

**Critical Fields**:

- `structSize`: Always set to `sizeof(XPLMCreateMapLayer_t)`
- `mapToCreateLayerIn`: Must be `XPLM_MAP_USER_INTERFACE` or `XPLM_MAP_IOS`
- `layerType`: Determines drawing order (`Fill` beneath `Markings`)
- `showUiToggle`: Creates user-controllable layer visibility checkbox

**Complete Layer Setup Example**:

```c
// Global layer management
typedef struct {
    XPLMMapLayerID layerID;
    TrafficData* trafficData;
    int isVisible;
    int needsCacheRefresh;
} TrafficLayer;

static TrafficLayer gTrafficLayer = {0};

// Layer creation
int CreateTrafficLayer() {
    // Ensure map exists before creating layer
    if (!XPLMMapExists(XPLM_MAP_USER_INTERFACE)) {
        return 0;  // Map not available yet
    }
  
    // Initialize traffic data
    gTrafficLayer.trafficData = LoadTrafficData();
    if (!gTrafficLayer.trafficData) {
        return 0;
    }
  
    // Configure layer creation parameters
    XPLMCreateMapLayer_t layerDef;
    memset(&layerDef, 0, sizeof(layerDef));
  
    layerDef.structSize = sizeof(XPLMCreateMapLayer_t);
    layerDef.mapToCreateLayerIn = XPLM_MAP_USER_INTERFACE;
    layerDef.layerType = xplm_MapLayer_Markings;  // Traffic icons above terrain
    layerDef.willBeDeletedCallback = TrafficLayerWillBeDeleted;
    layerDef.prepCacheCallback = TrafficLayerPrepareCache;
    layerDef.drawCallback = NULL;              // No OpenGL drawing needed
    layerDef.iconCallback = TrafficLayerDrawIcons;
    layerDef.labelCallback = TrafficLayerDrawLabels;
    layerDef.showUiToggle = 1;                // User can toggle visibility
    layerDef.layerName = "AI Traffic";        // Label in map UI
    layerDef.refcon = &gTrafficLayer;         // Pass our data structure
  
    // Create the layer
    gTrafficLayer.layerID = XPLMCreateMapLayer(&layerDef);
    if (!gTrafficLayer.layerID) {
        ReleaseTrafficData(gTrafficLayer.trafficData);
        return 0;
    }
  
    gTrafficLayer.isVisible = 1;
    gTrafficLayer.needsCacheRefresh = 1;
  
    return 1;
}

// Layer cleanup callback
void TrafficLayerWillBeDeleted(XPLMMapLayerID layer, void* refcon) {
    TrafficLayer* trafficLayer = (TrafficLayer*)refcon;
  
    // Clean up resources
    if (trafficLayer->trafficData) {
        ReleaseTrafficData(trafficLayer->trafficData);
        trafficLayer->trafficData = NULL;
    }
  
    trafficLayer->layerID = NULL;
    trafficLayer->isVisible = 0;
}
```

### Cache Management

#### XPLMMapPrepareCacheCallback_f

```c
typedef void (* XPLMMapPrepareCacheCallback_f)(
    XPLMMapLayerID       inLayer,
    const float *        inTotalMapBoundsLeftTopRightBottom,
    XPLMMapProjectionID  projection,
    void *               inRefcon);
```

**Purpose**: Pre-compute expensive operations when map bounds change.

**Key Benefits**:

- Projection won't change until next cache preparation
- Can pre-project all coordinates for visible area
- Significantly improves drawing performance

**Example Implementation**:

```c
typedef struct {
    float mapX, mapY;           // Pre-projected coordinates
    float iconSize;             // Pre-calculated size
    int isVisible;              // Visibility flag
    char labelText[32];         // Pre-formatted label
} CachedWaypoint;

typedef struct {
    CachedWaypoint* waypoints;
    int waypointCount;
    int cacheVersion;
} WaypointCache;

void WaypointLayerPrepareCache(XPLMMapLayerID layer,
                              const float* totalBounds,
                              XPLMMapProjectionID projection,
                              void* refcon) {
  
    WaypointCache* cache = (WaypointCache*)refcon;
  
    // Clear previous cache
    if (cache->waypoints) {
        free(cache->waypoints);
        cache->waypoints = NULL;
        cache->waypointCount = 0;
    }
  
    // Get source waypoint data
    WaypointDatabase* db = GetWaypointDatabase();
  
    // Allocate cache array
    cache->waypoints = malloc(sizeof(CachedWaypoint) * db->count);
    if (!cache->waypoints) return;
  
    // Pre-process all waypoints for current bounds
    int cacheIndex = 0;
    for (int i = 0; i < db->count; i++) {
        Waypoint* wp = &db->waypoints[i];
      
        // Project to map coordinates once
        float mapX, mapY;
        XPLMMapProject(projection, wp->latitude, wp->longitude, &mapX, &mapY);
      
        // Check if waypoint could be visible (with generous padding)
        float padding = 10000.0f;  // 10km padding
        if (mapX < totalBounds[0] - padding || mapX > totalBounds[2] + padding ||
            mapY < totalBounds[3] - padding || mapY > totalBounds[1] + padding) {
            continue;  // Outside possible viewing area
        }
      
        // Cache this waypoint
        CachedWaypoint* cached = &cache->waypoints[cacheIndex];
        cached->mapX = mapX;
        cached->mapY = mapY;
        cached->isVisible = 1;
      
        // Pre-calculate icon size based on waypoint type
        if (wp->type == WAYPOINT_AIRPORT) {
            cached->iconSize = 48.0f;
        } else if (wp->type == WAYPOINT_VOR) {
            cached->iconSize = 32.0f;
        } else {
            cached->iconSize = 24.0f;
        }
      
        // Pre-format label text
        sprintf(cached->labelText, "%s", wp->identifier);
      
        cacheIndex++;
    }
  
    cache->waypointCount = cacheIndex;
    cache->cacheVersion++;
  
    // Debug output
    char msg[128];
    sprintf(msg, "Cached %d waypoints for current map area\n", cacheIndex);
    XPLMDebugString(msg);
}

// Fast drawing using cached data
void WaypointLayerDrawIcons(XPLMMapLayerID layer,
                           const float* bounds,
                           float zoomRatio,
                           float mapUnitsPerUIUnit,
                           XPLMMapStyle mapStyle,
                           XPLMMapProjectionID projection,
                           void* refcon) {
  
    WaypointCache* cache = (WaypointCache*)refcon;
    if (!cache->waypoints) return;
  
    // Iterate through pre-computed waypoints
    for (int i = 0; i < cache->waypointCount; i++) {
        CachedWaypoint* wp = &cache->waypoints[i];
      
        // Quick bounds check using pre-projected coordinates
        float iconRadius = wp->iconSize * mapUnitsPerUIUnit;
        if (wp->mapX < bounds[0] - iconRadius || wp->mapX > bounds[2] + iconRadius ||
            wp->mapY < bounds[3] - iconRadius || wp->mapY > bounds[1] + iconRadius) {
            continue;
        }
      
        // Scale icon with zoom level
        float scaledSize = wp->iconSize * mapUnitsPerUIUnit * zoomRatio;
      
        // Draw icon (coordinates already projected!)
        XPLMDrawMapIconFromSheet(layer,
            "Resources/plugins/MyPlugin/icons/waypoints.png",
            0, 0,  // Icon coordinates in sheet
            4, 4,  // Sheet is 4x4 grid
            wp->mapX, wp->mapY,
            xplm_MapOrientation_Map,
            0.0f,
            scaledSize);
    }
}
```

## Map Projections

### Coordinate Transformation

#### XPLMMapProject / XPLMMapUnproject

```c
XPLM_API void XPLMMapProject(
    XPLMMapProjectionID  projection,
    double               latitude,
    double               longitude, 
    float *              outX,
    float *              outY);

XPLM_API void XPLMMapUnproject(
    XPLMMapProjectionID  projection,
    float                mapX,
    float                mapY,
    double *             outLatitude,
    double *             outLongitude);
```

**Usage Patterns**:

```c
// Forward projection: lat/lon to map coordinates
double aircraftLat = 37.621311;   // San Francisco
double aircraftLon = -122.378968;
float mapX, mapY;
XPLMMapProject(projection, aircraftLat, aircraftLon, &mapX, &mapY);

// Reverse projection: map coordinates to lat/lon
double clickLat, clickLon;
XPLMMapUnproject(projection, mouseX, mouseY, &clickLat, &clickLon);
```

#### XPLMMapScaleMeter

```c
XPLM_API float XPLMMapScaleMeter(
    XPLMMapProjectionID  projection,
    float                mapX,
    float                mapY);
```

**Purpose**: Converts real-world distances to map coordinate distances.

**Usage Examples**:

```c
// Draw a 5-nautical-mile range ring
float centerX, centerY;
XPLMMapProject(projection, navaidLat, navaidLon, &centerX, &centerY);

float metersPerMapUnit = XPLMMapScaleMeter(projection, centerX, centerY);
float nauticalMileInMeters = 1852.0f;
float ringRadiusMapUnits = 5.0f * nauticalMileInMeters * metersPerMapUnit;

// Draw circular range ring
glBegin(GL_LINE_LOOP);
for (int angle = 0; angle < 360; angle += 10) {
    float rad = angle * M_PI / 180.0f;
    float x = centerX + ringRadiusMapUnits * cos(rad);
    float y = centerY + ringRadiusMapUnits * sin(rad);
    glVertex2f(x, y);
}
glEnd();
```

#### XPLMMapGetNorthHeading

```c
XPLM_API float XPLMMapGetNorthHeading(
    XPLMMapProjectionID  projection,
    float                mapX,
    float                mapY);
```

**Purpose**: Compensates for map rotation and projection distortion.

**Usage Examples**:

```c
// Draw runway aligned to true direction
void DrawRunway(XPLMMapProjectionID projection, 
               float mapX, float mapY,
               float trueHeading,
               float lengthMeters) {
  
    // Get current north heading at this position
    float northOffset = XPLMMapGetNorthHeading(projection, mapX, mapY);
  
    // Calculate runway direction in map coordinates
    float mapHeading = trueHeading + northOffset;
    float rad = mapHeading * M_PI / 180.0f;
  
    // Convert runway length to map units
    float scale = XPLMMapScaleMeter(projection, mapX, mapY);
    float mapLength = lengthMeters * scale;
  
    // Calculate runway endpoints
    float dx = mapLength * sin(rad) * 0.5f;
    float dy = mapLength * cos(rad) * 0.5f;
  
    // Draw runway
    glBegin(GL_LINES);
    glVertex2f(mapX - dx, mapY - dy);
    glVertex2f(mapX + dx, mapY + dy);
    glEnd();
}
```

## Implementation Guidelines

### Map Discovery and Layer Creation

```c
// Plugin initialization
static int gLayersCreated = 0;

int XPluginStart(char* outName, char* outSig, char* outDesc) {
    strcpy(outName, "Map Layer Plugin");
    strcpy(outSig, "com.example.maplayer");
    strcpy(outDesc, "Example map layer integration");
  
    // Register for map creation notifications
    XPLMRegisterMapCreationHook(OnMapCreated, NULL);
  
    // Check for existing maps
    if (XPLMMapExists(XPLM_MAP_USER_INTERFACE)) {
        CreateLayerInMap(XPLM_MAP_USER_INTERFACE);
    }
  
    if (XPLMMapExists(XPLM_MAP_IOS)) {
        CreateLayerInMap(XPLM_MAP_IOS);
    }
  
    return 1;
}

// Map creation notification
void OnMapCreated(const char* mapIdentifier, void* refcon) {
    CreateLayerInMap(mapIdentifier);
}

void CreateLayerInMap(const char* mapID) {
    // Avoid duplicate layer creation
    if (strcmp(mapID, XPLM_MAP_USER_INTERFACE) == 0 && gMainUILayer) {
        return;  // Already created
    }
  
    XPLMCreateMapLayer_t params;
    memset(&params, 0, sizeof(params));
    params.structSize = sizeof(params);
    params.mapToCreateLayerIn = mapID;
    params.layerType = xplm_MapLayer_Markings;
    params.willBeDeletedCallback = LayerWillBeDeleted;
    params.prepCacheCallback = LayerPrepareCache;
    params.iconCallback = LayerDrawIcons;
    params.labelCallback = LayerDrawLabels;
    params.showUiToggle = 1;
    params.layerName = "Custom Layer";
    params.refcon = CreateLayerData(mapID);
  
    XPLMMapLayerID layer = XPLMCreateMapLayer(&params);
    if (layer) {
        RegisterLayer(mapID, layer);
        gLayersCreated++;
    }
}
```

### Resource Management

```c
// Proper resource cleanup
typedef struct {
    char* iconTexturePath;
    int* iconSheet;
    int iconSheetSize;
    CachedData* cache;
    int isActive;
} LayerResources;

void LayerWillBeDeleted(XPLMMapLayerID layer, void* refcon) {
    LayerResources* resources = (LayerResources*)refcon;
  
    // Clean up cached data
    if (resources->cache) {
        FreeCachedData(resources->cache);
        resources->cache = NULL;
    }
  
    // Clean up icon resources  
    if (resources->iconSheet) {
        free(resources->iconSheet);
        resources->iconSheet = NULL;
    }
  
    if (resources->iconTexturePath) {
        free(resources->iconTexturePath);
        resources->iconTexturePath = NULL;
    }
  
    resources->isActive = 0;
    free(resources);
}

// Plugin shutdown
void XPluginStop(void) {
    // Layers are automatically destroyed when their maps are destroyed
    // Just clean up any global resources
  
    CleanupGlobalMapData();
}
```

## Use Cases

### 1. Weather Radar Overlay

```c
// Weather radar system
typedef struct {
    float latitude, longitude;
    float intensity;          // 0.0 to 1.0
    float size;              // Radius in meters
    int weatherType;         // Rain, snow, hail, etc.
    time_t timestamp;
} WeatherCell;

typedef struct {
    WeatherCell* cells;
    int cellCount;
    int maxCells;
    time_t lastUpdate;
} WeatherData;

void WeatherLayerDraw(XPLMMapLayerID layer,
                     const float* bounds,
                     float zoomRatio,
                     float mapUnitsPerUIUnit, 
                     XPLMMapStyle mapStyle,
                     XPLMMapProjectionID projection,
                     void* refcon) {
  
    WeatherData* weather = (WeatherData*)refcon;
    time_t currentTime = time(NULL);
  
    // Set up blending for weather transparency
    glEnable(GL_BLEND);
    glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA);
  
    for (int i = 0; i < weather->cellCount; i++) {
        WeatherCell* cell = &weather->cells[i];
      
        // Age cells over time
        float age = (float)(currentTime - cell->timestamp);
        if (age > 300.0f) continue;  // Skip cells older than 5 minutes
      
        float ageFactor = 1.0f - (age / 300.0f);  // Fade with age
      
        // Project to map coordinates
        float mapX, mapY;
        XPLMMapProject(projection, cell->latitude, cell->longitude, &mapX, &mapY);
      
        // Calculate cell size in map units
        float scale = XPLMMapScaleMeter(projection, mapX, mapY);
        float radius = cell->size * scale;
      
        // Skip tiny or out-of-bounds cells
        if (radius < 2.0f || mapX < bounds[0] - radius || mapX > bounds[2] + radius ||
            mapY < bounds[3] - radius || mapY > bounds[1] + radius) {
            continue;
        }
      
        // Set color based on weather type and intensity
        float alpha = cell->intensity * ageFactor * 0.6f;
      
        switch (cell->weatherType) {
            case WEATHER_RAIN:
                glColor4f(0.0f, 1.0f, 0.0f, alpha);  // Green
                break;
            case WEATHER_SNOW:
                glColor4f(0.8f, 0.8f, 1.0f, alpha);  // Light blue
                break;
            case WEATHER_THUNDERSTORM:
                glColor4f(1.0f, 0.0f, 0.0f, alpha);  // Red
                break;
            default:
                glColor4f(0.5f, 0.5f, 0.5f, alpha);  // Gray
        }
      
        // Draw filled circle
        glBegin(GL_TRIANGLE_FAN);
        glVertex2f(mapX, mapY);
      
        for (int angle = 0; angle <= 360; angle += 15) {
            float rad = angle * M_PI / 180.0f;
            glVertex2f(mapX + radius * cos(rad), mapY + radius * sin(rad));
        }
        glEnd();
      
        // Draw intensity ring for stronger cells
        if (cell->intensity > 0.7f) {
            glColor4f(1.0f, 1.0f, 0.0f, alpha * 0.8f);  // Yellow ring
            glBegin(GL_LINE_LOOP);
            for (int angle = 0; angle < 360; angle += 10) {
                float rad = angle * M_PI / 180.0f;
                glVertex2f(mapX + radius * 1.2f * cos(rad), 
                          mapY + radius * 1.2f * sin(rad));
            }
            glEnd();
        }
    }
  
    glDisable(GL_BLEND);
}
```

### 2. Flight Plan Visualization

```c
// Flight plan display system
typedef struct {
    double latitude, longitude;
    char identifier[16];
    int waypointType;  // Airport, waypoint, navaid
    float altitude;    // Target altitude MSL
} FlightPlanWaypoint;

typedef struct {
    FlightPlanWaypoint* waypoints;
    int waypointCount;
    int activeWaypoint;  // Currently active waypoint index
    float totalDistance;
    float distanceRemaining;
} FlightPlan;

void FlightPlanLayerDraw(XPLMMapLayerID layer,
                        const float* bounds,
                        float zoomRatio,
                        float mapUnitsPerUIUnit,
                        XPLMMapStyle mapStyle, 
                        XPLMMapProjectionID projection,
                        void* refcon) {
  
    FlightPlan* plan = (FlightPlan*)refcon;
    if (plan->waypointCount < 2) return;
  
    // Draw flight plan route
    glColor4f(1.0f, 0.0f, 1.0f, 0.8f);  // Magenta route line
    glLineWidth(3.0f * mapUnitsPerUIUnit);
  
    glBegin(GL_LINE_STRIP);
    for (int i = 0; i < plan->waypointCount; i++) {
        float mapX, mapY;
        XPLMMapProject(projection, 
                      plan->waypoints[i].latitude,
                      plan->waypoints[i].longitude,
                      &mapX, &mapY);
        glVertex2f(mapX, mapY);
    }
    glEnd();
  
    // Highlight active leg
    if (plan->activeWaypoint < plan->waypointCount - 1) {
        glColor4f(1.0f, 1.0f, 0.0f, 1.0f);  // Yellow active leg
        glLineWidth(5.0f * mapUnitsPerUIUnit);
      
        glBegin(GL_LINES);
      
        // From waypoint
        float fromX, fromY;
        XPLMMapProject(projection,
                      plan->waypoints[plan->activeWaypoint].latitude,
                      plan->waypoints[plan->activeWaypoint].longitude,
                      &fromX, &fromY);
        glVertex2f(fromX, fromY);
      
        // To waypoint
        float toX, toY;
        XPLMMapProject(projection,
                      plan->waypoints[plan->activeWaypoint + 1].latitude,
                      plan->waypoints[plan->activeWaypoint + 1].longitude,
                      &toX, &toY);
        glVertex2f(toX, toY);
      
        glEnd();
    }
  
    glLineWidth(1.0f);
}

void FlightPlanLayerDrawIcons(XPLMMapLayerID layer,
                             const float* bounds,
                             float zoomRatio,
                             float mapUnitsPerUIUnit,
                             XPLMMapStyle mapStyle,
                             XPLMMapProjectionID projection,
                             void* refcon) {
  
    FlightPlan* plan = (FlightPlan*)refcon;
  
    for (int i = 0; i < plan->waypointCount; i++) {
        FlightPlanWaypoint* wp = &plan->waypoints[i];
      
        float mapX, mapY;
        XPLMMapProject(projection, wp->latitude, wp->longitude, &mapX, &mapY);
      
        // Cull out-of-bounds waypoints
        float iconSize = 32.0f * mapUnitsPerUIUnit;
        if (mapX < bounds[0] - iconSize || mapX > bounds[2] + iconSize ||
            mapY < bounds[3] - iconSize || mapY > bounds[1] + iconSize) {
            continue;
        }
      
        // Select icon based on waypoint type
        int iconS = 0, iconT = 0;
        switch (wp->waypointType) {
            case WP_AIRPORT:
                iconS = 0; iconT = 0;
                break;
            case WP_WAYPOINT:
                iconS = 1; iconT = 0;
                break;
            case WP_VOR:
                iconS = 2; iconT = 0;
                break;
            case WP_NDB:
                iconS = 3; iconT = 0;
                break;
        }
      
        // Highlight active waypoint
        if (i == plan->activeWaypoint) {
            iconT = 1;  // Use highlighted row of icons
        }
      
        XPLMDrawMapIconFromSheet(layer,
            "Resources/plugins/MyPlugin/icons/waypoints.png",
            iconS, iconT,
            4, 2,  // 4x2 icon sheet
            mapX, mapY,
            xplm_MapOrientation_Map,
            0.0f,
            iconSize);
    }
}

void FlightPlanLayerDrawLabels(XPLMMapLayerID layer,
                              const float* bounds,
                              float zoomRatio,
                              float mapUnitsPerUIUnit,
                              XPLMMapStyle mapStyle,
                              XPLMMapProjectionID projection,
                              void* refcon) {
  
    FlightPlan* plan = (FlightPlan*)refcon;
  
    // Only show labels when zoomed in enough
    if (zoomRatio < 0.4f) return;
  
    for (int i = 0; i < plan->waypointCount; i++) {
        FlightPlanWaypoint* wp = &plan->waypoints[i];
      
        float mapX, mapY;
        XPLMMapProject(projection, wp->latitude, wp->longitude, &mapX, &mapY);
      
        // Position label below waypoint icon
        float labelX = mapX;
        float labelY = mapY - (40.0f * mapUnitsPerUIUnit);
      
        // Create detailed label text
        char labelText[64];
        if (zoomRatio > 1.0f) {
            // High zoom: show altitude information
            sprintf(labelText, "%s\n%.0fft", wp->identifier, wp->altitude);
        } else {
            // Medium zoom: just identifier
            strcpy(labelText, wp->identifier);
        }
      
        XPLMDrawMapLabel(layer,
            labelText,
            labelX, labelY,
            xplm_MapOrientation_UI,
            0.0f);
    }
}
```

### 3. Real-Time Traffic Display

```c
// AI traffic display system
typedef struct {
    char callsign[16];
    double latitude, longitude;
    float altitude;      // MSL in feet
    float heading;       // True heading
    float speed;         // Ground speed in knots
    int aircraftType;    // Icon selection
    time_t lastUpdate;
} TrafficTarget;

typedef struct {
    TrafficTarget* targets;
    int targetCount;
    int maxTargets;
    double ownshipLat, ownshipLon;
    float ownshipAlt;
} TrafficSystem;

void TrafficLayerDrawIcons(XPLMMapLayerID layer,
                          const float* bounds,
                          float zoomRatio,
                          float mapUnitsPerUIUnit,
                          XPLMMapStyle mapStyle,
                          XPLMMapProjectionID projection,
                          void* refcon) {
  
    TrafficSystem* traffic = (TrafficSystem*)refcon;
    time_t currentTime = time(NULL);
  
    for (int i = 0; i < traffic->targetCount; i++) {
        TrafficTarget* target = &traffic->targets[i];
      
        // Skip stale targets
        if ((currentTime - target->lastUpdate) > 10) {
            continue;  // Target older than 10 seconds
        }
      
        float mapX, mapY;
        XPLMMapProject(projection, target->latitude, target->longitude, &mapX, &mapY);
      
        // Cull out-of-bounds targets
        float iconSize = 24.0f * mapUnitsPerUIUnit;
        if (mapX < bounds[0] - iconSize || mapX > bounds[2] + iconSize ||
            mapY < bounds[3] - iconSize || mapY > bounds[1] + iconSize) {
            continue;
        }
      
        // Calculate relative altitude
        float altDiff = target->altitude - traffic->ownshipAlt;
      
        // Select icon based on relative altitude
        int iconS = 0, iconT = 0;
        if (altDiff > 1000.0f) {
            iconT = 0;  // Above
        } else if (altDiff < -1000.0f) {
            iconT = 2;  // Below
        } else {
            iconT = 1;  // Same level
        }
      
        // Aircraft type affects icon column
        switch (target->aircraftType) {
            case AIRCRAFT_AIRLINER:
                iconS = 0;
                break;
            case AIRCRAFT_GA:
                iconS = 1;
                break;
            case AIRCRAFT_HELICOPTER:
                iconS = 2;
                break;
            default:
                iconS = 3;  // Unknown
        }
      
        // Calculate north heading for rotation
        float northHeading = XPLMMapGetNorthHeading(projection, mapX, mapY);
        float iconRotation = target->heading + northHeading;
      
        XPLMDrawMapIconFromSheet(layer,
            "Resources/plugins/MyPlugin/icons/traffic.png",
            iconS, iconT,
            4, 3,  // 4x3 icon sheet
            mapX, mapY,
            xplm_MapOrientation_Map,
            iconRotation,  // Rotate to show heading
            iconSize);
    }
}

void TrafficLayerDrawLabels(XPLMMapLayerID layer,
                           const float* bounds,
                           float zoomRatio,
                           float mapUnitsPerUIUnit,
                           XPLMMapStyle mapStyle,
                           XPLMMapProjectionID projection,
                           void* refcon) {
  
    TrafficSystem* traffic = (TrafficSystem*)refcon;
  
    // Only show labels when significantly zoomed in
    if (zoomRatio < 0.6f) return;
  
    time_t currentTime = time(NULL);
  
    for (int i = 0; i < traffic->targetCount; i++) {
        TrafficTarget* target = &traffic->targets[i];
      
        if ((currentTime - target->lastUpdate) > 10) {
            continue;
        }
      
        float mapX, mapY;
        XPLMMapProject(projection, target->latitude, target->longitude, &mapX, &mapY);
      
        // Position label to the right of aircraft icon
        float labelX = mapX + (30.0f * mapUnitsPerUIUnit);
        float labelY = mapY + (10.0f * mapUnitsPerUIUnit);
      
        // Create informative label
        char labelText[64];
        float altDiff = target->altitude - traffic->ownshipAlt;
      
        sprintf(labelText, "%s\n%+.0f %dkt", 
                target->callsign,
                altDiff / 100.0f,  // Show as flight levels
                (int)target->speed);
      
        XPLMDrawMapLabel(layer,
            labelText,
            labelX, labelY,
            xplm_MapOrientation_UI,
            0.0f);
    }
}
```

## Performance Considerations

### Efficient Culling

```c
// Hierarchical culling system
typedef struct {
    float centerX, centerY;
    float radius;
    int itemCount;
    void** items;
} SpatialBucket;

typedef struct {
    SpatialBucket* buckets;
    int bucketCount;
    float bucketSize;
} SpatialIndex;

void UpdateSpatialIndex(SpatialIndex* index, XPLMMapProjectionID projection) {
    // Rebuild spatial index for current projection
    ClearSpatialIndex(index);
  
    // Get all items to be indexed
    ItemDatabase* db = GetItemDatabase();
  
    for (int i = 0; i < db->count; i++) {
        Item* item = &db->items[i];
      
        // Project item location
        float mapX, mapY;
        XPLMMapProject(projection, item->latitude, item->longitude, &mapX, &mapY);
      
        // Add to appropriate spatial bucket
        AddToSpatialBucket(index, item, mapX, mapY);
    }
}

void DrawWithSpatialCulling(SpatialIndex* index, const float* bounds) {
    for (int i = 0; i < index->bucketCount; i++) {
        SpatialBucket* bucket = &index->buckets[i];
      
        // Quick bucket-level culling
        if (bucket->centerX + bucket->radius < bounds[0] ||
            bucket->centerX - bucket->radius > bounds[2] ||
            bucket->centerY + bucket->radius < bounds[3] ||
            bucket->centerY - bucket->radius > bounds[1]) {
            continue;  // Entire bucket out of view
        }
      
        // Draw items in visible bucket
        for (int j = 0; j < bucket->itemCount; j++) {
            DrawItem(bucket->items[j], bounds);
        }
    }
}
```

### Level-of-Detail Management

```c
// LOD system for map elements
typedef struct {
    float minZoom, maxZoom;
    int iconSize;
    int showLabels;
    int showDetails;
} LODLevel;

static const LODLevel gLODLevels[] = {
    // Far zoom: minimal details
    { 0.0f, 0.2f, 16, 0, 0 },
    // Medium zoom: basic info
    { 0.2f, 0.8f, 24, 1, 0 },
    // Close zoom: full details
    { 0.8f, 5.0f, 32, 1, 1 }
};

const LODLevel* GetLODForZoom(float zoomRatio) {
    for (int i = 0; i < sizeof(gLODLevels)/sizeof(LODLevel); i++) {
        if (zoomRatio >= gLODLevels[i].minZoom && 
            zoomRatio < gLODLevels[i].maxZoom) {
            return &gLODLevels[i];
        }
    }
    return &gLODLevels[0];  // Default to lowest detail
}

void DrawWithLOD(XPLMMapLayerID layer,
                float zoomRatio,
                float mapUnitsPerUIUnit,
                XPLMMapProjectionID projection) {
  
    const LODLevel* lod = GetLODForZoom(zoomRatio);
  
    // Adjust drawing based on LOD level
    float iconSize = lod->iconSize * mapUnitsPerUIUnit;
  
    for (int i = 0; i < gItemCount; i++) {
        Item* item = &gItems[i];
      
        // Skip items that are too small to see
        if (iconSize < 4.0f && item->importance < 0.8f) {
            continue;
        }
      
        // Draw with appropriate detail level
        if (lod->showDetails && item->hasDetailedIcon) {
            DrawDetailedIcon(layer, item, iconSize);
        } else {
            DrawSimpleIcon(layer, item, iconSize);
        }
    }
}
```

## Best Practices

### Error Handling

```c
// Robust layer implementation
int CreateMapLayerSafely(const char* mapID) {
    // Validate inputs
    if (!mapID) {
        XPLMDebugString("ERROR: NULL map identifier\n");
        return 0;
    }
  
    // Check if map exists
    if (!XPLMMapExists(mapID)) {
        char msg[128];
        sprintf(msg, "WARNING: Map %s does not exist yet\n", mapID);
        XPLMDebugString(msg);
        return 0;
    }
  
    // Initialize layer data
    LayerData* data = malloc(sizeof(LayerData));
    if (!data) {
        XPLMDebugString("ERROR: Failed to allocate layer data\n");
        return 0;
    }
  
    memset(data, 0, sizeof(LayerData));
    data->isValid = 1;
  
    // Set up layer creation parameters
    XPLMCreateMapLayer_t params;
    memset(&params, 0, sizeof(params));
  
    params.structSize = sizeof(XPLMCreateMapLayer_t);
    params.mapToCreateLayerIn = mapID;
    params.layerType = xplm_MapLayer_Markings;
    params.willBeDeletedCallback = SafeLayerWillBeDeleted;
    params.prepCacheCallback = SafeLayerPrepareCache;
    params.drawCallback = SafeLayerDraw;
    params.iconCallback = SafeLayerDrawIcons;
    params.labelCallback = SafeLayerDrawLabels;
    params.showUiToggle = 1;
    params.layerName = "Safe Layer";
    params.refcon = data;
  
    // Create layer with error checking
    XPLMMapLayerID layer = XPLMCreateMapLayer(&params);
    if (!layer) {
        XPLMDebugString("ERROR: XPLMCreateMapLayer failed\n");
        free(data);
        return 0;
    }
  
    data->layerID = layer;
    RegisterLayerGlobally(layer, data);
  
    XPLMDebugString("Successfully created map layer\n");
    return 1;
}

// Safe drawing with validation
void SafeLayerDrawIcons(XPLMMapLayerID layer,
                       const float* bounds,
                       float zoomRatio,
                       float mapUnitsPerUIUnit,
                       XPLMMapStyle mapStyle,
                       XPLMMapProjectionID projection,
                       void* refcon) {
  
    // Validate all parameters
    if (!layer || !bounds || !projection || !refcon) {
        XPLMDebugString("ERROR: Invalid parameters to icon callback\n");
        return;
    }
  
    LayerData* data = (LayerData*)refcon;
    if (!data->isValid) {
        XPLMDebugString("WARNING: Drawing on invalid layer\n");
        return;
    }
  
    // Validate bounds
    if (bounds[0] >= bounds[2] || bounds[3] >= bounds[1]) {
        XPLMDebugString("WARNING: Invalid map bounds\n");
        return;
    }
  
    // Validate zoom parameters
    if (zoomRatio <= 0.0f || mapUnitsPerUIUnit <= 0.0f) {
        XPLMDebugString("WARNING: Invalid zoom parameters\n");
        return;
    }
  
    // Perform actual drawing
    try {
        DrawIconsSafely(layer, bounds, zoomRatio, mapUnitsPerUIUnit, 
                       mapStyle, projection, data);
    } catch (...) {
        XPLMDebugString("ERROR: Exception in icon drawing\n");
    }
}
```
