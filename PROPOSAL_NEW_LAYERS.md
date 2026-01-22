# Proposal: Expanding Map View with Environmental & Socioeconomic Data

## Overview

This proposal outlines how to extend the maptoposter application to include three new data layers:
1. **Income Level** - Socioeconomic data visualization
2. **Pollution Levels** - Air quality and environmental contamination
3. **Superfund Site Locations** - EPA hazardous waste sites

Each layer follows the existing architecture pattern: **fetch data → create GeoDataFrame → render with theme colors**.

---

## 1. Income Level Layer

### Data Source Options

| Source | Coverage | API | Format | Cost |
|--------|----------|-----|--------|------|
| **US Census Bureau** | USA only | Free API | GeoJSON | Free |
| **OpenStreetMap Admin Boundaries** | Global | OSMnx | Polygons | Free |
| **World Bank Data** | Global (country-level) | REST API | JSON | Free |
| **Eurostat** | Europe | REST API | GeoJSON | Free |

### Recommended Approach: US Census Bureau API

**Why:** Provides census tract-level income data with precise boundaries, ideal for city-scale maps.

### Implementation

#### Step 1: Add Census Data Fetcher

```python
import requests

def fetch_income_data(lat, lon, dist):
    """
    Fetch median household income by census tract from US Census Bureau.
    Returns GeoDataFrame with income classifications.
    """
    # Get census tracts that intersect our bounding box
    bbox = ox.utils_geo.bbox_from_point((lat, lon), dist=dist)

    # Census TIGERweb API for tract boundaries
    tiger_url = "https://tigerweb.geo.census.gov/arcgis/rest/services/TIGERweb/tigerWMS_Census2020/MapServer/8/query"
    params = {
        "geometry": f"{bbox[1]},{bbox[0]},{bbox[3]},{bbox[2]}",
        "geometryType": "esriGeometryEnvelope",
        "spatialRel": "esriSpatialRelIntersects",
        "outFields": "GEOID,NAME",
        "returnGeometry": "true",
        "f": "geojson"
    }

    response = requests.get(tiger_url, params=params)
    tracts_gdf = gpd.GeoDataFrame.from_features(response.json()["features"])

    # Fetch income data from Census API (requires API key)
    census_url = "https://api.census.gov/data/2022/acs/acs5"
    # B19013_001E = Median household income
    income_params = {
        "get": "NAME,B19013_001E",
        "for": "tract:*",
        "in": f"state:*&county:*",
        "key": CENSUS_API_KEY
    }

    # Join income data to tract boundaries
    # Classify into quintiles for choropleth rendering
    return classify_income_levels(tracts_gdf, income_data)

def classify_income_levels(gdf, income_data):
    """Classify income into 5 levels for visualization."""
    gdf['income_level'] = pd.qcut(gdf['median_income'], q=5, labels=[1, 2, 3, 4, 5])
    return gdf
```

#### Step 2: Add Theme Properties

Add to each theme JSON file in `/themes/`:

```json
{
  "income_level_1": "#2C1810",
  "income_level_2": "#5D4037",
  "income_level_3": "#8D6E63",
  "income_level_4": "#BCAAA4",
  "income_level_5": "#EFEBE9",
  "income_opacity": 0.4
}
```

#### Step 3: Render Layer

```python
# In create_poster(), after parks rendering:
if income_data is not None and not income_data.empty and show_income:
    for level in range(1, 6):
        level_data = income_data[income_data['income_level'] == level]
        if not level_data.empty:
            level_data.plot(
                ax=ax,
                color=THEME[f'income_level_{level}'],
                alpha=THEME.get('income_opacity', 0.4),
                linewidth=0,
                zorder=0.5  # Below roads, above background
            )
```

#### CLI Integration

```python
parser.add_argument('--show-income', action='store_true',
                    help='Overlay income level choropleth (US only)')
parser.add_argument('--census-api-key', type=str,
                    help='US Census Bureau API key')
```

---

## 2. Pollution Levels Layer

### Data Source Options

| Source | Coverage | Data Type | Update Freq | Cost |
|--------|----------|-----------|-------------|------|
| **OpenWeather Air Pollution API** | Global | AQI + pollutants | Real-time | Free tier |
| **EPA AirNow** | USA | AQI | Hourly | Free |
| **AQICN (World AQI)** | Global | AQI | Real-time | Free tier |
| **Copernicus CAMS** | Global | Forecast models | Daily | Free |

