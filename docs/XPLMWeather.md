# XPLMWeather API Documentation

## Overview

The XPLMWeather API provides access to X-Plane 12's enhanced weather system, enabling plugins to retrieve current weather conditions, access METAR reports, and obtain detailed atmospheric data at specific locations. This API is essential for plugins that need realistic weather information for flight planning, navigation, or atmospheric simulation.

## Key Features

- **Real-Time Weather Access**: Get current weather conditions at any location
- **METAR Report Retrieval**: Access downloaded METAR reports for airports
- **Comprehensive Weather Data**: Temperature, pressure, winds, precipitation, clouds
- **Multi-Layer Wind Information**: Up to 13 wind layers with detailed parameters
- **Cloud Layer Details**: Up to 3 cloud layers with coverage and altitude data
- **Water Conditions**: Wave height, length, direction, and speed information
- **Atmospheric Physics**: Thermal activity, turbulence, and visibility data

## Architecture

### Weather System Integration

X-Plane 12's weather system provides:

1. **Real Weather Mode**: Downloads actual METAR reports from online sources
2. **Custom Weather**: User-defined weather conditions
3. **Regional Weather**: Interpolated conditions across geographic regions
4. **Dynamic Weather**: Evolving conditions over time
5. **Layered Atmosphere**: Multiple wind and cloud layers

### Data Accuracy and Limitations

- **Geographic Range**: Weather data available within loaded scenery regions
- **Temporal Accuracy**: METAR reports may be outdated vs. current simulation conditions
- **Interpolation**: Weather interpolated between reporting stations
- **Performance Considerations**: Weather queries are expensive operations

### Coordinate Systems

- **Geographic Coordinates**: Latitude/longitude for location specification
- **MSL Altitudes**: All altitude references are Mean Sea Level
- **International Units**: Temperatures in Celsius, pressures in Pascals, speeds in m/s

## Data Types and Structures

### XPLMWeatherInfoWinds_t

```c
typedef struct {
    float alt_msl;          // Altitude MSL in meters
    float speed;            // Wind speed in meters/second
    float direction;        // Wind direction in degrees true
    float gust_speed;       // Gust speed in meters/second
    float shear;            // Wind shear arc in degrees
    float turbulence;       // Clear-air turbulence ratio
} XPLMWeatherInfoWinds_t;
```

Defines wind conditions at a specific altitude layer:

- **alt_msl**: Altitude of this wind layer above sea level
- **speed**: Sustained wind speed (0-100+ m/s typical range)
- **direction**: True wind direction (0° = from north, 90° = from east)
- **gust_speed**: Additional speed during gusts
- **shear**: Wind direction variation (±shear/2 degrees from base direction)
- **turbulence**: Turbulence intensity (0.0 = none, 1.0 = severe)

### XPLMWeatherInfoClouds_t

```c
typedef struct {
    float cloud_type;       // Cloud type as float enum
    float coverage;         // Coverage ratio (0.0 to 1.0)
    float alt_top;          // Cloud top altitude MSL in meters
    float alt_base;         // Cloud base altitude MSL in meters
} XPLMWeatherInfoClouds_t;
```

Defines cloud layer characteristics:

- **cloud_type**: Numeric cloud type identifier (implementation-specific)
- **coverage**: Cloud coverage fraction (0.0 = clear, 1.0 = overcast)
- **alt_top**: Altitude of cloud layer top
- **alt_base**: Altitude of cloud layer base

### XPLMWeatherInfo_t

```c
typedef struct {
    int   structSize;                          // Structure size for API compatibility
    float temperature_alt;                     // Temperature at altitude (°C)
    float dewpoint_alt;                        // Dewpoint at altitude (°C)
    float pressure_alt;                        // Pressure at altitude (Pa)
    float precip_rate_alt;                     // Precipitation rate at altitude
    float wind_dir_alt;                        // Wind direction at altitude (degrees)
    float wind_spd_alt;                        // Wind speed at altitude (m/s)
    float turbulence_alt;                      // Turbulence at altitude
    float wave_height;                         // Water wave height (meters)
    float wave_length;                         // Water wave length (meters)
    int   wave_dir;                            // Wave direction (degrees)
    float wave_speed;                          // Wave advance speed (m/s)
    float visibility;                          // Base visibility at 0 altitude (meters)
    float precip_rate;                         // Base precipitation rate
    float thermal_climb;                       // Thermal climb rate (m/s)
    float pressure_sl;                         // Sea level pressure (Pa)
    XPLMWeatherInfoWinds_t wind_layers[13];    // Wind layer definitions
    XPLMWeatherInfoClouds_t cloud_layers[3];   // Cloud layer definitions
} XPLMWeatherInfo_t;
```

Complete weather conditions at a specific location and altitude.

**Altitude-Specific Data** (at requested altitude):

- **temperature_alt**: Air temperature in Celsius
- **dewpoint_alt**: Dewpoint temperature in Celsius
- **pressure_alt**: Barometric pressure in Pascals
- **precip_rate_alt**: Precipitation intensity
- **wind_dir_alt**: Wind direction in degrees true
- **wind_spd_alt**: Wind speed in meters per second
- **turbulence_alt**: Turbulence intensity ratio

**Sea Level/Surface Data**:

- **visibility**: Horizontal visibility distance in meters
- **precip_rate**: Surface precipitation rate
- **thermal_climb**: Vertical air movement due to thermals
- **pressure_sl**: Sea level pressure in Pascals

**Water Surface Conditions**:

- **wave_height**: Significant wave height in meters
- **wave_length**: Dominant wave length in meters
- **wave_dir**: Direction waves are traveling toward (degrees)
- **wave_speed**: Speed of wave propagation in m/s

**Atmospheric Layers**:

