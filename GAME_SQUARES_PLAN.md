# Game Squares Implementation Plan

## Overview

Add a "game squares" layer to the fantasy city map viewer that partitions each district into 5-10 squares based on road boundaries, with each square tracking which buildings it contains. This will support integration with a board game where players place influence on squares.

## Data Structure

Each game square will have:
```javascript
{
  id: "gs_0_5",                    // gs_{districtIndex}_{squareIndex}
  name: "Block A",                 // User-editable
  districtIndex: 0,
  districtName: "Old Wall",
  polygon: [[x,y], ...],           // Boundary coordinates
  centroid: [cx, cy],              // For label placement
  buildingIndices: [45, 46, ...],  // References to cityData.buildings
  buildingCount: 23,
  generationType: "road_block" | "cluster",
  notes: "",
  color: null                      // Optional override
}
```

## Algorithm Summary

### Step 1: Assign Buildings to Districts
- For each of 12,009 buildings, calculate centroid
- Find which district polygon contains it (point-in-polygon test)
- Result: Map of districtIndex -> Set of buildingIndices

### Step 2: Partition Each District into Squares
**Primary approach (road-based):** Use roads as spatial dividers
- Extract roads within each district
- Use road segments as barriers to partition buildings
- Buildings on same "side" of major roads grouped together

**Fallback (k-means clustering):** For districts with few roads
- Cluster buildings by proximity
- Target k = 5-10 based on district size

### Step 3: Generate Square Boundaries
- Compute convex hull of all building vertices in each cluster
- Add small buffer for visual separation

### Target: 5-10 Squares Per District
- Scale by district area relative to average
- Minimum 1 square for tiny districts
- Maximum 15 for very large districts (Butchers Crossing)

## Files to Create/Modify

**Create: `viewer2.html`**
- Copy of viewer.html with all game squares functionality added
- Original viewer.html preserved unchanged

### Changes in viewer2.html

#### 1. State Variables (~line 132)
```javascript
let gameSquares = [];
let showGameSquares = false;
let selectedSquareIndex = null;
```

#### 2. UI Elements
- Add "Generate Squares" button in header (~line 84)
- Add "Toggle Squares" button (hidden until generated)
- Add square info panel (similar to building panel, ~line 121)

#### 3. Generation Algorithm (~600 lines of new code)
- `assignBuildingsToDistricts()` - spatial containment
- `extractDistrictRoads(districtIndex)` - filter roads by district
- `partitionByRoads()` - primary partitioning using road barriers
- `partitionByClustering()` - k-means fallback
- `generateSquareBoundary()` - convex hull calculation
- `generateGameSquares()` - main orchestration function

#### 4. Rendering (~line 995, after greens, before roads)
- Semi-transparent fill per square (20% opacity)
- Dashed stroke for boundaries
- Different color per district
- Square labels when showLabels is on
- Selection highlight (50% opacity, solid stroke)

#### 5. Interaction
- `findSquareAtWorld(x, y)` - hit detection with bounds check
- Click handler modification - check squares before buildings
- `showSquarePanel()` / `hideSquarePanel()` - panel display

#### 6. Export/Import (~line 1649, 1769)
- Save gameSquares array to exported JSON
- Load and reconstruct computed fields on import

## Utility Functions Needed

Several exist and can be reused:
- `pointInBuildingPoly()` (line 414) - adapt for districts
- `polygonArea()` (line 401) - already exists
- `drawPolygonCoords()` (line 930) - for rendering

New utilities to add:
- `calculatePolygonCentroid(coords)`
- `convexHull(points)` - Graham scan or Andrew's monotone chain
- `distance(p1, p2)` - Euclidean distance
- `getDistrictSquareColor(index)` - color palette
- `adjustAlpha(hexColor, alpha)` - rgba conversion

## Visual Design

- Each district gets a unique color from an 11-color palette
- Squares rendered as semi-transparent overlays (20% fill, 80% stroke)
- Dashed boundaries (5px dash, 3px gap)
- Selected square: solid boundary, 50% fill
- Labels: district-colored text at centroid

## Edge Cases Handled

1. **Buildings outside all districts** - assign to nearest district
2. **Districts with <3 roads** - skip road partitioning, use clustering
3. **Very small districts** - minimum 1 square
4. **Very large districts** - allow up to 15 squares
5. **Empty clusters** - merge into nearest non-empty cluster

## Testing

1. Load `hunters_church.json` (11 districts, 12,009 buildings)
2. Click "Generate Squares" button
3. Verify each district has 5-10 squares (check console log)
4. Toggle squares visibility on/off
5. Click squares to select and view info panel
6. Edit square name, save, verify persistence
7. Export JSON, reload, verify squares restored
8. Test with `western_islands.json` for different map configuration

## Implementation Order

1. Copy viewer.html to viewer2.html
2. Add state variables and UI buttons
3. Implement utility functions (centroid, convex hull, distance)
4. Implement building-to-district assignment
5. Implement k-means clustering (simpler, test first)
6. Implement road-based partitioning
7. Implement boundary generation
8. Implement main generation function
9. Add rendering code
10. Add click/selection handlers
11. Add square info panel
12. Add export/import support
13. Test and tune parameters

## Key Data from Exploration

**hunters_church.json statistics:**
- 11 named districts (Military District, The Harbour, Old Wall, etc.)
- 12,009 buildings (flat MultiPolygon, no properties)
- 1,172 roads (LineStrings, widths 4.8 and 8)
- Roads are NOT connected (independent segments)
- Buildings have no district association (must compute spatially)

**District sizes vary dramatically:**
- Smallest: Dark Crowns Ward (7 vertices)
- Largest: Butchers Crossing (2.2M sq units, 191 roads)
- Some districts have very few roads (Military District: 1 road)

This means the fallback clustering algorithm is essential for districts with sparse road networks.