### Recommended Approach: OpenWeather Air Pollution API

**Why:** Global coverage, detailed pollutant breakdown, generous free tier.

### Implementation

#### Step 1: Add Pollution Data Fetcher

```python
def fetch_pollution_data(lat, lon, dist, api_key):
    """
    Fetch air quality data and create interpolated grid.
    Returns GeoDataFrame with pollution levels.
    """
    import numpy as np
    from scipy.interpolate import griddata

    # Create grid of sample points within bounding box
    bbox = ox.utils_geo.bbox_from_point((lat, lon), dist=dist)
    grid_points = create_sample_grid(bbox, resolution=20)  # 20x20 grid

    pollution_values = []
    for point in grid_points:
        # OpenWeather Air Pollution API
        url = f"http://api.openweathermap.org/data/2.5/air_pollution"
        params = {
            "lat": point[0],
            "lon": point[1],
            "appid": api_key
        }
        response = requests.get(url, params=params)
        data = response.json()

        # AQI scale: 1 (Good) to 5 (Very Poor)
        aqi = data['list'][0]['main']['aqi']
        pollution_values.append(aqi)
        time.sleep(0.1)  # Rate limiting

    # Interpolate to create smooth contours
    return create_pollution_contours(grid_points, pollution_values, bbox)

def create_pollution_contours(points, values, bbox):
    """Create GeoDataFrame of pollution level polygons."""
    from shapely.geometry import Polygon
    import matplotlib.pyplot as plt

    # Create dense interpolation grid
    xi = np.linspace(bbox[1], bbox[3], 100)
    yi = np.linspace(bbox[0], bbox[2], 100)
    zi = griddata(points, values, (xi[None,:], yi[:,None]), method='cubic')

    # Generate contour polygons for each AQI level
    contours = plt.contourf(xi, yi, zi, levels=[1, 2, 3, 4, 5])

    # Convert matplotlib contours to shapely polygons
    polygons = []
    levels = []
    for i, collection in enumerate(contours.collections):
        for path in collection.get_paths():
            poly = Polygon(path.vertices)
            polygons.append(poly)
            levels.append(i + 1)

    return gpd.GeoDataFrame({'geometry': polygons, 'aqi_level': levels})
```

#### Step 2: Add Theme Properties

```json
{
  "pollution_good": "#00E400",
  "pollution_moderate": "#FFFF00",
  "pollution_unhealthy_sensitive": "#FF7E00",
  "pollution_unhealthy": "#FF0000",
  "pollution_very_unhealthy": "#8F3F97",
  "pollution_hazardous": "#7E0023",
  "pollution_opacity": 0.3
}
```

#### Step 3: Render Layer

```python
# AQI color mapping
AQI_COLORS = {
    1: 'pollution_good',
    2: 'pollution_moderate',
    3: 'pollution_unhealthy_sensitive',
    4: 'pollution_unhealthy',
    5: 'pollution_very_unhealthy'
}

if pollution_data is not None and not pollution_data.empty and show_pollution:
    for level, theme_key in AQI_COLORS.items():
        level_data = pollution_data[pollution_data['aqi_level'] == level]
        if not level_data.empty:
            level_data.plot(
                ax=ax,
                color=THEME.get(theme_key, '#CCCCCC'),
                alpha=THEME.get('pollution_opacity', 0.3),
                linewidth=0,
                zorder=0.6  # Above income, below water
            )
```

#### CLI Integration

```python
parser.add_argument('--show-pollution', action='store_true',
                    help='Overlay air quality heatmap')
parser.add_argument('--openweather-api-key', type=str,
                    help='OpenWeather API key for pollution data')
```

---

## 3. Superfund Site Locations Layer

### Data Source

| Source | Coverage | Format | Update Freq |
|--------|----------|--------|-------------|
| **EPA Superfund Sites (NPL)** | USA | GeoJSON/CSV | Quarterly |
| **EPA FRS (Facility Registry)** | USA | API | Daily |
| **EPA ECHO Database** | USA | API | Monthly |

### Recommended Approach: EPA Envirofacts API + FRS

**Why:** Official EPA data, includes exact coordinates, site status, and contamination details.

### Implementation

#### Step 1: Add Superfund Data Fetcher