- **wind_layers[13]**: Array of wind layer definitions (not all may be active)
- **cloud_layers[3]**: Array of cloud layer definitions (not all may be active)

### XPLMFixedString150_t

```c
typedef char XPLMFixedString150_t[150];
```

Fixed-size string buffer for METAR text data. Must be at least 150 characters to accommodate full METAR reports.

## Core Weather Functions (XPLM400+)

### XPLMGetMETARForAirport

```c
XPLM_API void XPLMGetMETARForAirport(
    const char              *airport_id,
    XPLMFixedString150_t    *outMETAR
);
```

Retrieves the last downloaded METAR report for specified airport.

**Parameters**:

- **airport_id**: ICAO airport code (e.g., "KJFK", "EGLL", "KORD")
- **outMETAR**: Buffer for METAR text (must be at least 150 characters)

**Important Limitations**:

- Only returns data in real-weather mode
- METAR may be significantly outdated vs. current sim conditions
- Returns empty string if no METAR available or not in real-weather mode
- Not intended for per-frame usage due to performance impact

**Example - Airport Weather Display**:

```c
void DisplayAirportWeather(const char *icaoCode) {
    XPLMFixedString150_t metarText;
  
    XPLMGetMETARForAirport(icaoCode, &metarText);
  
    if (strlen(metarText) > 0) {
        char display[256];
        snprintf(display, sizeof(display), "%s METAR: %s\n", icaoCode, metarText);
        XPLMDebugString(display);
      
        // Parse METAR for specific values
        ParsedMETAR parsed;
        if (ParseMETARString(metarText, &parsed)) {
            printf("Wind: %d°/%dkt, Visibility: %s, Clouds: %s\n",
                   parsed.windDir, parsed.windSpeed, 
                   parsed.visibility, parsed.cloudConditions);
        }
    } else {
        char msg[128];
        snprintf(msg, sizeof(msg), "No METAR available for %s\n", icaoCode);
        XPLMDebugString(msg);
    }
}

// Cache METARs to avoid repeated queries
typedef struct {
    char icao[8];
    XPLMFixedString150_t metar;
    float lastUpdate;
} METARCache;

static METARCache gMETARCache[50];
static int gCacheCount = 0;

const char* GetCachedMETAR(const char *icao) {
    float currentTime = XPLMGetElapsedTime();
  
    // Check cache first
    for (int i = 0; i < gCacheCount; i++) {
        if (strcmp(gMETARCache[i].icao, icao) == 0) {
            // Use cached data if less than 5 minutes old
            if (currentTime - gMETARCache[i].lastUpdate < 300.0f) {
                return gMETARCache[i].metar;
            }
          
            // Update existing cache entry
            XPLMGetMETARForAirport(icao, &gMETARCache[i].metar);
            gMETARCache[i].lastUpdate = currentTime;
            return gMETARCache[i].metar;
        }
    }
  
    // Add new cache entry
    if (gCacheCount < 50) {
        strcpy(gMETARCache[gCacheCount].icao, icao);
        XPLMGetMETARForAirport(icao, &gMETARCache[gCacheCount].metar);
        gMETARCache[gCacheCount].lastUpdate = currentTime;
        gCacheCount++;
        return gMETARCache[gCacheCount-1].metar;
    }
  
    return ""; // Cache full
}
```

### XPLMGetWeatherAtLocation

```c
XPLM_API int XPLMGetWeatherAtLocation(
    double              latitude,
    double              longitude,  
    double              altitude_m,
    XPLMWeatherInfo_t   *out_info
);
```

Retrieves comprehensive weather conditions at specified geographic location and altitude.

**Parameters**:

- **latitude**: Geographic latitude in degrees (-90 to +90)
- **longitude**: Geographic longitude in degrees (-180 to +180)
- **altitude_m**: Altitude MSL in meters for atmospheric data
- **out_info**: Weather information structure (must set structSize first)

**Returns**:

- **1**: Detailed weather found (airport-specific METAR data available)
- **0**: General weather returned (interpolated/default conditions)

**Performance Note**: Expensive operation, not suitable for per-frame usage

**Example - Location Weather Analysis**:

```c
typedef struct {
    double lat, lon;
    float altitude;
    XPLMWeatherInfo_t weather;
    int hasDetailedData;
    float lastUpdate;
} WeatherStation;

int UpdateWeatherStation(WeatherStation *station) {
    station->weather.structSize = sizeof(XPLMWeatherInfo_t);
  
    int hasDetailed = XPLMGetWeatherAtLocation(
        station->lat, 
        station->lon,
        station->altitude,
        &station->weather
    );
  
    station->hasDetailedData = hasDetailed;
    station->lastUpdate = XPLMGetElapsedTime();
  
    return 1;
}

void AnalyzeWeatherConditions(WeatherStation *station) {
    XPLMWeatherInfo_t *w = &station->weather;
  
    printf("Weather Analysis for %.4f, %.4f at %.0fm MSL:\n", 
           station->lat, station->lon, station->altitude);
  
    // Temperature conditions
    printf("Temperature: %.1f°C, Dewpoint: %.1f°C\n", 
           w->temperature_alt, w->dewpoint_alt);
  
    float relativeHumidity = CalculateRelativeHumidity(
        w->temperature_alt, w->dewpoint_alt);
    printf("Relative Humidity: %.1f%%\n", relativeHumidity * 100.0f);
  
    // Pressure and altitude
    printf("Pressure: %.1f hPa (%.2f inHg)\n", 
           w->pressure_alt / 100.0f, w->pressure_alt * 0.0002953f);
  
    float pressureAltitude = CalculatePressureAltitude(
        station->altitude, w->pressure_alt);
    printf("Pressure Altitude: %.0f ft\n", pressureAltitude * 3.28084f);
  
    // Wind conditions
    printf("Wind: %03.0f° at %.1f m/s (%.0f kt)\n",
           w->wind_dir_alt, w->wind_spd_alt, w->wind_spd_alt * 1.944f);
  
    // Visibility and precipitation
    printf("Visibility: %.0f m (%.1f sm)\n", 
           w->visibility, w->visibility * 0.000621371f);
  
    if (w->precip_rate > 0.01f) {
        printf("Precipitation: %.3f (intensity)\n", w->precip_rate);
    }
  
    // Turbulence and thermals
    if (w->turbulence_alt > 0.01f) {
        printf("Turbulence: %.3f intensity\n", w->turbulence_alt);
    }
  
    if (fabs(w->thermal_climb) > 0.1f) {
        printf("Thermal Activity: %.1f m/s %s\n", 
               fabs(w->thermal_climb), 
               w->thermal_climb > 0 ? "climb" : "sink");
    }
  
    // Cloud layers
    printf("Cloud Layers:\n");
    for (int i = 0; i < 3; i++) {
        if (w->cloud_layers[i].coverage > 0.01f) {
            printf("  Layer %d: %.0f%% coverage, %.0f-%.0f m MSL\n",
                   i+1, w->cloud_layers[i].coverage * 100.0f,
                   w->cloud_layers[i].alt_base, w->cloud_layers[i].alt_top);
        }
    }
  
    // Wind layers
    printf("Wind Layers:\n");
    for (int i = 0; i < 13; i++) {
        XPLMWeatherInfoWinds_t *wind = &w->wind_layers[i];
        if (wind->speed > 0.1f) {
            printf("  %.0f m: %03.0f°/%.1f m/s",
                   wind->alt_msl, wind->direction, wind->speed);
          
            if (wind->gust_speed > wind->speed + 0.5f) {
                printf(" G%.1f", wind->gust_speed);
            }
          
            if (wind->turbulence > 0.01f) {
                printf(" (turb %.3f)", wind->turbulence);
            }
          
            if (wind->shear > 5.0f) {
                printf(" (shear ±%.0f°)", wind->shear/2.0f);
            }
          
            printf("\n");
        }
    }
  
    // Water conditions (if applicable)
    if (w->wave_height > 0.1f) {
        printf("Sea State: %.1f m waves, %.0f m length, %03d° direction, %.1f m/s speed\n",
               w->wave_height, w->wave_length, w->wave_dir, w->wave_speed);
    }
}
```

**Example - Flight Planning Integration**:

```c
typedef struct {
    double lat, lon;
    float altitude;
    float distance; // Distance along route
    XPLMWeatherInfo_t weather;
} RouteWeatherPoint;

int AnalyzeRouteWeather(RouteWeatherPoint *route, int pointCount, 
                       char *summary, size_t summarySize) {
    float maxWindSpeed = 0.0f;
    float maxTurbulence = 0.0f; 
    float minVisibility = 99999.0f;
    float maxPrecip = 0.0f;
    int severeConditions = 0;
  
    for (int i = 0; i < pointCount; i++) {
        RouteWeatherPoint *point = &route[i];
      
        // Get weather at this route point
        point->weather.structSize = sizeof(XPLMWeatherInfo_t);
        XPLMGetWeatherAtLocation(
            point->lat, point->lon, point->altitude, &point->weather
        );
      
        // Analyze conditions
        if (point->weather.wind_spd_alt > maxWindSpeed) {
            maxWindSpeed = point->weather.wind_spd_alt;
        }
      
        if (point->weather.turbulence_alt > maxTurbulence) {
            maxTurbulence = point->weather.turbulence_alt;
        }
      
        if (point->weather.visibility < minVisibility) {
            minVisibility = point->weather.visibility;
        }
      
        if (point->weather.precip_rate > maxPrecip) {
            maxPrecip = point->weather.precip_rate;
        }
      
        // Check for severe conditions
        if (point->weather.wind_spd_alt > 25.0f ||          // >50kt winds
            point->weather.turbulence_alt > 0.3f ||         // Significant turbulence
            point->weather.visibility < 1600.0f ||          // <1 mile visibility
            point->weather.precip_rate > 0.5f) {            // Heavy precipitation
            severeConditions++;
        }
    }
  
    // Generate summary
    snprintf(summary, summarySize,
        "Route Weather Summary:\n"
        "Max Wind: %.0f kt, Max Turbulence: %.3f\n"
        "Min Visibility: %.1f sm, Max Precip: %.3f\n"
        "Severe Conditions: %d of %d points\n",
        maxWindSpeed * 1.944f, maxTurbulence,
        minVisibility * 0.000621371f, maxPrecip,
        severeConditions, pointCount
    );
  
    return severeConditions; // Return count of severe weather points
}
```

## Implementation Guidelines

### Weather Data Caching Strategy

```c
typedef struct {
    double lat, lon;
    float altitude;
    XPLMWeatherInfo_t weather;
    float lastUpdate;
    int isValid;
} WeatherCache;

static WeatherCache gWeatherCache[20];
static int gCacheIndex = 0;

int GetCachedWeather(double lat, double lon, float alt, 
                     XPLMWeatherInfo_t *weather) {
    float currentTime = XPLMGetElapsedTime();
    const float cacheTimeout = 120.0f; // 2 minutes
    const float locationThreshold = 0.01f; // ~1km at equator
    const float altitudeThreshold = 100.0f; // 100m
  
    // Search for existing cache entry
    for (int i = 0; i < 20; i++) {
        WeatherCache *cache = &gWeatherCache[i];
      
        if (!cache->isValid) continue;
      
        // Check if cache is still valid
        if (currentTime - cache->lastUpdate > cacheTimeout) {
            cache->isValid = 0;
            continue;
        }
      
        // Check location proximity
        double dLat = fabs(lat - cache->lat);
        double dLon = fabs(lon - cache->lon);
        float dAlt = fabs(alt - cache->altitude);
      
        if (dLat < locationThreshold && 
            dLon < locationThreshold && 
            dAlt < altitudeThreshold) {
            // Cache hit
            *weather = cache->weather;
            return 1;
        }
    }
  
    // Cache miss - get new data
    weather->structSize = sizeof(XPLMWeatherInfo_t);
    int result = XPLMGetWeatherAtLocation(lat, lon, alt, weather);
  
    // Store in cache
    WeatherCache *cache = &gWeatherCache[gCacheIndex];
    cache->lat = lat;
    cache->lon = lon; 
    cache->altitude = alt;
    cache->weather = *weather;
    cache->lastUpdate = currentTime;
    cache->isValid = 1;
  
    gCacheIndex = (gCacheIndex + 1) % 20; // Round-robin replacement
  
    return result;
}
```

