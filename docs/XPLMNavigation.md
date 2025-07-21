# XPLMNavigation API Documentation

## Overview

The XPLMNavigation API provides comprehensive access to X-Plane's navigation database and flight management systems. This API allows plugins to search navigation aids, read GPS data, and program Flight Management Systems (FMS) with waypoints, approaches, and flight plans.

## Key Features

- **Navigation Database Access**: Search and query airports, VORs, NDBs, ILS, fixes, and other navigation aids
- **Flight Management System (FMS) Control**: Program and manage flight plans with multiple device support
- **GPS Integration**: Read GPS destination and navigation data
- **Real-time Navigation Data**: Access to live navigation information stored in X-Plane's memory
- **Multi-Device Support**: Handle pilot and co-pilot navigation devices separately (XPLM410+)

## Architecture

### Navigation Database Structure

X-Plane stores all navigation information in RAM as an array-like structure. Navigation aids of the same type are grouped together but the database is not necessarily densely populated. The API provides iterator-based access for efficient traversal.

### FMS System

The FMS operates on an array of entries (waypoints) with a maximum of 100 entries. Each entry contains:

- Navigation aid reference or lat/lon coordinates
- Altitude information
- Entry type identification

Modern X-Plane supports multiple flight plans per device:

- **GPS Devices**: Enroute and Approach flight plans
- **FMS Devices**: Active and Temporary flight plans

## Data Types

### XPLMNavType

```c
enum {
    xplm_Nav_Unknown       = 0,
    xplm_Nav_Airport       = 1,
    xplm_Nav_NDB           = 2,
    xplm_Nav_VOR           = 4,
    xplm_Nav_ILS           = 8,
    xplm_Nav_Localizer     = 16,
    xplm_Nav_GlideSlope    = 32,
    xplm_Nav_OuterMarker   = 64,
    xplm_Nav_MiddleMarker  = 128,
    xplm_Nav_InnerMarker   = 256,
    xplm_Nav_Fix           = 512,
    xplm_Nav_DME           = 1024,
    xplm_Nav_LatLon        = 2048,
    xplm_Nav_TACAN         = 4096
};
typedef int XPLMNavType;
```

Navigation aid types using bitfield values that can be combined with OR operations for multi-type searches.

**Special Notes:**

- `xplm_Nav_LatLon`: Used only for FMS entries, not database searches
- Values can be combined: `xplm_Nav_VOR | xplm_Nav_NDB` for multiple types

### XPLMNavRef

```c
typedef int XPLMNavRef;
#define XPLM_NAV_NOT_FOUND (-1)
```

Iterator/reference into the navigation database. Represents a specific navigation aid.

### XPLMNavFlightPlan (XPLM410+)

```c
enum {
    xplm_Fpl_Pilot_Primary      = 0,
    xplm_Fpl_CoPilot_Primary    = 1,
    xplm_Fpl_Pilot_Approach     = 2,
    xplm_Fpl_CoPilot_Approach   = 3,
    xplm_Fpl_Pilot_Temporary    = 4,
    xplm_Fpl_CoPilot_Temporary  = 5
};
typedef int XPLMNavFlightPlan;
```

Identifies specific flight plan instances in modern navigation systems.

## Navigation Database Functions

### Database Traversal

#### XPLMGetFirstNavAid

```c
XPLM_API XPLMNavRef XPLMGetFirstNavAid(void);
```

Returns the first navigation aid in the database for complete traversal.

**Returns**: First navaid reference or `XPLM_NAV_NOT_FOUND` if database is empty

**Use Cases:**

- Complete database enumeration
- Building custom navigation databases
- Statistical analysis of available navigation aids

#### XPLMGetNextNavAid

```c
XPLM_API XPLMNavRef XPLMGetNextNavAid(XPLMNavRef inNavAidRef);
```

Returns the next navigation aid in sequence.

**Parameters:**

- `inNavAidRef`: Current navigation aid reference

**Returns**: Next navaid reference or `XPLM_NAV_NOT_FOUND` at end

**Example - Complete Database Traversal:**