```python
def fetch_superfund_sites(lat, lon, dist):
    """
    Fetch EPA Superfund (NPL) sites within the map bounds.
    Returns GeoDataFrame with site locations and details.
    """
    bbox = ox.utils_geo.bbox_from_point((lat, lon), dist=dist)

    # EPA Envirofacts REST API
    # SEMS = Superfund Enterprise Management System
    url = "https://enviro.epa.gov/enviro/efservice/SEMS_ACTIVE_SITES/JSON"

    # Alternative: Direct NPL GeoJSON (more reliable)
    npl_url = "https://services.arcgis.com/cJ9YHowT8TU7DUyn/arcgis/rest/services/Superfund_National_Priorities_List_(NPL)_Sites/FeatureServer/0/query"

    params = {
        "where": f"LATITUDE >= {bbox[0]} AND LATITUDE <= {bbox[2]} AND LONGITUDE >= {bbox[1]} AND LONGITUDE <= {bbox[3]}",
        "outFields": "SITE_NAME,CITY,STATE,STATUS,NPL_STATUS,LATITUDE,LONGITUDE",
        "returnGeometry": "true",
        "f": "geojson"
    }

    response = requests.get(npl_url, params=params)

    if response.status_code == 200:
        data = response.json()
        if data.get('features'):
            gdf = gpd.GeoDataFrame.from_features(data['features'])
            gdf.set_crs(epsg=4326, inplace=True)
            return gdf

    return None

def create_superfund_markers(gdf, buffer_meters=500):
    """
    Create circular buffer zones around Superfund sites.
    Optionally differentiate by status (active, construction, deleted).
    """
    # Project to UTM for accurate buffer calculation
    gdf_utm = gdf.to_crs(gdf.estimate_utm_crs())
    gdf_utm['geometry'] = gdf_utm.geometry.buffer(buffer_meters)

    # Project back to WGS84
    return gdf_utm.to_crs(epsg=4326)
```

#### Step 2: Add Theme Properties

```json
{
  "superfund_active": "#FF0000",
  "superfund_construction": "#FFA500",
  "superfund_deleted": "#808080",
  "superfund_proposed": "#FFFF00",
  "superfund_opacity": 0.6,
  "superfund_marker_size": 100,
  "superfund_border_color": "#000000",
  "superfund_border_width": 1.5
}
```

#### Step 3: Render Layer

```python
# Superfund status to theme key mapping
SUPERFUND_STATUS_COLORS = {
    'Final NPL': 'superfund_active',
    'Active': 'superfund_active',
    'Construction Complete': 'superfund_construction',
    'Deleted': 'superfund_deleted',
    'Proposed': 'superfund_proposed'
}

if superfund_data is not None and not superfund_data.empty and show_superfund:
    # Option A: Render as buffered zones
    buffered = create_superfund_markers(superfund_data, buffer_meters=300)
    for status, theme_key in SUPERFUND_STATUS_COLORS.items():
        status_data = buffered[buffered['NPL_STATUS'].str.contains(status, na=False)]
        if not status_data.empty:
            status_data.plot(
                ax=ax,
                color=THEME.get(theme_key, '#FF0000'),
                alpha=THEME.get('superfund_opacity', 0.6),
                edgecolor=THEME.get('superfund_border_color', '#000000'),
                linewidth=THEME.get('superfund_border_width', 1.5),
                zorder=4  # Above roads for visibility
            )

    # Option B: Render as point markers with hazard symbol
    # superfund_data.plot(
    #     ax=ax,
    #     marker='o',  # or custom hazard marker
    #     markersize=THEME.get('superfund_marker_size', 100),
    #     color=THEME['superfund_active'],
    #     edgecolor='black',
    #     zorder=5
    # )
```

#### CLI Integration

```python
parser.add_argument('--show-superfund', action='store_true',
                    help='Show EPA Superfund site locations (US only)')
parser.add_argument('--superfund-buffer', type=int, default=300,
                    help='Buffer radius around Superfund sites in meters')
```

---

## Architecture Changes

### Updated Layer Z-Order Stack

```
z=11   Text labels (city, country, coordinates)
z=10   Gradient fades (top & bottom)
z=5    Superfund markers (highest data layer)
z=4    Superfund buffer zones
z=3    Roads
z=2.5  Parks
z=2    Water
z=1    Pollution contours
z=0.5  Income choropleth
z=0    Background color
```

### New File Structure