### Weather Monitoring System

```c
typedef struct {
    double lat, lon;
    float altitude;
    XPLMWeatherInfo_t lastWeather;
    XPLMWeatherInfo_t currentWeather;
    float updateInterval;
    float lastUpdate;
    void (*changeCallback)(void *ref, const XPLMWeatherInfo_t *old, const XPLMWeatherInfo_t *new);
    void *callbackRef;
} WeatherMonitor;

void UpdateWeatherMonitor(WeatherMonitor *monitor) {
    float currentTime = XPLMGetElapsedTime();
  
    if (currentTime - monitor->lastUpdate < monitor->updateInterval) {
        return; // Too soon to update
    }
  
    // Save previous weather
    monitor->lastWeather = monitor->currentWeather;
  
    // Get current weather
    monitor->currentWeather.structSize = sizeof(XPLMWeatherInfo_t);
    XPLMGetWeatherAtLocation(
        monitor->lat, monitor->lon, monitor->altitude,
        &monitor->currentWeather
    );
  
    monitor->lastUpdate = currentTime;
  
    // Check for significant changes
    if (HasSignificantWeatherChange(&monitor->lastWeather, &monitor->currentWeather)) {
        if (monitor->changeCallback) {
            monitor->changeCallback(monitor->callbackRef, 
                                   &monitor->lastWeather, 
                                   &monitor->currentWeather);
        }
    }
}

int HasSignificantWeatherChange(const XPLMWeatherInfo_t *old, 
                               const XPLMWeatherInfo_t *new) {
    // Temperature change > 2°C
    if (fabs(new->temperature_alt - old->temperature_alt) > 2.0f) return 1;
  
    // Wind speed change > 5 m/s
    if (fabs(new->wind_spd_alt - old->wind_spd_alt) > 5.0f) return 1;
  
    // Wind direction change > 30°
    float windDirChange = fabs(new->wind_dir_alt - old->wind_dir_alt);
    if (windDirChange > 180.0f) windDirChange = 360.0f - windDirChange;
    if (windDirChange > 30.0f) return 1;
  
    // Visibility change > 1000m
    if (fabs(new->visibility - old->visibility) > 1000.0f) return 1;
  
    // Pressure change > 200 Pa (~2 hPa)
    if (fabs(new->pressure_alt - old->pressure_alt) > 200.0f) return 1;
  
    // Turbulence increase > 0.1
    if ((new->turbulence_alt - old->turbulence_alt) > 0.1f) return 1;
  
    return 0; // No significant change
}

void WeatherChangeAlert(void *ref, const XPLMWeatherInfo_t *old, 
                       const XPLMWeatherInfo_t *new) {
    const char *location = (const char*)ref;
  
    char alert[512];
    snprintf(alert, sizeof(alert), 
        "Weather Alert at %s:\n"
        "Temperature: %.1f°C -> %.1f°C (Δ%.1f°C)\n"
        "Wind: %03.0f°/%.0fkt -> %03.0f°/%.0fkt\n"
        "Visibility: %.1fsm -> %.1fsm\n"
        "Turbulence: %.3f -> %.3f\n",
        location,
        old->temperature_alt, new->temperature_alt, 
        new->temperature_alt - old->temperature_alt,
        old->wind_dir_alt, old->wind_spd_alt * 1.944f,
        new->wind_dir_alt, new->wind_spd_alt * 1.944f,
        old->visibility * 0.000621371f, new->visibility * 0.000621371f,
        old->turbulence_alt, new->turbulence_alt
    );
  
    XPLMDebugString(alert);
}
```

### Weather-Based Calculations