```c
XPLMNavRef navRef = XPLMGetFirstNavAid();
int totalNavaids = 0;

while (navRef != XPLM_NAV_NOT_FOUND) {
    XPLMNavType type;
    char id[32], name[256];
  
    XPLMGetNavAidInfo(navRef, &type, NULL, NULL, NULL, NULL, NULL, 
                      id, name, NULL);
  
    printf("Navaid: %s (%s) Type: %d\n", name, id, type);
    totalNavaids++;
  
    navRef = XPLMGetNextNavAid(navRef);
}

printf("Total navigation aids: %d\n", totalNavaids);
```

### Type-Specific Searches

#### XPLMFindFirstNavAidOfType

```c
XPLM_API XPLMNavRef XPLMFindFirstNavAidOfType(XPLMNavType inType);
```

Finds the first navigation aid of a specific type.

**Parameters:**

- `inType`: Single navigation aid type (not combined types)

**Returns**: First navaid of type or `XPLM_NAV_NOT_FOUND`

#### XPLMFindLastNavAidOfType

```c
XPLM_API XPLMNavRef XPLMFindLastNavAidOfType(XPLMNavType inType);
```

Finds the last navigation aid of a specific type.

**Use Cases:**

- Reverse iteration through specific navaid types
- Finding range boundaries for type-specific processing

### Advanced Navigation Search

#### XPLMFindNavAid

```c
XPLM_API XPLMNavRef XPLMFindNavAid(
    const char * inNameFragment,    // Name search string (can be NULL)
    const char * inIDFragment,      // ID search string (can be NULL)
    float *      inLat,             // Latitude for proximity search (can be NULL)
    float *      inLon,             // Longitude for proximity search (can be NULL)
    int *        inFrequency,       // Frequency filter (can be NULL)
    XPLMNavType  inType             // Navigation aid type(s)
);
```

Powerful multi-criteria search function for finding specific navigation aids.

**Search Logic:**

- If lat/lon provided: Returns nearest match
- If lat/lon NULL: Returns last match found
- All non-NULL criteria must match
- Multiple types can be combined with OR

**Parameters:**

- `inNameFragment`: Partial name match (case-insensitive substring)
- `inIDFragment`: Partial ID match (case-insensitive substring)
- `inLat/inLon`: Search center coordinates for proximity search
- `inFrequency`: Exact frequency match (nav.dat format: NDB exact, others ×100)
- `inType`: Navigation aid types to include (can be combined)

**Common Search Patterns:**

```c
// Find nearest airport to position
float lat = 42.3601, lon = -71.0589;  // Boston area
XPLMNavRef airport = XPLMFindNavAid(NULL, NULL, &lat, &lon, NULL, xplm_Nav_Airport);

// Find VOR by ID
XPLMNavRef bostonVOR = XPLMFindNavAid(NULL, "BOS", NULL, NULL, NULL, xplm_Nav_VOR);

// Find ILS by frequency
int ilsFreq = 11110;  // 111.10 MHz × 100
XPLMNavRef ils = XPLMFindNavAid(NULL, NULL, NULL, NULL, &ilsFreq, xplm_Nav_ILS);

// Find airports with "Chicago" in name
XPLMNavRef chicagoAirport = XPLMFindNavAid("Chicago", NULL, NULL, NULL, NULL, xplm_Nav_Airport);

// Find any radio navaid on frequency
XPLMNavRef radioNav = XPLMFindNavAid(NULL, NULL, NULL, NULL, &frequency, 
                                     xplm_Nav_VOR | xplm_Nav_NDB | xplm_Nav_TACAN);
```

### Navigation Aid Information

#### XPLMGetNavAidInfo

```c
XPLM_API void XPLMGetNavAidInfo(
    XPLMNavRef   inRef,           // Navigation aid reference
    XPLMNavType *outType,         // Navigation aid type
    float *      outLatitude,     // Latitude in degrees
    float *      outLongitude,    // Longitude in degrees
    float *      outHeight,       // Height in meters MSL
    int *        outFrequency,    // Frequency (nav.dat format)
    float *      outHeading,      // Magnetic heading (localizers, ILS)
    char *       outID,           // Navigation aid identifier
    char *       outName,         // Full name
    char *       outReg           // Region flag (single byte: 0/1)
);
```

