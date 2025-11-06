# Africa Railway Distance and Emissions Calculator

A web-based tool for calculating railway distances, travel times, and carbon emissions across African railway networks. Built with OpenStreetMap data and advanced pathfinding algorithms.

🌐 **Live Demo:** [View the calculator](https://njfinney.github.io/Africa_Railway_Calculator)

---

## 🎯 What This Tool Does

This calculator helps you:
- **Calculate actual railway distances** between any two stations in Africa
- **Estimate travel and shipment times** based on average freight speeds
- **Calculate CO2e emissions** per metric ton of cargo
- **Visualize routes** on an interactive map
- **Identify disconnections** in railway networks (with gap analysis)

### Supported Countries
**Southern Africa:** Mozambique, Malawi, Zambia, Zimbabwe, South Africa  
**East Africa:** Tanzania, Kenya  
**West Africa:** Nigeria, Cameroon, Benin, Côte d'Ivoire, Burkina Faso, Liberia, Senegal, Mali

---

## 📚 How It Works: From Simple to Complex

### Level 1: The Basic Concept
Think of railways as a network of connected points (like a connect-the-dots puzzle):
- **Stations** are your start and end points
- **Railway tracks** connect these stations
- The tool finds the shortest path along these tracks

### Level 2: The Process Flow
```
1. Select countries → Load railway data from OpenStreetMap
2. Choose stations → Search by name, country, or map
3. Click calculate → Algorithm finds the shortest path
4. View results → Distance, time, emissions, and map visualization
```

### Level 3: The Core Logic

#### A. Data Collection
1. **Query OpenStreetMap** for railway stations in selected region
2. Fetch stations tagged as: `railway=station|halt|stop`, `building=train_station`, etc.
3. Filter out irrelevant buildings (keep only those near towns)
4. Assign countries using multiple methods (OSM tags → text search → geographic location)

#### B. Route Calculation
When you request a route:
1. **Expand the search area** by 1° around both stations (to catch connecting railways)
2. **Fetch all railway lines** in that bounding box from OpenStreetMap
3. **Build a graph**:
   - Each point on a railway becomes a "node"
   - Connections between adjacent points become "edges" with distances
4. **Find closest nodes** to each station (within a few km)
5. **Run Dijkstra's algorithm** to find shortest path through the graph
6. **Calculate total distance** = (station to rail) + (path along railways) + (rail to station)

#### C. Disconnection Analysis
If no path exists:
1. **Perform bidirectional search** from both stations
2. Find which nodes are reachable from each side
3. **Calculate the gap** between the two disconnected networks
4. **Check for obstacles** (rivers, borders) at the break point
5. **Show partial routes** with the exact gap location

---

## 🔧 Technical Implementation

### Architecture Overview
```
┌─────────────────────────────────────────┐
│  User Interface (HTML + Tailwind CSS)  │
├─────────────────────────────────────────┤
│  Application Logic (Vanilla JavaScript) │
│  ├─ Data Layer (Overpass API queries)  │
│  ├─ Graph Builder (Railway network)    │
│  ├─ Pathfinding (Dijkstra's algorithm) │
│  └─ Visualization (Leaflet.js maps)    │
├─────────────────────────────────────────┤
│  External APIs                          │
│  ├─ Overpass API (OpenStreetMap data)  │
│  └─ Nominatim (reverse geocoding)      │
└─────────────────────────────────────────┘
```

### Key Technologies

#### 1. **OpenStreetMap & Overpass API**
- **What:** Real-time access to global mapping data
- **Why:** Contains detailed railway infrastructure data
- **Query Example:**
```javascript
[out:json][bbox:south,west,north,east];
(
  node["railway"~"station|halt|stop"];
  way["railway"="station"];
);
out center tags;
```

#### 2. **Graph Theory & Dijkstra's Algorithm**
- **Graph Structure:** `{ nodes: Map<id, {lat, lon, edges}>, nodeIndex: Map<coords, id> }`
- **Algorithm:** Finds shortest path in weighted graph
- **Time Complexity:** O((V + E) log V) where V=nodes, E=edges
- **Why Dijkstra?** 
  - Guarantees shortest path
  - Works well with real distances
  - Efficient for sparse graphs (railway networks)

#### 3. **Haversine Distance Formula**
Calculates great-circle distance between two points on Earth:
```javascript
function haversine(lat1, lon1, lat2, lon2) {
    const R = 6371; // Earth radius in km
    const dLat = (lat2 - lat1) * Math.PI / 180;
    const dLon = (lon2 - lon1) * Math.PI / 180;
    const a = Math.sin(dLat/2) * Math.sin(dLat/2) +
              Math.cos(lat1 * Math.PI / 180) * Math.cos(lat2 * Math.PI / 180) *
              Math.sin(dLon/2) * Math.sin(dLon/2);
    return R * 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a));
}
```

#### 4. **Leaflet.js Mapping**
- Interactive map with OpenStreetMap tiles
- Custom markers for stations and break points
- Polylines for railway routes (color-coded by status)

### Data Flow Diagram
```
User Input
    ↓
[Select Countries] → Combine bounding boxes
    ↓
[Fetch Stations] → Overpass API query
    ↓
[Process Data] → Deduplicate, assign countries, sort
    ↓
[Display UI] → Populate dropdowns/map
    ↓
[Calculate Route]
    ↓
[Fetch Railways] → Query railways in route area
    ↓
[Build Graph] → Create nodes from track geometry
    ↓
[Find Path] → Dijkstra's algorithm
    ↓
[Check Result]
    ├─ Connected → Calculate metrics, display route
    └─ Disconnected → Analyze gap, show partial routes
```

### Country Detection Algorithm
Multi-stage fallback system (highest priority first):

1. **Single-country mode:** If user selected one country, assign that
2. **OSM tags:** Check `addr:country`, `is_in:country`, `country` tags
3. **Text extraction:** Search address fields for country names
4. **Geographic center:** Calculate distance to each country's center point, pick closest

### Disconnection Analysis Algorithm

When Dijkstra's algorithm returns `distance = Infinity`:

1. **Bidirectional BFS (Breadth-First Search):**
   - From Station 1: Find all reachable nodes (forward search)
   - From Station 2: Find all reachable nodes (backward search)

2. **Gap Calculation:**
   - Find closest pair of nodes between the two networks
   - Calculate distance between them (the "gap")

3. **Obstacle Detection:**
   - Query Overpass API for rivers/waterways at gap midpoint
   - Check if gap crosses international border

4. **Path Reconstruction:**
   - Build partial path from Station 1 to its network's edge
   - Build partial path from Station 2 to its network's edge
   - Calculate distances for each segment

5. **Visualization:**
   - Blue line: Railway from Station 1
   - Purple line: Railway from Station 2
   - Red dashed line: The gap
   - Broken chain icons: Exact break points

---

## 📊 Calculations Explained

### 1. Railway Distance
- Actual distance along railway tracks (not straight-line)
- Calculated by summing edge weights in shortest path
- Includes connection distance from stations to nearest rail node

### 2. Travel Time
```
Travel Time (hours) = Railway Distance (km) / 35 km/h
```
- Assumes average freight train speed of 35 km/h
- Conservative estimate accounting for stops, delays

### 3. Shipment Time
```
Shipment Time (days) = (Travel Time × 3) / 24
```
- Multiplied by 3 to account for loading, unloading, customs
- Converted from hours to days

### 4. CO2e Emissions
```
Emissions (kgCO2e per MT) = Railway Distance (km) × 0.024278
```
- Based on rail freight emission factor: **24.278 gCO2e per ton-km**
- Significantly lower than road transport (~62 gCO2e/ton-km)
- Calculated per metric ton (MT) of cargo

---

## 🛠️ Development Details

### File Structure
```
Africa_Railway_Calculator/
├── index.html          # Main application (self-contained)
└── README.md          # This file
```

### State Management
All application state in single object:
```javascript
const STATE = {
    stations: [],              // All loaded stations
    filteredStations: [[], []], // Filtered for each dropdown
    selectedStations: [null, null], // Currently selected
    railwayGraph: null,        // Built graph for routing
    map: null,                 // Leaflet map instance
    pickerMap: null,          // Map picker modal instance
    towns: []                 // Major towns for filtering
};
```

### Performance Optimizations

1. **Caching:** Town locations cached to avoid repeated lookups
2. **Deduplication:** Stations within 500m combined to single point
3. **Lazy Loading:** Railway data only fetched when calculating route
4. **Smart Queries:** Bounding box queries instead of full country downloads
5. **Iteration Limits:** Pathfinding capped at 100,000 iterations

### Error Handling

- **Network failures:** Graceful degradation with error messages
- **Invalid selections:** User validation before API calls
- **No data found:** Clear explanations with suggestions
- **Disconnected networks:** Detailed analysis instead of generic error

---

## 🚀 Usage Guide

### Basic Workflow
1. **Select countries** (hold Ctrl/Cmd for multiple)
2. Click "Load Selected Regions"
3. **Choose stations** using any method:
   - 🔍 Type in search box
   - 🌍 Filter by country
   - 📋 Select from list
   - 🗺️ Click on map
4. Click "Calculate Rail Distance & Emissions"

### Multi-Select Tips
- Select 2-3 countries for optimal performance
- Avoid loading entire regions unless necessary
- Use country filters to narrow down station lists

### Interpreting Results

#### Connected Routes
- **Railway Distance:** Actual km along tracks
- **Travel Time:** Estimated hours at 35 km/h
- **Shipment Time:** Total days including logistics
- **CO2e Emissions:** Environmental impact per ton

#### Disconnected Routes
- **Partial distances:** How far each railway extends
- **Gap size:** Distance of missing connection
- **Obstacle type:** River crossing, border, etc.
- **Hypothetical total:** What distance would be if connected

---

## 🔍 Data Sources & Attribution

### OpenStreetMap
- **License:** ODbL (Open Database License)
- **Data access:** Overpass API
- **Attribution:** © OpenStreetMap contributors

### Emission Factors
- Rail freight: 24.278 gCO2e/ton-km (IPCC guidelines)
- Based on diesel locomotives (average for African railways)

---

## ⚠️ Limitations & Known Issues

### Data Quality
- **OSM completeness varies** by country (some areas better mapped than others)
- **Station names** may be inconsistent or missing
- **Railway status** not always current (some lines may be inactive)

### Routing Limitations
- **No consideration for:**
  - Gauge differences (standard vs narrow gauge)
  - Operational status (active vs abandoned)
  - Political restrictions (borders may not be crossable)
  - Track conditions or speed limits

### Calculation Assumptions
- **Fixed speed:** 35 km/h may not reflect actual operations
- **Simplified logistics:** 3× multiplier is rough estimate
- **Emission factor:** Assumes diesel locomotives (not electrified routes)

### Technical Constraints
- **Network requests:** Limited by Overpass API rate limits
- **Browser memory:** Very large regions may be slow
- **Precision:** Coordinates rounded to 6 decimal places (~0.1m accuracy)

---

## 🤝 Contributing

This project is open for contributions! Areas for improvement:

### High Priority
- [ ] Add more African countries (Ghana, Uganda, Ethiopia, etc.)
- [ ] Incorporate gauge information (for compatibility checking)
- [ ] Add operational status filtering (active vs inactive lines)
- [ ] Improve emission calculations (account for electrification)

### Medium Priority
- [ ] Export results to CSV/PDF
- [ ] Save/share route links
- [ ] Historical distance comparisons
- [ ] Multi-stop route planning

### Low Priority
- [ ] Dark mode
- [ ] Internationalization (French, Portuguese, etc.)
- [ ] Mobile app version

### How to Contribute
1. Fork the repository
2. Create a feature branch (`git checkout -b feature/new-feature`)
3. Make your changes
4. Test thoroughly
5. Submit a pull request

---

## 📝 Version History

### v1.0.0 (Current)
- ✅ 15 African countries supported
- ✅ Actual railway routing with Dijkstra's algorithm
- ✅ Disconnection analysis with gap detection
- ✅ Multi-country selection
- ✅ Multiple station selection methods
- ✅ River/border obstacle detection
- ✅ CO2e emissions calculation

---

## 📧 Contact & Support

**Developer:** Nicholas Finney  
**Repository:** [github.com/njfinney/Africa_Railway_Calculator](https://github.com/njfinney/Africa_Railway_Calculator)  
**Issues:** [Report a bug or request a feature](https://github.com/njfinney/Africa_Railway_Calculator/issues)

---

## 📄 License

This project is licensed under the MIT License - see below for details:

```
MIT License

Copyright (c) 2024 Nicholas Finney

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

---

## 🙏 Acknowledgments

- **OpenStreetMap contributors** for railway infrastructure data
- **Overpass API** for efficient OSM data access
- **Leaflet.js** for mapping functionality
- **Tailwind CSS** for UI styling
- **Claude AI** for development assistance

---

## 🎓 Educational Use

This tool is ideal for:
- **Logistics planning:** Understanding freight route options
- **Infrastructure analysis:** Identifying gaps in railway networks
- **Environmental studies:** Comparing transport emissions
- **Geographic education:** Learning about African rail systems
- **Development planning:** Prioritizing railway investments

---

**Built with ❤️ for African rail development**