```c
// Standard atmosphere calculations
float CalculatePressureAltitude(float trueAltitude, float actualPressure) {
    const float seaLevelPressure = 101325.0f; // Pa
    const float tempLapseRate = 0.0065f; // K/m
    const float standardTemp = 288.15f; // K at sea level
  
    float pressureRatio = actualPressure / seaLevelPressure;
    float pressureAlt = trueAltitude + 
        (standardTemp / tempLapseRate) * (1.0f - powf(pressureRatio, 0.190263f));
  
    return pressureAlt;
}

float CalculateDensityAltitude(float pressureAltitude, float temperature) {
    const float standardTemp = 15.0f; // °C
    const float tempLapseRate = 1.98f; // °C per 1000 ft
  
    float tempDeviation = temperature - (standardTemp - 
        (pressureAltitude * 3.28084f / 1000.0f) * tempLapseRate);
  
    float densityAlt = pressureAltitude + (tempDeviation * 120.0f / 3.28084f);
  
    return densityAlt;
}

float CalculateRelativeHumidity(float temperature, float dewpoint) {
    // Magnus formula approximation
    float a = 17.27f;
    float b = 237.7f;
  
    float alpha_temp = a * temperature / (b + temperature);
    float alpha_dew = a * dewpoint / (b + dewpoint);
  
    return expf(alpha_dew - alpha_temp);
}

// Wind triangle calculations
void CalculateWindTriangle(float trueAirspeed, float heading,
                          float windSpeed, float windDirection,
                          float *groundSpeed, float *track) {
    float headingRad = heading * M_PI / 180.0f;
    float windDirRad = windDirection * M_PI / 180.0f;
  
    // Aircraft velocity components
    float tasNorth = trueAirspeed * cosf(headingRad);
    float tasEast = trueAirspeed * sinf(headingRad);
  
    // Wind velocity components (FROM direction)
    float windNorth = -windSpeed * cosf(windDirRad);
    float windEast = -windSpeed * sinf(windDirRad);
  
    // Ground velocity components
    float gsNorth = tasNorth + windNorth;
    float gsEast = tasEast + windEast;
  
    // Calculate ground speed and track
    *groundSpeed = sqrtf(gsNorth * gsNorth + gsEast * gsEast);
    *track = atan2f(gsEast, gsNorth) * 180.0f / M_PI;
  
    if (*track < 0.0f) *track += 360.0f;
}

// Crosswind component calculation
float CalculateCrosswindComponent(float windSpeed, float windDirection,
                                 float runwayHeading) {
    float windAngle = windDirection - runwayHeading;
  
    // Normalize to ±180°
    while (windAngle > 180.0f) windAngle -= 360.0f;
    while (windAngle < -180.0f) windAngle += 360.0f;
  
    return windSpeed * sinf(windAngle * M_PI / 180.0f);
}

float CalculateHeadwindComponent(float windSpeed, float windDirection,
                                float runwayHeading) {
    float windAngle = windDirection - runwayHeading;
  
    // Normalize to ±180°
    while (windAngle > 180.0f) windAngle -= 360.0f;
    while (windAngle < -180.0f) windAngle += 360.0f;
  
    return windSpeed * cosf(windAngle * M_PI / 180.0f);
}
```

### Weather Layer Analysis

```c
typedef struct {
    float altitude;
    float windSpeed;
    float windDirection;
    float turbulence;
    float temperature;
    float pressure;
} FlightLevel;

void AnalyzeFlightLevels(const XPLMWeatherInfo_t *weather, 
                        FlightLevel *levels, int levelCount) {
    for (int i = 0; i < levelCount; i++) {
        FlightLevel *level = &levels[i];
      
        // Interpolate conditions at this altitude
        InterpolateAtAltitude(weather, level->altitude, level);
      
        // Calculate additional parameters
        level->pressure = CalculatePressureAtAltitude(
            level->altitude, weather->pressure_sl);
      
        level->temperature = CalculateTemperatureAtAltitude(
            level->altitude, weather->temperature_alt);
    }
}

void InterpolateAtAltitude(const XPLMWeatherInfo_t *weather, 
                          float targetAltitude, FlightLevel *level) {
    level->altitude = targetAltitude;
  
    // Find bracketing wind layers
    int lowerLayer = -1, upperLayer = -1;
  
    for (int i = 0; i < 13; i++) {
        if (weather->wind_layers[i].speed < 0.1f) continue; // Invalid layer
      
        if (weather->wind_layers[i].alt_msl <= targetAltitude) {
            if (lowerLayer == -1 || 
                weather->wind_layers[i].alt_msl > weather->wind_layers[lowerLayer].alt_msl) {
                lowerLayer = i;
            }
        }
      
        if (weather->wind_layers[i].alt_msl >= targetAltitude) {
            if (upperLayer == -1 || 
                weather->wind_layers[i].alt_msl < weather->wind_layers[upperLayer].alt_msl) {
                upperLayer = i;
            }
        }
    }
  
    // Interpolate wind conditions
    if (lowerLayer != -1 && upperLayer != -1 && lowerLayer != upperLayer) {
        // Linear interpolation between layers
        float lowerAlt = weather->wind_layers[lowerLayer].alt_msl;
        float upperAlt = weather->wind_layers[upperLayer].alt_msl;
        float ratio = (targetAltitude - lowerAlt) / (upperAlt - lowerAlt);
      
        level->windSpeed = weather->wind_layers[lowerLayer].speed + 
            ratio * (weather->wind_layers[upperLayer].speed - 
                    weather->wind_layers[lowerLayer].speed);
      
        // Handle wind direction interpolation (circular values)
        level->windDirection = InterpolateWindDirection(
            weather->wind_layers[lowerLayer].direction,
            weather->wind_layers[upperLayer].direction, ratio);
      
        level->turbulence = weather->wind_layers[lowerLayer].turbulence + 
            ratio * (weather->wind_layers[upperLayer].turbulence - 
                    weather->wind_layers[lowerLayer].turbulence);
    } 
    else if (lowerLayer != -1) {
        // Use lower layer data
        level->windSpeed = weather->wind_layers[lowerLayer].speed;
        level->windDirection = weather->wind_layers[lowerLayer].direction;
        level->turbulence = weather->wind_layers[lowerLayer].turbulence;
    }
    else if (upperLayer != -1) {
        // Use upper layer data
        level->windSpeed = weather->wind_layers[upperLayer].speed;
        level->windDirection = weather->wind_layers[upperLayer].direction;
        level->turbulence = weather->wind_layers[upperLayer].turbulence;
    }
    else {
        // No wind data available
        level->windSpeed = 0.0f;
        level->windDirection = 0.0f;
        level->turbulence = 0.0f;
    }
}

float InterpolateWindDirection(float dir1, float dir2, float ratio) {
    // Handle circular interpolation for wind direction
    float diff = dir2 - dir1;
  
    // Ensure we take the shortest angular path
    if (diff > 180.0f) diff -= 360.0f;
    if (diff < -180.0f) diff += 360.0f;
  
    float result = dir1 + ratio * diff;
  
    // Normalize to 0-360°
    while (result < 0.0f) result += 360.0f;
    while (result >= 360.0f) result -= 360.0f;
  
    return result;
}
```