Retrieves comprehensive information about a navigation aid.

**Buffer Requirements:**

- `outID`: Minimum 6 characters (recommended 32)
- `outName`: Minimum 41 characters (recommended 256)
- `outReg`: Single byte value (0 or 1), not a C string

**Frequency Format (nav.dat convention):**

- **NDB**: Exact frequency in kHz
- **All Others**: Frequency × 100 (e.g., 111.10 MHz = 11110)

**Example - Detailed Navaid Analysis:**

```c
void AnalyzeNavaid(XPLMNavRef navRef) {
    XPLMNavType type;
    float lat, lon, height, heading;
    int frequency;
    char id[32], name[256];
    char region;
  
    XPLMGetNavAidInfo(navRef, &type, &lat, &lon, &height, &frequency, 
                      &heading, id, name, &region);
  
    printf("=== Navigation Aid Analysis ===\n");
    printf("ID: %s\n", id);
    printf("Name: %s\n", name);
    printf("Type: %s\n", GetNavTypeName(type));
    printf("Position: %.6f, %.6f\n", lat, lon);
    printf("Elevation: %.1f m MSL\n", height);
  
    if (frequency > 0) {
        if (type == xplm_Nav_NDB) {
            printf("Frequency: %d kHz\n", frequency);
        } else {
            printf("Frequency: %.2f MHz\n", frequency / 100.0f);
        }
    }
  
    if (type == xplm_Nav_ILS || type == xplm_Nav_Localizer) {
        printf("Heading: %.1f°\n", heading);
    }
  
    printf("In Local Region: %s\n", region ? "Yes" : "No");
}
```

## Legacy FMS Functions

### FMS Entry Management

#### XPLMCountFMSEntries

```c
XPLM_API int XPLMCountFMSEntries(void);
```

Returns the total number of entries in the legacy FMS.

#### XPLMGetDisplayedFMSEntry / XPLMSetDisplayedFMSEntry

```c
XPLM_API int  XPLMGetDisplayedFMSEntry(void);
XPLM_API void XPLMSetDisplayedFMSEntry(int inIndex);
```

Gets/sets which FMS entry is currently displayed to the pilot.

#### XPLMGetDestinationFMSEntry / XPLMSetDestinationFMSEntry

```c
XPLM_API int  XPLMGetDestinationFMSEntry(void);
XPLM_API void XPLMSetDestinationFMSEntry(int inIndex);
```

Gets/sets which FMS entry the aircraft is currently flying toward.

**Note**: The track is from entry (n-1) to entry (n).

### FMS Entry Operations

#### XPLMGetFMSEntryInfo

```c
XPLM_API void XPLMGetFMSEntryInfo(
    int          inIndex,      // Entry index (0-based)
    XPLMNavType *outType,      // Entry type
    char *       outID,        // Entry identifier
    XPLMNavRef * outRef,       // Navigation database reference
    int *        outAltitude,  // Altitude constraint
    float *      outLat,       // Latitude (for lat/lon entries)
    float *      outLon        // Longitude (for lat/lon entries)
);
```

Retrieves information about an FMS entry.

**Important Notes:**

- `outRef` may be `XPLM_NAV_NOT_FOUND` temporarily after flight plan changes
- Database lookup is asynchronous and may take up to 1 second
- Always initialize `outRef` to `XPLM_NAV_NOT_FOUND` before calling (X-Plane bug workaround)

#### XPLMSetFMSEntryInfo

```c
XPLM_API void XPLMSetFMSEntryInfo(
    int        inIndex,     // Entry index
    XPLMNavRef inRef,       // Navigation aid reference
    int        inAltitude   // Altitude constraint
);
```

Sets an FMS entry to a specific navigation aid.

**Supported Types:** Airports, fixes, VORs, NDBs

#### XPLMSetFMSEntryLatLon

```c
XPLM_API void XPLMSetFMSEntryLatLon(
    int   inIndex,     // Entry index
    float inLat,       // Latitude
    float inLon,       // Longitude
    int   inAltitude   // Altitude constraint
);
```

Sets an FMS entry to a specific lat/lon coordinate.

#### XPLMClearFMSEntry

```c
XPLM_API void XPLMClearFMSEntry(int inIndex);
```