```
maptoposter/
├── create_map_poster.py       # Main application (updated)
├── data_fetchers/             # NEW: Modular data fetching
│   ├── __init__.py
│   ├── census.py              # Income data from Census Bureau
│   ├── pollution.py           # Air quality from OpenWeather
│   └── superfund.py           # EPA Superfund sites
├── themes/                    # Updated with new color properties
├── fonts/
├── posters/
└── README.md
```

### Updated CLI Interface

```bash
# Full example with all new layers
python create_map_poster.py \
    --city "Newark" \
    --country "USA" \
    --theme "blueprint" \
    --show-income \
    --show-pollution \
    --show-superfund \
    --census-api-key "YOUR_KEY" \
    --openweather-api-key "YOUR_KEY" \
    --superfund-buffer 500
```

---

## Configuration File Support

Add support for a `.maptoposter.json` config file to store API keys:

```json
{
  "api_keys": {
    "census": "your-census-api-key",
    "openweather": "your-openweather-api-key"
  },
  "defaults": {
    "show_income": false,
    "show_pollution": false,
    "show_superfund": false,
    "superfund_buffer": 300
  }
}
```

---

## Theme Considerations

### Creating Environmental-Focused Themes

New theme file: `themes/environmental.json`

```json
{
  "name": "Environmental Impact",
  "description": "Designed to highlight environmental data layers",
  "bg": "#1A1A2E",
  "text": "#EAEAEA",
  "gradient_color": "#1A1A2E",
  "water": "#0077B6",
  "parks": "#2D6A4F",
  "road_motorway": "#4A4A4A",
  "road_primary": "#3A3A3A",
  "road_secondary": "#2A2A2A",
  "road_tertiary": "#1A1A1A",
  "road_residential": "#151515",
  "road_default": "#2A2A2A",
  "income_level_1": "#081C15",
  "income_level_2": "#1B4332",
  "income_level_3": "#40916C",
  "income_level_4": "#74C69D",
  "income_level_5": "#B7E4C7",
  "income_opacity": 0.35,
  "pollution_good": "#06D6A0",
  "pollution_moderate": "#FFD166",
  "pollution_unhealthy_sensitive": "#F77F00",
  "pollution_unhealthy": "#EF476F",
  "pollution_very_unhealthy": "#9D4EDD",
  "pollution_opacity": 0.4,
  "superfund_active": "#FF006E",
  "superfund_construction": "#FB5607",
  "superfund_deleted": "#8D99AE",
  "superfund_proposed": "#FFBE0B",
  "superfund_opacity": 0.7,
  "superfund_border_color": "#FFFFFF",
  "superfund_border_width": 2
}
```

---

## Dependencies to Add

```txt
# requirements.txt additions
scipy>=1.11.0        # For pollution interpolation
requests>=2.31.0     # For API calls (likely already available)
```

---

## Implementation Priority

| Phase | Layer | Effort | Value |
|-------|-------|--------|-------|
| 1 | Superfund Sites | Low | High (point data, simple fetch) |
| 2 | Pollution Levels | Medium | High (requires interpolation) |
| 3 | Income Levels | Medium | Medium (requires Census API key) |

### Recommended Order
1. **Superfund Sites** - Simplest to implement, clear visual impact
2. **Pollution Levels** - More complex but highly visual
3. **Income Levels** - Requires most data processing

---

## Limitations & Considerations

### Data Coverage
- **Income data**: US Census only covers USA; international requires different sources
- **Superfund sites**: USA only (EPA jurisdiction)
- **Pollution data**: Global via OpenWeather, but resolution varies

### Performance Impact
- Additional API calls increase poster generation time
- Pollution grid interpolation is computationally expensive
- Consider caching API responses for repeated requests

### Visual Clarity
- Multiple overlapping layers may reduce map readability
- Consider adding legend/key for data layers
- Opacity tuning is critical for layer stacking

### API Rate Limits
- Census API: 500 requests/day (free key)
- OpenWeather: 60 requests/minute (free tier)
- EPA APIs: Generally unrestricted

---

## Summary

This proposal outlines a modular approach to adding income, pollution, and Superfund site data to maptoposter. The architecture follows existing patterns:

1. **Fetch** → External APIs (Census, OpenWeather, EPA)
2. **Transform** → GeoDataFrames with classification
3. **Render** → Theme-aware plotting with z-order stacking
4. **Configure** → CLI flags and theme JSON properties

The implementation maintains backwards compatibility while enabling powerful new visualizations for environmental and socioeconomic analysis.