## Common Use Cases

### 1. Flight Planning Weather Brief

```c
typedef struct {
    char departureICAO[8];
    char arrivalICAO[8];
    double routePoints[10][2]; // lat/lon pairs
    int routePointCount;
    char briefing[4096];
} WeatherBrief;

int GenerateWeatherBrief(WeatherBrief *brief) {
    char tempBuffer[512];
    strcpy(brief->briefing, "WEATHER BRIEFING\n================\n\n");
  
    // Departure airport METAR
    XPLMFixedString150_t depMETAR;
    XPLMGetMETARForAirport(brief->departureICAO, &depMETAR);
    if (strlen(depMETAR) > 0) {
        snprintf(tempBuffer, sizeof(tempBuffer), 
                "DEPARTURE (%s):\n%s\n\n", brief->departureICAO, depMETAR);
        strcat(brief->briefing, tempBuffer);
    }
  
    // Arrival airport METAR
    XPLMFixedString150_t arrMETAR;
    XPLMGetMETARForAirport(brief->arrivalICAO, &arrMETAR);
    if (strlen(arrMETAR) > 0) {
        snprintf(tempBuffer, sizeof(tempBuffer), 
                "ARRIVAL (%s):\n%s\n\n", brief->arrivalICAO, arrMETAR);
        strcat(brief->briefing, tempBuffer);
    }
  
    // Route weather analysis
    strcat(brief->briefing, "ROUTE ANALYSIS:\n");
  
    for (int i = 0; i < brief->routePointCount; i++) {
        XPLMWeatherInfo_t weather;
        weather.structSize = sizeof(weather);
      
        int hasDetailed = XPLMGetWeatherAtLocation(
            brief->routePoints[i][0], // latitude
            brief->routePoints[i][1], // longitude
            10000.0f,                 // FL100
            &weather
        );
      
        snprintf(tempBuffer, sizeof(tempBuffer),
                "Point %d (%.2f, %.2f): Wind %03.0f°/%.0fkt, "
                "Temp %.0f°C, Turb %.3f\n",
                i+1, brief->routePoints[i][0], brief->routePoints[i][1],
                weather.wind_dir_alt, weather.wind_spd_alt * 1.944f,
                weather.temperature_alt, weather.turbulence_alt);
      
        strcat(brief->briefing, tempBuffer);
    }
  
    return 1;
}
```

### 2. Real-Time Weather Display

```c
typedef struct {
    float x, y, width, height;
    XPLMWeatherInfo_t weather;
    float lastUpdate;
} WeatherDisplay;

void DrawWeatherDisplay(WeatherDisplay *display) {
    float currentTime = XPLMGetElapsedTime();
  
    // Update weather every 30 seconds
    if (currentTime - display->lastUpdate > 30.0f) {
        // Get aircraft position
        float lat = XPLMGetDataf(latitudeRef);
        float lon = XPLMGetDataf(longitudeRef);
        float alt = XPLMGetDataf(altitudeRef);
      
        display->weather.structSize = sizeof(display->weather);
        XPLMGetWeatherAtLocation(lat, lon, alt, &display->weather);
        display->lastUpdate = currentTime;
    }
  
    // Draw weather information
    char line[256];
    float lineHeight = 20.0f;
    float y = display->y + display->height - lineHeight;
  
    // Wind
    snprintf(line, sizeof(line), "Wind: %03.0f° %.0fkt", 
            display->weather.wind_dir_alt, 
            display->weather.wind_spd_alt * 1.944f);
    XPLMDrawString(RGB(1,1,1), display->x, y, line, NULL, xplmFont_Proportional);
    y -= lineHeight;
  
    // Temperature
    snprintf(line, sizeof(line), "OAT: %.0f°C", display->weather.temperature_alt);
    XPLMDrawString(RGB(1,1,1), display->x, y, line, NULL, xplmFont_Proportional);
    y -= lineHeight;
  
    // Pressure
    snprintf(line, sizeof(line), "Baro: %.2f inHg", 
            display->weather.pressure_alt * 0.0002953f);
    XPLMDrawString(RGB(1,1,1), display->x, y, line, NULL, xplmFont_Proportional);
    y -= lineHeight;
  
    // Visibility
    snprintf(line, sizeof(line), "Vis: %.1f sm", 
            display->weather.visibility * 0.000621371f);
    XPLMDrawString(RGB(1,1,1), display->x, y, line, NULL, xplmFont_Proportional);
    y -= lineHeight;
  
    // Turbulence warning
    if (display->weather.turbulence_alt > 0.1f) {
        snprintf(line, sizeof(line), "TURBULENCE: %.3f", display->weather.turbulence_alt);
        XPLMDrawString(RGB(1,1,0), display->x, y, line, NULL, xplmFont_Proportional);
    }
}
```

### 3. Weather-Based Autopilot Adjustments