Clears an FMS entry, potentially shortening the flight plan.

## Modern FMS Functions (XPLM410+)

### Multi-Device Flight Plan Management

The modern API supports multiple navigation devices and flight plan types:

#### XPLMCountFMSFlightPlanEntries

```c
XPLM_API int XPLMCountFMSFlightPlanEntries(XPLMNavFlightPlan inFlightPlan);
```

Returns entry count for a specific flight plan.

#### Display and Destination Management

```c
XPLM_API int  XPLMGetDisplayedFMSFlightPlanEntry(XPLMNavFlightPlan inFlightPlan);
XPLM_API void XPLMSetDisplayedFMSFlightPlanEntry(XPLMNavFlightPlan inFlightPlan, int inIndex);

XPLM_API int  XPLMGetDestinationFMSFlightPlanEntry(XPLMNavFlightPlan inFlightPlan);
XPLM_API void XPLMSetDestinationFMSFlightPlanEntry(XPLMNavFlightPlan inFlightPlan, int inIndex);
```

### Direct-To Navigation

#### XPLMSetDirectToFMSFlightPlanEntry

```c
XPLM_API void XPLMSetDirectToFMSFlightPlanEntry(
    XPLMNavFlightPlan inFlightPlan,
    int               inIndex
);
```

Sets direct-to navigation, creating a track from current aircraft position directly to the specified waypoint.

### Flight Plan Entry Operations

#### XPLMGetFMSFlightPlanEntryInfo

```c
XPLM_API void XPLMGetFMSFlightPlanEntryInfo(
    XPLMNavFlightPlan inFlightPlan,
    int               inIndex,
    XPLMNavType *     outType,
    char *            outID,
    XPLMNavRef *      outRef,
    int *             outAltitude,
    float *           outLat,
    float *           outLon
);
```

Same as legacy version but for specific flight plans.

#### XPLMSetFMSFlightPlanEntryInfo

```c
XPLM_API void XPLMSetFMSFlightPlanEntryInfo(
    XPLMNavFlightPlan inFlightPlan,
    int               inIndex,
    XPLMNavRef        inRef,
    int               inAltitude
);
```

Sets flight plan entry to navigation aid. Supports VORs, NDBs, TACANs, airports, and fixes.

#### XPLMSetFMSFlightPlanEntryLatLon

```c
XPLM_API void XPLMSetFMSFlightPlanEntryLatLon(
    XPLMNavFlightPlan inFlightPlan,
    int               inIndex,
    float             inLat,
    float             inLon,
    int               inAltitude
);
```

#### XPLMSetFMSFlightPlanEntryLatLonWithId

```c
XPLM_API void XPLMSetFMSFlightPlanEntryLatLonWithId(
    XPLMNavFlightPlan inFlightPlan,
    int               inIndex,
    float             inLat,
    float             inLon,
    int               inAltitude,
    const char *      inId,
    unsigned int      inIdLength
);
```

Sets lat/lon waypoint with custom display identifier.

#### XPLMClearFMSFlightPlanEntry

```c
XPLM_API void XPLMClearFMSFlightPlanEntry(
    XPLMNavFlightPlan inFlightPlan,
    int               inIndex
);
```

### Flight Plan File Operations

#### XPLMLoadFMSFlightPlan

```c
XPLM_API void XPLMLoadFMSFlightPlan(
    int          inDevice,     // Device index (0=pilot, 1=copilot)
    const char * inBuffer,     // Flight plan data
    unsigned int inBufferLen   // Buffer length
);
```

Loads X-Plane 11+ formatted flight plan including procedures.

#### XPLMSaveFMSFlightPlan

```c
XPLM_API unsigned int XPLMSaveFMSFlightPlan(
    int          inDevice,      // Device index
    char *       inBuffer,      // Output buffer
    unsigned int inBufferLen    // Buffer size
);
```

Saves flight plan to buffer in X-Plane 11+ format.

**Returns**: Required buffer size (including null terminator)

**Buffer Handling:**

- If buffer too small: Data truncated, not null-terminated
- Return value > buffer size: Data incomplete

```c
// Determine required buffer size
unsigned int requiredSize = XPLMSaveFMSFlightPlan(0, NULL, 0);

// Allocate and save
char* buffer = malloc(requiredSize);
unsigned int actualSize = XPLMSaveFMSFlightPlan(0, buffer, requiredSize);

if (actualSize <= requiredSize) {
    // Success - buffer contains null-terminated flight plan
    ProcessFlightPlan(buffer);
}
free(buffer);
```

## GPS Functions

### GPS Destination Access

#### XPLMGetGPSDestinationType

```c
XPLM_API XPLMNavType XPLMGetGPSDestinationType(void);
```

Returns the type of currently selected GPS destination.

#### XPLMGetGPSDestination

```c
XPLM_API XPLMNavRef XPLMGetGPSDestination(void);
```

Returns reference to current GPS destination.

**Example - GPS Monitoring:**

```c
void MonitorGPSDestination() {
    XPLMNavType destType = XPLMGetGPSDestinationType();
    XPLMNavRef destRef = XPLMGetGPSDestination();
  
    if (destRef != XPLM_NAV_NOT_FOUND) {
        char id[32], name[256];
        float lat, lon;
      
        XPLMGetNavAidInfo(destRef, NULL, &lat, &lon, NULL, NULL, NULL,
                          id, name, NULL);
      
        printf("GPS Destination: %s (%s) at %.6f, %.6f\n", 
               name, id, lat, lon);
    }
}
```

## Implementation Patterns

### Navigation Database Search System

```c
typedef struct {
    XPLMNavRef ref;
    float distance;
    char id[16];
    char name[128];
} NavAidResult;

// Find all airports within radius
int FindNearbyAirports(float centerLat, float centerLon, float radiusNM, 
                      NavAidResult* results, int maxResults) {
    int count = 0;
    XPLMNavRef navRef = XPLMFindFirstNavAidOfType(xplm_Nav_Airport);
  
    while (navRef != XPLM_NAV_NOT_FOUND && count < maxResults) {
        float lat, lon;
        char id[16], name[128];
      
        XPLMGetNavAidInfo(navRef, NULL, &lat, &lon, NULL, NULL, NULL,
                          id, name, NULL);
      
        float distance = CalculateDistance(centerLat, centerLon, lat, lon);
      
        if (distance <= radiusNM) {
            results[count].ref = navRef;
            results[count].distance = distance;
            strcpy(results[count].id, id);
            strcpy(results[count].name, name);
            count++;
        }
      
        navRef = XPLMGetNextNavAid(navRef);
    }
  
    // Sort by distance
    qsort(results, count, sizeof(NavAidResult), CompareByDistance);
  
    return count;
}
```

### FMS Programming System

```c
typedef struct {
    XPLMNavRef navRef;
    float lat, lon;
    int altitude;
    char id[16];
    XPLMNavType type;
} Waypoint;

// Program complete flight plan
void ProgramFlightPlan(XPLMNavFlightPlan flightPlan, Waypoint* waypoints, int waypointCount) {
    // Clear existing flight plan
    int existingCount = XPLMCountFMSFlightPlanEntries(flightPlan);
    for (int i = existingCount - 1; i >= 0; i--) {
        XPLMClearFMSFlightPlanEntry(flightPlan, i);
    }
  
    // Program new waypoints
    for (int i = 0; i < waypointCount; i++) {
        if (waypoints[i].type == xplm_Nav_LatLon) {
            XPLMSetFMSFlightPlanEntryLatLonWithId(
                flightPlan, i,
                waypoints[i].lat, waypoints[i].lon, waypoints[i].altitude,
                waypoints[i].id, strlen(waypoints[i].id)
            );
        } else {
            XPLMSetFMSFlightPlanEntryInfo(
                flightPlan, i,
                waypoints[i].navRef, waypoints[i].altitude
            );
        }
    }
  
    // Set destination to first waypoint
    if (waypointCount > 0) {
        XPLMSetDestinationFMSFlightPlanEntry(flightPlan, 0);
    }
}
```

### Navigation Aid Frequency Scanner