```c
void UpdateAutopilotForWeather(void) {
    static float lastWeatherUpdate = 0.0f;
    float currentTime = XPLMGetElapsedTime();
  
    if (currentTime - lastWeatherUpdate < 5.0f) return; // Update every 5 seconds
  
    // Get current position and altitude
    float lat = XPLMGetDataf(latitudeRef);
    float lon = XPLMGetDataf(longitudeRef);
    float alt = XPLMGetDataf(altitudeRef);
  
    XPLMWeatherInfo_t weather;
    weather.structSize = sizeof(weather);
    XPLMGetWeatherAtLocation(lat, lon, alt, &weather);
  
    // Adjust autopilot for wind conditions
    float crosswind = CalculateCrosswindComponent(
        weather.wind_spd_alt,
        weather.wind_dir_alt,
        XPLMGetDataf(headingRef)
    );
  
    // Apply wind correction to autopilot
    if (fabs(crosswind) > 2.5f) { // > 5kt crosswind
        float windCorrection = atan2f(crosswind, 
            XPLMGetDataf(airspeedRef)) * 180.0f / M_PI;
      
        float targetHeading = XPLMGetDataf(autopilotHeadingRef);
        float correctedHeading = targetHeading + windCorrection;
      
        XPLMSetDataf(autopilotHeadingBugRef, correctedHeading);
    }
  
    // Adjust altitude for turbulence
    if (weather.turbulence_alt > 0.2f) {
        // Reduce autopilot sensitivity in turbulence
        XPLMSetDataf(autopilotVerticalSpeedRef, 
                    XPLMGetDataf(autopilotVerticalSpeedRef) * 0.8f);
    }
  
    // Adjust airspeed for strong winds
    float headwind = CalculateHeadwindComponent(
        weather.wind_spd_alt,
        weather.wind_dir_alt,
        XPLMGetDataf(headingRef)
    );
  
    if (headwind < -10.0f) { // Strong tailwind
        // Reduce target airspeed slightly
        float currentTarget = XPLMGetDataf(autopilotAirspeedRef);
        XPLMSetDataf(autopilotAirspeedRef, currentTarget - 5.0f);
    }
  
    lastWeatherUpdate = currentTime;
}
```

## Performance Considerations

### Update Frequency Guidelines

```c
// Different update frequencies for different uses
typedef enum {
    WEATHER_UPDATE_CRITICAL = 5,    // 5 seconds - flight critical systems
    WEATHER_UPDATE_NORMAL = 30,     // 30 seconds - normal operations  
    WEATHER_UPDATE_BACKGROUND = 120, // 2 minutes - background monitoring
    WEATHER_UPDATE_ARCHIVE = 600    // 10 minutes - data logging
} WeatherUpdateFrequency;

void ScheduleWeatherUpdates(void) {
    // Critical flight systems
    XPLMRegisterFlightLoopCallback(CriticalWeatherUpdate, 
                                  WEATHER_UPDATE_CRITICAL, NULL);
  
    // Display and user interface
    XPLMRegisterFlightLoopCallback(DisplayWeatherUpdate, 
                                  WEATHER_UPDATE_NORMAL, NULL);
  
    // Data logging and analysis
    XPLMRegisterFlightLoopCallback(ArchiveWeatherUpdate, 
                                  WEATHER_UPDATE_ARCHIVE, NULL);
}
```

### Memory Usage Optimization

```c
// Use smaller structures for cached data
typedef struct {
    float temperature;
    float windSpeed;
    float windDirection;
    float turbulence;
    float visibility;
    float pressure;
} CompactWeather;

void CompressWeatherData(const XPLMWeatherInfo_t *full, CompactWeather *compact) {
    compact->temperature = full->temperature_alt;
    compact->windSpeed = full->wind_spd_alt;
    compact->windDirection = full->wind_dir_alt;
    compact->turbulence = full->turbulence_alt;
    compact->visibility = full->visibility;
    compact->pressure = full->pressure_alt;
}
```

## Integration with Other Systems

### Dataref Integration

```c
// Custom datarefs for weather information
static XPLMDataRef gWeatherWindSpeedRef = NULL;
static XPLMDataRef gWeatherWindDirRef = NULL;
static XPLMDataRef gWeatherTurbulenceRef = NULL;

static float gWeatherData[3] = {0.0f, 0.0f, 0.0f}; // speed, dir, turbulence

int CreateWeatherDatarefs(void) {
    gWeatherWindSpeedRef = XPLMRegisterDataAccessor(
        "myplugin/weather/wind_speed_ms",
        xplmType_Float, 0,
        NULL, NULL,
        GetWeatherWindSpeed, NULL,
        NULL, NULL, NULL, NULL, NULL, NULL,
        NULL, NULL
    );
  
    gWeatherWindDirRef = XPLMRegisterDataAccessor(
        "myplugin/weather/wind_direction_deg",
        xplmType_Float, 0,
        NULL, NULL,
        GetWeatherWindDir, NULL,
        NULL, NULL, NULL, NULL, NULL, NULL,
        NULL, NULL
    );
  
    gWeatherTurbulenceRef = XPLMRegisterDataAccessor(
        "myplugin/weather/turbulence",
        xplmType_Float, 0,
        NULL, NULL,
        GetWeatherTurbulence, NULL,
        NULL, NULL, NULL, NULL, NULL, NULL,
        NULL, NULL
    );
  
    return (gWeatherWindSpeedRef && gWeatherWindDirRef && gWeatherTurbulenceRef);
}

float GetWeatherWindSpeed(void *refcon) {
    return gWeatherData[0];
}

float GetWeatherWindDir(void *refcon) {
    return gWeatherData[1];
}

float GetWeatherTurbulence(void *refcon) {
    return gWeatherData[2];
}
```

### Command Integration

```c
static XPLMCommandRef gUpdateWeatherCmd = NULL;
static XPLMCommandRef gToggleWeatherDisplayCmd = NULL;

void CreateWeatherCommands(void) {
    gUpdateWeatherCmd = XPLMCreateCommand(
        "myplugin/weather/update",
        "Force weather data update"
    );
  
    gToggleWeatherDisplayCmd = XPLMCreateCommand(
        "myplugin/weather/toggle_display", 
        "Toggle weather display window"
    );
  
    XPLMRegisterCommandHandler(gUpdateWeatherCmd, WeatherCommandHandler, 1, NULL);
    XPLMRegisterCommandHandler(gToggleWeatherDisplayCmd, WeatherCommandHandler, 1, NULL);
}

int WeatherCommandHandler(XPLMCommandRef cmd, XPLMCommandPhase phase, void *refcon) {
    if (phase == xplm_CommandBegin) {
        if (cmd == gUpdateWeatherCmd) {
            ForceWeatherUpdate();
        }
        else if (cmd == gToggleWeatherDisplayCmd) {
            ToggleWeatherDisplay();
        }
    }
    return 1;
}
```