```c
typedef struct {
    int frequency;
    XPLMNavType type;
    XPLMNavRef ref;
    char id[16];
    float distance;
} FrequencyInfo;

// Scan for navigation aids on specific frequency
int ScanFrequency(int frequency, float aircraftLat, float aircraftLon,
                 FrequencyInfo* results, int maxResults) {
    int count = 0;
  
    // Search radio navigation types
    XPLMNavType searchTypes = xplm_Nav_VOR | xplm_Nav_NDB | xplm_Nav_TACAN |
                             xplm_Nav_ILS | xplm_Nav_Localizer;
  
    XPLMNavRef navRef = XPLMFindNavAid(NULL, NULL, NULL, NULL, 
                                       &frequency, searchTypes);
  
    // Find all matches (not just first)
    XPLMNavRef firstMatch = navRef;
    XPLMNavRef currentRef = XPLMGetFirstNavAid();
  
    while (currentRef != XPLM_NAV_NOT_FOUND && count < maxResults) {
        XPLMNavType type;
        int navFreq;
        float lat, lon;
        char id[16];
      
        XPLMGetNavAidInfo(currentRef, &type, &lat, &lon, NULL, 
                          &navFreq, NULL, id, NULL, NULL);
      
        if (navFreq == frequency && (type & searchTypes)) {
            results[count].frequency = frequency;
            results[count].type = type;
            results[count].ref = currentRef;
            strcpy(results[count].id, id);
            results[count].distance = CalculateDistance(aircraftLat, aircraftLon, lat, lon);
            count++;
        }
      
        currentRef = XPLMGetNextNavAid(currentRef);
    }
  
    return count;
}
```

## Best Practices

### Performance Optimization

1. **Cache Navigation References**: Store frequently used XPLMNavRef values
2. **Batch Operations**: Group multiple FMS operations together
3. **Proximity Searches**: Use lat/lon parameters for efficient searches
4. **Type Filtering**: Use specific nav types rather than scanning all types

### Error Handling

```c
// Always check for valid references
XPLMNavRef FindSafeNavaid(const char* id, XPLMNavType type) {
    if (!id || strlen(id) == 0) {
        return XPLM_NAV_NOT_FOUND;
    }
  
    XPLMNavRef ref = XPLMFindNavAid(NULL, id, NULL, NULL, NULL, type);
  
    if (ref != XPLM_NAV_NOT_FOUND) {
        // Verify the navaid is actually valid
        XPLMNavType actualType;
        XPLMGetNavAidInfo(ref, &actualType, NULL, NULL, NULL, NULL, NULL,
                          NULL, NULL, NULL);
      
        if (actualType & type) {
            return ref;
        }
    }
  
    return XPLM_NAV_NOT_FOUND;
}
```

### Multi-Device Support

```c
// Handle both pilot and copilot devices
void ProgramBothDevices(Waypoint* waypoints, int count) {
    XPLMNavFlightPlan pilotPlan = xplm_Fpl_Pilot_Primary;
    XPLMNavFlightPlan copilotPlan = xplm_Fpl_CoPilot_Primary;
  
    // Check if both devices exist
    int pilotEntries = XPLMCountFMSFlightPlanEntries(pilotPlan);
    int copilotEntries = XPLMCountFMSFlightPlanEntries(copilotPlan);
  
    if (pilotEntries >= 0) {
        ProgramFlightPlan(pilotPlan, waypoints, count);
    }
  
    if (copilotEntries >= 0) {
        ProgramFlightPlan(copilotPlan, waypoints, count);
    }
}
```

## Version Compatibility

- **Base Navigation Database**: All SDK versions
- **Legacy FMS Functions**: XPLM200+
- **Modern Flight Plan API**: XPLM410+ (X-Plane 12.1.0+)
- **TACAN Support**: XPLM410+ for FMS programming
- **Flight Plan File Operations**: XPLM410+

## Integration with Other Systems

### Dataref Integration