## Error Handling and Debugging

```c
void ValidateWeatherData(const XPLMWeatherInfo_t *weather, const char *source) {
    char warning[512];
  
    // Sanity check temperature values
    if (weather->temperature_alt < -80.0f || weather->temperature_alt > 60.0f) {
        snprintf(warning, sizeof(warning), 
                "Warning: Extreme temperature %.1f°C from %s\n",
                weather->temperature_alt, source);
        XPLMDebugString(warning);
    }
  
    // Check wind speed range
    if (weather->wind_spd_alt > 100.0f) {
        snprintf(warning, sizeof(warning),
                "Warning: Extreme wind speed %.1f m/s from %s\n", 
                weather->wind_spd_alt, source);
        XPLMDebugString(warning);
    }
  
    // Validate pressure values
    if (weather->pressure_alt < 50000.0f || weather->pressure_alt > 110000.0f) {
        snprintf(warning, sizeof(warning),
                "Warning: Extreme pressure %.0f Pa from %s\n",
                weather->pressure_alt, source);
        XPLMDebugString(warning);
    }
  
    // Check for valid visibility
    if (weather->visibility < 0.0f || weather->visibility > 50000.0f) {
        snprintf(warning, sizeof(warning),
                "Warning: Invalid visibility %.0f m from %s\n",
                weather->visibility, source);
        XPLMDebugString(warning);
    }
}

void LogWeatherDebugInfo(const XPLMWeatherInfo_t *weather, double lat, double lon) {
    char debugInfo[2048];
  
    snprintf(debugInfo, sizeof(debugInfo),
            "Weather Debug Info (%.4f, %.4f):\n"
            "Structure Size: %d (expected %zu)\n"
            "Temperature: %.2f°C, Dewpoint: %.2f°C\n"
            "Pressure: %.1f hPa, Sea Level: %.1f hPa\n"
            "Wind: %03.0f°/%.1f m/s, Turbulence: %.4f\n"
            "Visibility: %.0f m, Precipitation: %.4f\n"
            "Waves: H=%.1fm L=%.0fm D=%03d° S=%.1fm/s\n"
            "Thermals: %.2f m/s\n",
            lat, lon,
            weather->structSize, sizeof(XPLMWeatherInfo_t),
            weather->temperature_alt, weather->dewpoint_alt,
            weather->pressure_alt / 100.0f, weather->pressure_sl / 100.0f,
            weather->wind_dir_alt, weather->wind_spd_alt, weather->turbulence_alt,
            weather->visibility, weather->precip_rate,
            weather->wave_height, weather->wave_length, 
            weather->wave_dir, weather->wave_speed,
            weather->thermal_climb
    );
  
    XPLMDebugString(debugInfo);
  
    // Log wind layers
    for (int i = 0; i < 13; i++) {
        if (weather->wind_layers[i].speed > 0.1f) {
            snprintf(debugInfo, sizeof(debugInfo),
                    "Wind Layer %d: %.0fm MSL, %03.0f°/%.1f m/s, "
                    "Gust %.1f, Shear %.1f°, Turb %.4f\n",
                    i, weather->wind_layers[i].alt_msl,
                    weather->wind_layers[i].direction, weather->wind_layers[i].speed,
                    weather->wind_layers[i].gust_speed, weather->wind_layers[i].shear,
                    weather->wind_layers[i].turbulence);
            XPLMDebugString(debugInfo);
        }
    }
  
    // Log cloud layers  
    for (int i = 0; i < 3; i++) {
        if (weather->cloud_layers[i].coverage > 0.01f) {
            snprintf(debugInfo, sizeof(debugInfo),
                    "Cloud Layer %d: Type %.0f, Coverage %.0f%%, "
                    "Base %.0fm, Top %.0fm\n",
                    i, weather->cloud_layers[i].cloud_type,
                    weather->cloud_layers[i].coverage * 100.0f,
                    weather->cloud_layers[i].alt_base, weather->cloud_layers[i].alt_top);
            XPLMDebugString(debugInfo);
        }
    }
}
```

## Version Compatibility

The XPLMWeather API is only available in XPLM400+:

```c
int InitializeWeatherSystem(void) {
#if defined(XPLM400)
    XPLMDebugString("Weather API available - initializing\n");
  
    // Test weather API availability
    XPLMFixedString150_t testMETAR;
    XPLMGetMETARForAirport("KJFK", &testMETAR);
  
    return 1; // Success
#else
    XPLMDebugString("Weather API requires XPLM400+\n");
    return 0; // Weather API not available
#endif
}
```

## Summary

The XPLMWeather API provides comprehensive access to X-Plane 12's advanced weather system:

1. **Real-Time Data**: Access to current weather conditions with detailed atmospheric information
2. **METAR Integration**: Retrieval of actual weather reports from airports worldwide
3. **Multi-Layer Atmosphere**: Detailed wind and cloud layer information for realistic flight planning
4. **Performance Optimized**: Designed for occasional use with appropriate caching strategies
5. **Flight Integration**: Essential data for realistic flight simulation and navigation

Key implementation principles:

- **Performance Awareness**: Weather queries are expensive - use caching and appropriate update frequencies
- **Data Validation**: Always validate weather data for realistic ranges and handle edge cases
- **User Experience**: Present weather information in familiar aviation formats and units
- **Integration Strategy**: Coordinate weather data with flight planning, autopilot, and display systems
- **Error Handling**: Graceful degradation when weather data is unavailable or invalid