```c
// Monitor FMS state changes via datarefs
static XPLMDataRef gFMSEntryCountRef = NULL;
static XPLMDataRef gFMSDestinationRef = NULL;

void InitializeFMSMonitoring() {
    gFMSEntryCountRef = XPLMFindDataRef("sim/cockpit2/radios/actuators/gps_flight_plan_size");
    gFMSDestinationRef = XPLMFindDataRef("sim/cockpit2/radios/actuators/gps_flight_plan_next_index");
}

void MonitorFMSChanges() {
    static int lastEntryCount = -1;
    static int lastDestination = -1;
  
    int currentEntryCount = XPLMGetDatai(gFMSEntryCountRef);
    int currentDestination = XPLMGetDatai(gFMSDestinationRef);
  
    if (currentEntryCount != lastEntryCount) {
        OnFlightPlanChanged(currentEntryCount);
        lastEntryCount = currentEntryCount;
    }
  
    if (currentDestination != lastDestination) {
        OnDestinationChanged(currentDestination);
        lastDestination = currentDestination;
    }
}
```

### Command Integration

```c
// Create commands for navigation functions
static XPLMCommandRef gDirectToCommand = NULL;
static XPLMCommandRef gNextWaypointCommand = NULL;

void CreateNavigationCommands() {
    gDirectToCommand = XPLMCreateCommand("myplugin/direct_to", "Direct To Nearest Airport");
    gNextWaypointCommand = XPLMCreateCommand("myplugin/next_waypoint", "Skip to Next Waypoint");
  
    XPLMRegisterCommandHandler(gDirectToCommand, DirectToHandler, 1, NULL);
    XPLMRegisterCommandHandler(gNextWaypointCommand, NextWaypointHandler, 1, NULL);
}

int DirectToHandler(XPLMCommandRef cmd, XPLMCommandPhase phase, void* refcon) {
    if (phase == xplm_CommandBegin) {
        // Get aircraft position
        float lat = XPLMGetDataf(XPLMFindDataRef("sim/flightmodel/position/latitude"));
        float lon = XPLMGetDataf(XPLMFindDataRef("sim/flightmodel/position/longitude"));
      
        // Find nearest airport
        XPLMNavRef airport = XPLMFindNavAid(NULL, NULL, &lat, &lon, NULL, xplm_Nav_Airport);
      
        if (airport != XPLM_NAV_NOT_FOUND) {
            // Set direct-to
            XPLMSetDirectToFMSFlightPlanEntry(xplm_Fpl_Pilot_Primary, 0);
            XPLMSetFMSFlightPlanEntryInfo(xplm_Fpl_Pilot_Primary, 0, airport, 3000);
        }
    }
    return 1;
}
```

## Common Use Cases

### 1. Nearest Airport Finder

```c
XPLMNavRef FindNearestAirport(float lat, float lon) {
    return XPLMFindNavAid(NULL, NULL, &lat, &lon, NULL, xplm_Nav_Airport);
}
```

### 2. VOR Radial Navigation

```c
typedef struct {
    XPLMNavRef vor;
    float radial;
    float distance;
} VORRadial;

VORRadial CalculateVORPosition(const char* vorID, float radial, float distance) {
    VORRadial result = {XPLM_NAV_NOT_FOUND, 0, 0};
  
    XPLMNavRef vor = XPLMFindNavAid(NULL, vorID, NULL, NULL, NULL, xplm_Nav_VOR);
    if (vor != XPLM_NAV_NOT_FOUND) {
        result.vor = vor;
        result.radial = radial;
        result.distance = distance;
    }
  
    return result;
}
```

### 3. ILS Approach Setup

```c
bool SetupILSApproach(const char* airportID, const char* runway) {
    // Find airport
    XPLMNavRef airport = XPLMFindNavAid(NULL, airportID, NULL, NULL, NULL, xplm_Nav_Airport);
    if (airport == XPLM_NAV_NOT_FOUND) return false;
  
    // Find ILS for runway
    char ilsID[16];
    snprintf(ilsID, sizeof(ilsID), "I%s", runway);  // ILS naming convention
  
    XPLMNavRef ils = XPLMFindNavAid(NULL, ilsID, NULL, NULL, NULL, xplm_Nav_ILS);
    if (ils == XPLM_NAV_NOT_FOUND) return false;
  
    // Program approach
    XPLMNavFlightPlan approachPlan = xplm_Fpl_Pilot_Approach;
    XPLMSetFMSFlightPlanEntryInfo(approachPlan, 0, ils, 0);
  
    return true;
}
```
