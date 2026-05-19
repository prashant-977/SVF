# DIGO Visitor Flow Map - Intern Project Setup

## Project Overview

**Goal**: Build a standalone Next.js application for visualizing tourism visitor flows on interactive maps for Nepal municipalities.

**Current Status**: We have a working React prototype (Figma Make) with Pokhara & Madi maps. Your task is to rebuild this as a production-ready Next.js application with real data integration.

**Timeline**: 
- **Phase 1 (Weeks 1-2)**: Map visualizer only
- **Phase 2 (Weeks 3-6)**: Full DIGO platform features
- **Phase 3 (Weeks 7-8)**: Production deployment

---

## Phase 1: Standalone Map Visualizer (Start Here)

### Week 1: Project Setup & Basic Map

**What you'll build**: A Next.js app that shows Pokhara visitor flows on a Leaflet map.

#### Setup Instructions

```bash
# 1. Create Next.js project
npx create-next-app@latest digo-visitor-flow-map
# Choose: TypeScript ✓, Tailwind CSS ✓, App Router ✓

cd digo-visitor-flow-map

# 2. Install dependencies
npm install leaflet react-leaflet @types/leaflet
npm install recharts  # for charts later
npm install lucide-react  # icons

# 3. Create project structure
mkdir -p src/app/api
mkdir -p src/components/map
mkdir -p src/lib
mkdir -p src/types
mkdir -p public/data
```

#### File Structure
```
digo-visitor-flow-map/
├── src/
│   ├── app/
│   │   ├── page.tsx                 # Home page with map
│   │   ├── layout.tsx               # Root layout
│   │   └── api/                     # API routes (Phase 2)
│   ├── components/
│   │   ├── map/
│   │   │   ├── VisitorFlowMap.tsx   # Main map component
│   │   │   ├── FlowLayer.tsx        # Flow lines rendering
│   │   │   ├── LocationMarker.tsx   # Individual markers
│   │   │   └── MapLegend.tsx        # Legend component
│   │   └── ui/                      # Reusable UI components
│   ├── lib/
│   │   ├── data.ts                  # Data loading utilities
│   │   └── utils.ts                 # Helper functions
│   └── types/
│       └── index.ts                 # TypeScript types
├── public/
│   └── data/
│       ├── pokhara-locations.json   # Location coordinates
│       └── pokhara-flows.json       # Flow data
└── package.json
```

#### Step 1: Define TypeScript Types

**File**: `src/types/index.ts`

```typescript
// Evidence Quality Tiers for Data Validation
export type EvidenceTier = 'tier1' | 'tier2' | 'tier3';

export interface EvidenceQuality {
  tier: EvidenceTier;
  sources: string[];  // List of data sources used
  validated_by: string;  // Who validated this data (e.g., "Municipality Team", "DIGO Workshop")
  validation_date: string;  // ISO date string
  notes?: string;  // Additional context
}

// Location on the map
export interface Location {
  id: string;
  name: string;
  lat: number;
  lng: number;
  type: 'entry' | 'attraction' | 'service' | 'accommodation' | 'challenge';
  visitors: number;
  capacity?: number;
  category: string;
  evidenceQuality: EvidenceQuality;  // Data quality tracking
}

// A single path segment in a flow
export interface FlowPath {
  from: string;  // Location ID
  to: string;    // Location ID
  visitors: number;
  mode: 'walk' | 'taxi' | 'boat' | 'jeep' | 'hike';
  evidenceQuality: EvidenceQuality;  // Data quality tracking
}

// A complete visitor flow
export interface VisitorFlow {
  id: string;
  name: string;
  paths: FlowPath[];
  pressure: 'low' | 'medium' | 'high' | 'critical';
  destination: 'pokhara' | 'madi';
}

// Helper to get pressure color
export function getPressureColor(pressure: VisitorFlow['pressure']): string {
  const colors = {
    low: '#10b981',      // Green
    medium: '#eab308',   // Yellow
    high: '#f59e0b',     // Amber
    critical: '#dc2626'  // Red
  };
  return colors[pressure];
}

// Helper to get evidence tier info
export function getEvidenceTierInfo(tier: EvidenceTier) {
  const tiers = {
    tier1: {
      name: 'Tier 1 - High Confidence',
      color: '#059669',  // Green
      description: 'Validated by multiple sources',
      examples: ['Official entry data', 'Hotel occupancy records', 'Payment gateway transactions', 'Government survey data']
    },
    tier2: {
      name: 'Tier 2 - Medium Confidence',
      color: '#d97706',  // Amber
      description: 'Stakeholder consensus',
      examples: ['Hotel association estimates', 'Guide direct records', 'Business owner observations', 'Cross-validated workshop estimates']
    },
    tier3: {
      name: 'Tier 3 - Low Confidence',
      color: '#dc2626',  // Red
      description: 'Single source - needs verification',
      examples: ['Individual business estimates', 'Seasonal observations', 'Anecdotal reports']
    }
  };
  return tiers[tier];
}
```

#### Step 2: Create Sample Data Files

**File**: `public/data/pokhara-locations.json`

```json
[
  {
    "id": "airport",
    "name": "Pokhara Airport",
    "lat": 28.2009,
    "lng": 83.9821,
    "type": "entry",
    "visitors": 4200,
    "category": "Entry Point",
    "evidenceQuality": {
      "tier": "tier1",
      "sources": ["Official airport arrival records", "Immigration data", "Airport authority monthly reports"],
      "validated_by": "Civil Aviation Authority & DIGO Team",
      "validation_date": "2026-04-15",
      "notes": "Cross-referenced with national tourism statistics"
    }
  },
  {
    "id": "fewa-lake",
    "name": "Fewa Lake Waterfront",
    "lat": 28.2096,
    "lng": 83.9586,
    "type": "attraction",
    "visitors": 12000,
    "capacity": 8000,
    "category": "Natural Attraction",
    "evidenceQuality": {
      "tier": "tier2",
      "sources": ["Hotel association estimates", "Lakeside business owners survey", "Municipal workshop consensus"],
      "validated_by": "Pokhara Tourism Stakeholder Workshop",
      "validation_date": "2026-04-20",
      "notes": "Estimates vary by season, average taken across 3 months"
    }
  },
  {
    "id": "lakeside",
    "name": "Lakeside (Baidam)",
    "lat": 28.2089,
    "lng": 83.9595,
    "type": "service",
    "visitors": 9500,
    "category": "Tourism Hub",
    "evidenceQuality": {
      "tier": "tier3",
      "sources": ["Individual restaurant owner estimate"],
      "validated_by": "Single stakeholder interview",
      "validation_date": "2026-04-18",
      "notes": "Needs verification through workshop or multiple business surveys"
    }
  }
]
```

**File**: `public/data/pokhara-flows.json`

```json
[
  {
    "id": "flow-1",
    "name": "Airport Arrival → Lakeside",
    "paths": [
      { 
        "from": "airport", 
        "to": "lakeside", 
        "visitors": 3500, 
        "mode": "taxi",
        "evidenceQuality": {
          "tier": "tier1",
          "sources": ["Taxi association daily logs", "Hotel check-in records", "Payment gateway data"],
          "validated_by": "Transportation & Hospitality Stakeholder Meeting",
          "validation_date": "2026-04-22"
        }
      },
      { 
        "from": "lakeside", 
        "to": "fewa-lake", 
        "visitors": 2800, 
        "mode": "walk",
        "evidenceQuality": {
          "tier": "tier2",
          "sources": ["Tour guide observations", "Restaurant owner estimates"],
          "validated_by": "Lakeside Business Workshop",
          "validation_date": "2026-04-20",
          "notes": "Pedestrian counting recommended for verification"
        }
      }
    ],
    "pressure": "medium",
    "destination": "pokhara"
  },
  {
    "id": "flow-7",
    "name": "Fewa Lake Convergence (Overcrowded)",
    "paths": [
      { 
        "from": "lakeside", 
        "to": "fewa-lake", 
        "visitors": 4700, 
        "mode": "walk",
        "evidenceQuality": {
          "tier": "tier3",
          "sources": ["Single boat operator estimate"],
          "validated_by": "Individual interview - needs workshop validation",
          "validation_date": "2026-04-19",
          "notes": "High priority for verification given critical pressure rating"
        }
      }
    ],
    "pressure": "critical",
    "destination": "pokhara"
  }
]
```

#### Step 3: Build the Map Component

**File**: `src/components/map/VisitorFlowMap.tsx`

```typescript
'use client';

import { useEffect, useRef } from 'react';
import L from 'leaflet';
import 'leaflet/dist/leaflet.css';
import { Location, VisitorFlow, getPressureColor } from '@/types';

// Fix Leaflet icon issue in Next.js
import icon from 'leaflet/dist/images/marker-icon.png';
import iconShadow from 'leaflet/dist/images/marker-shadow.png';

let DefaultIcon = L.icon({
  iconUrl: icon.src,
  shadowUrl: iconShadow.src,
  iconSize: [25, 41],
  iconAnchor: [12, 41]
});

L.Marker.prototype.options.icon = DefaultIcon;

interface Props {
  locations: Location[];
  flows: VisitorFlow[];
  center?: [number, number];
  zoom?: number;
}

export default function VisitorFlowMap({ 
  locations, 
  flows, 
  center = [28.2096, 83.9629], 
  zoom = 13 
}: Props) {
  const mapRef = useRef<L.Map | null>(null);
  const containerRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (!containerRef.current || mapRef.current) return;

    // Initialize map
    const map = L.map(containerRef.current).setView(center, zoom);

    // Add OpenStreetMap tiles
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
      attribution: '© OpenStreetMap contributors',
      maxZoom: 18
    }).addTo(map);

    // Add location markers
    locations.forEach(location => {
      const marker = L.marker([location.lat, location.lng]).addTo(map);
      
      const evidenceInfo = getEvidenceTierInfo(location.evidenceQuality.tier);
      
      marker.bindPopup(`
        <div style="min-width: 220px;">
          <h3 style="font-weight: bold; margin-bottom: 8px;">${location.name}</h3>
          <div><strong>Visitors:</strong> ${location.visitors.toLocaleString()}/day</div>
          ${location.capacity ? `
            <div><strong>Capacity:</strong> ${location.capacity.toLocaleString()}</div>
            <div style="margin-top: 8px; padding: 4px 8px; background: ${
              location.visitors > location.capacity * 1.2 ? '#dc2626' : 
              location.visitors > location.capacity ? '#f59e0b' : '#10b981'
            }; color: white; border-radius: 4px; text-align: center; font-weight: bold;">
              ${(location.visitors / location.capacity * 100).toFixed(0)}% CAPACITY
            </div>
          ` : ''}
          
          <!-- Evidence Quality Badge -->
          <div style="margin-top: 12px; padding: 8px; background: ${evidenceInfo.color}15; border-left: 3px solid ${evidenceInfo.color}; border-radius: 4px;">
            <div style="color: ${evidenceInfo.color}; font-weight: bold; font-size: 11px; text-transform: uppercase; margin-bottom: 4px;">
              ${evidenceInfo.name}
            </div>
            <div style="font-size: 11px; color: #666; margin-bottom: 4px;">
              ${evidenceInfo.description}
            </div>
            <div style="font-size: 10px; color: #888;">
              <strong>Sources:</strong> ${location.evidenceQuality.sources.join(', ')}
            </div>
            <div style="font-size: 10px; color: #888; margin-top: 2px;">
              <strong>Validated:</strong> ${location.evidenceQuality.validation_date}
            </div>
          </div>
        </div>
      `);
    });

    // Draw flow paths
    flows.forEach(flow => {
      const color = getPressureColor(flow.pressure);

      flow.paths.forEach(path => {
        const fromLoc = locations.find(l => l.id === path.from);
        const toLoc = locations.find(l => l.id === path.to);

        if (fromLoc && toLoc) {
          const polyline = L.polyline(
            [[fromLoc.lat, fromLoc.lng], [toLoc.lat, toLoc.lng]],
            {
              color,
              weight: 3 + (path.visitors / 1000) * 2,
              opacity: 0.7,
              dashArray: path.mode === 'walk' ? '8, 4' : undefined
            }
          ).addTo(map);

          const pathEvidenceInfo = getEvidenceTierInfo(path.evidenceQuality.tier);
          
          polyline.bindPopup(`
            <div style="min-width: 220px;">
              <h3 style="color: ${color}; font-weight: bold; margin-bottom: 8px;">
                ${flow.name}
              </h3>
              <div><strong>Route:</strong> ${fromLoc.name} → ${toLoc.name}</div>
              <div><strong>Visitors:</strong> ${path.visitors.toLocaleString()}/day</div>
              <div><strong>Transport:</strong> ${path.mode}</div>
              <div style="margin-top: 8px; padding: 4px; background: ${color}; color: white; border-radius: 4px; text-align: center; text-transform: uppercase;">
                ${flow.pressure} PRESSURE
              </div>
              
              <!-- Evidence Quality Badge -->
              <div style="margin-top: 12px; padding: 8px; background: ${pathEvidenceInfo.color}15; border-left: 3px solid ${pathEvidenceInfo.color}; border-radius: 4px;">
                <div style="color: ${pathEvidenceInfo.color}; font-weight: bold; font-size: 11px; text-transform: uppercase; margin-bottom: 4px;">
                  ${pathEvidenceInfo.name}
                </div>
                <div style="font-size: 11px; color: #666; margin-bottom: 4px;">
                  ${pathEvidenceInfo.description}
                </div>
                <div style="font-size: 10px; color: #888;">
                  <strong>Sources:</strong> ${path.evidenceQuality.sources.join(', ')}
                </div>
                ${path.evidenceQuality.notes ? `
                  <div style="font-size: 10px; color: #d97706; margin-top: 4px; font-style: italic;">
                    ⚠ ${path.evidenceQuality.notes}
                  </div>
                ` : ''}
              </div>
            </div>
          `);
        }
      });
    });

    mapRef.current = map;

    return () => {
      map.remove();
      mapRef.current = null;
    };
  }, [locations, flows, center, zoom]);

  return (
    <div 
      ref={containerRef} 
      className="h-[600px] w-full rounded-lg border-2 border-gray-300"
    />
  );
}
```

#### Step 4: Create the Home Page

**File**: `src/app/page.tsx`

```typescript
import dynamic from 'next/dynamic';
import { Location, VisitorFlow } from '@/types';

// Import map with no SSR (Leaflet needs browser)
const VisitorFlowMap = dynamic(
  () => import('@/components/map/VisitorFlowMap'),
  { ssr: false }
);

async function getMapData() {
  const baseUrl = process.env.NEXT_PUBLIC_BASE_URL || 'http://localhost:3000';
  
  const [locationsRes, flowsRes] = await Promise.all([
    fetch(`${baseUrl}/data/pokhara-locations.json`, { cache: 'no-store' }),
    fetch(`${baseUrl}/data/pokhara-flows.json`, { cache: 'no-store' })
  ]);

  const locations: Location[] = await locationsRes.json();
  const flows: VisitorFlow[] = await flowsRes.json();

  return { locations, flows };
}

export default async function Home() {
  const { locations, flows } = await getMapData();

  return (
    <main className="min-h-screen p-8 bg-gray-50">
      <div className="max-w-7xl mx-auto">
        <h1 className="text-3xl font-bold mb-2">Pokhara Visitor Flow Map</h1>
        <p className="text-gray-600 mb-6">
          Interactive visualization of {flows.length} validated visitor flows
        </p>

        <div className="bg-white rounded-lg shadow-lg p-6">
          <VisitorFlowMap locations={locations} flows={flows} />
        </div>

        {/* Stats */}
        <div className="mt-6 grid grid-cols-3 gap-4">
          <div className="bg-white p-4 rounded-lg shadow">
            <div className="text-sm text-gray-600">Total Locations</div>
            <div className="text-2xl font-bold">{locations.length}</div>
          </div>
          <div className="bg-white p-4 rounded-lg shadow">
            <div className="text-sm text-gray-600">Visitor Flows</div>
            <div className="text-2xl font-bold">{flows.length}</div>
          </div>
          <div className="bg-white p-4 rounded-lg shadow">
            <div className="text-sm text-gray-600">Daily Visitors</div>
            <div className="text-2xl font-bold">
              {locations.reduce((sum, loc) => sum + loc.visitors, 0).toLocaleString()}
            </div>
          </div>
        </div>
      </div>
    </main>
  );
}
```

#### Step 5: Run and Test

```bash
npm run dev
# Open http://localhost:3000
```

**Expected Result**: Interactive map of Pokhara with:
- Location markers
- Flow lines (color-coded by pressure)
- Clickable popups
- Stats summary

---

### Week 2: Data Loading & Multiple Destinations

**Tasks**:
1. ✅ Add Madi data files (`madi-locations.json`, `madi-flows.json`)
2. ✅ Create destination switcher UI
3. ✅ Add map legend component
4. ✅ Implement filter controls (show/hide flows by pressure)
5. ✅ Add loading states

**New Components**:
- `src/components/map/DestinationSelector.tsx` - Dropdown to switch between Pokhara/Madi
- `src/components/map/MapLegend.tsx` - Legend showing color codes
- `src/components/map/FlowFilter.tsx` - Checkboxes to filter by pressure level

**Example Destination Selector**:

```typescript
'use client';

import { useState } from 'react';

export default function DestinationSelector({ 
  onSelect 
}: { 
  onSelect: (dest: 'pokhara' | 'madi') => void 
}) {
  const [selected, setSelected] = useState<'pokhara' | 'madi'>('pokhara');

  return (
    <div className="flex gap-4 mb-6">
      <button
        onClick={() => { setSelected('pokhara'); onSelect('pokhara'); }}
        className={`px-6 py-3 rounded-lg font-medium transition ${
          selected === 'pokhara' 
            ? 'bg-blue-600 text-white' 
            : 'bg-white border-2 border-gray-200'
        }`}
      >
        Pokhara (19 flows)
      </button>
      <button
        onClick={() => { setSelected('madi'); onSelect('madi'); }}
        className={`px-6 py-3 rounded-lg font-medium transition ${
          selected === 'madi' 
            ? 'bg-green-600 text-white' 
            : 'bg-white border-2 border-gray-200'
        }`}
      >
        Madi RM (8 flows)
      </button>
    </div>
  );
}
```

**Evidence Quality Components**:

Add these components to help interns track and improve data quality:

**File**: `src/components/map/EvidenceQualityLegend.tsx`

```typescript
'use client';

import { getEvidenceTierInfo } from '@/types';

export default function EvidenceQualityLegend() {
  const tiers: ('tier1' | 'tier2' | 'tier3')[] = ['tier1', 'tier2', 'tier3'];
  
  return (
    <div className="bg-white p-4 rounded-lg shadow-lg">
      <h3 className="font-bold text-sm mb-3">Evidence Quality Guide</h3>
      
      {tiers.map(tier => {
        const info = getEvidenceTierInfo(tier);
        return (
          <div key={tier} className="mb-3 last:mb-0">
            <div 
              className="flex items-center gap-2 mb-1"
              style={{ borderLeft: `4px solid ${info.color}`, paddingLeft: '8px' }}
            >
              <div 
                className="w-3 h-3 rounded-full" 
                style={{ background: info.color }}
              />
              <span className="font-semibold text-xs">{info.name}</span>
            </div>
            <p className="text-xs text-gray-600 ml-6 mb-1">{info.description}</p>
            <div className="text-xs text-gray-500 ml-6">
              <strong>Examples:</strong>
              <ul className="list-disc list-inside mt-1">
                {info.examples.slice(0, 2).map((ex, i) => (
                  <li key={i}>{ex}</li>
                ))}
              </ul>
            </div>
          </div>
        );
      })}
      
      <div className="mt-4 pt-3 border-t border-gray-200">
        <p className="text-xs text-gray-600">
          <strong>Intern Goal:</strong> Move all Tier 3 data to Tier 2+ through workshops and cross-validation
        </p>
      </div>
    </div>
  );
}
```

**File**: `src/components/EvidenceQualityDashboard.tsx`

```typescript
'use client';

import { Location, FlowPath, getEvidenceTierInfo } from '@/types';
import { PieChart, Pie, Cell, ResponsiveContainer, Legend, Tooltip } from 'recharts';

interface Props {
  locations: Location[];
  flows: { paths: FlowPath[] }[];
}

export default function EvidenceQualityDashboard({ locations, flows }: Props) {
  // Calculate statistics
  const allPaths = flows.flatMap(f => f.paths);
  
  const locationsByTier = {
    tier1: locations.filter(l => l.evidenceQuality.tier === 'tier1').length,
    tier2: locations.filter(l => l.evidenceQuality.tier === 'tier2').length,
    tier3: locations.filter(l => l.evidenceQuality.tier === 'tier3').length,
  };
  
  const pathsByTier = {
    tier1: allPaths.filter(p => p.evidenceQuality.tier === 'tier1').length,
    tier2: allPaths.filter(p => p.evidenceQuality.tier === 'tier2').length,
    tier3: allPaths.filter(p => p.evidenceQuality.tier === 'tier3').length,
  };
  
  const totalLocations = locations.length;
  const totalPaths = allPaths.length;
  
  const chartData = [
    { name: 'Tier 1 (High)', value: locationsByTier.tier1 + pathsByTier.tier1, color: '#059669' },
    { name: 'Tier 2 (Medium)', value: locationsByTier.tier2 + pathsByTier.tier2, color: '#d97706' },
    { name: 'Tier 3 (Low)', value: locationsByTier.tier3 + pathsByTier.tier3, color: '#dc2626' },
  ];
  
  const dataQualityScore = Math.round(
    ((locationsByTier.tier1 + pathsByTier.tier1) * 100 + 
     (locationsByTier.tier2 + pathsByTier.tier2) * 60 + 
     (locationsByTier.tier3 + pathsByTier.tier3) * 20) / 
    ((totalLocations + totalPaths) * 100) * 100
  );
  
  return (
    <div className="bg-white p-6 rounded-lg shadow-lg">
      <h2 className="text-xl font-bold mb-4">Data Quality Dashboard</h2>
      
      <div className="grid grid-cols-2 gap-6">
        {/* Summary Stats */}
        <div>
          <div className="mb-4">
            <div className="text-3xl font-bold text-blue-600">{dataQualityScore}%</div>
            <div className="text-sm text-gray-600">Overall Data Quality Score</div>
          </div>
          
          <div className="space-y-3">
            <div className="border-l-4 border-green-600 pl-3">
              <div className="text-2xl font-bold">{locationsByTier.tier1 + pathsByTier.tier1}</div>
              <div className="text-xs text-gray-600">High Confidence Data Points</div>
            </div>
            
            <div className="border-l-4 border-amber-600 pl-3">
              <div className="text-2xl font-bold">{locationsByTier.tier2 + pathsByTier.tier2}</div>
              <div className="text-xs text-gray-600">Medium Confidence - Need Review</div>
            </div>
            
            <div className="border-l-4 border-red-600 pl-3">
              <div className="text-2xl font-bold text-red-600">{locationsByTier.tier3 + pathsByTier.tier3}</div>
              <div className="text-xs text-gray-600">Low Confidence - PRIORITY</div>
            </div>
          </div>
        </div>
        
        {/* Chart */}
        <div>
          <ResponsiveContainer width="100%" height={200}>
            <PieChart>
              <Pie
                data={chartData}
                dataKey="value"
                nameKey="name"
                cx="50%"
                cy="50%"
                outerRadius={80}
                label
              >
                {chartData.map((entry, index) => (
                  <Cell key={`cell-${index}`} fill={entry.color} />
                ))}
              </Pie>
              <Tooltip />
              <Legend />
            </PieChart>
          </ResponsiveContainer>
        </div>
      </div>
      
      {/* Action Items */}
      <div className="mt-6 p-4 bg-amber-50 border-l-4 border-amber-500 rounded">
        <h3 className="font-bold text-sm mb-2">Intern Action Items:</h3>
        <ul className="text-sm space-y-1">
          <li>✓ Validate {pathsByTier.tier3} Tier 3 flow paths through stakeholder workshops</li>
          <li>✓ Cross-check {locationsByTier.tier3} Tier 3 locations with municipal records</li>
          <li>✓ Document all evidence sources in validation notes</li>
          <li>✓ Target: 80%+ data quality score before production deployment</li>
        </ul>
      </div>
    </div>
  );
}
```

---

## Phase 2: Full DIGO Platform (Weeks 3-6)

### Week 3: Backend Integration

**Goal**: Replace JSON files with real database & API.

**Stack**: Supabase (Postgres + Auth)

```bash
npm install @supabase/supabase-js
```

**Setup Supabase**:
1. Create account at supabase.com
2. Create new project
3. Create tables:

```sql
-- Locations table
CREATE TABLE locations (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  lat DECIMAL(9,6) NOT NULL,
  lng DECIMAL(9,6) NOT NULL,
  type TEXT NOT NULL,
  visitors INTEGER NOT NULL,
  capacity INTEGER,
  category TEXT,
  destination TEXT NOT NULL,
  -- Evidence Quality Fields
  evidence_tier TEXT CHECK (evidence_tier IN ('tier1', 'tier2', 'tier3')) NOT NULL,
  evidence_sources TEXT[] NOT NULL,  -- Array of source descriptions
  validated_by TEXT NOT NULL,
  validation_date DATE NOT NULL,
  evidence_notes TEXT
);

-- Flows table
CREATE TABLE flows (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  pressure TEXT NOT NULL,
  destination TEXT NOT NULL,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Flow paths table
CREATE TABLE flow_paths (
  id SERIAL PRIMARY KEY,
  flow_id TEXT REFERENCES flows(id),
  from_location TEXT REFERENCES locations(id),
  to_location TEXT REFERENCES locations(id),
  visitors INTEGER NOT NULL,
  transport_mode TEXT NOT NULL,
  -- Evidence Quality Fields
  evidence_tier TEXT CHECK (evidence_tier IN ('tier1', 'tier2', 'tier3')) NOT NULL,
  evidence_sources TEXT[] NOT NULL,
  validated_by TEXT NOT NULL,
  validation_date DATE NOT NULL,
  evidence_notes TEXT
);

-- Evidence Quality View (helpful for dashboards)
CREATE VIEW evidence_quality_summary AS
SELECT 
  destination,
  evidence_tier,
  COUNT(*) as count,
  'location' as data_type
FROM locations
GROUP BY destination, evidence_tier
UNION ALL
SELECT 
  f.destination,
  fp.evidence_tier,
  COUNT(*) as count,
  'flow_path' as data_type
FROM flow_paths fp
JOIN flows f ON fp.flow_id = f.id
GROUP BY f.destination, fp.evidence_tier;
```

**Create API Route**:

**File**: `src/app/api/flows/[destination]/route.ts`

```typescript
import { createClient } from '@supabase/supabase-js';
import { NextResponse } from 'next/server';

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
);

export async function GET(
  request: Request,
  { params }: { params: { destination: string } }
) {
  const { data: flows, error } = await supabase
    .from('flows')
    .select(`
      *,
      flow_paths (
        from_location,
        to_location,
        visitors,
        transport_mode
      )
    `)
    .eq('destination', params.destination);

  if (error) {
    return NextResponse.json({ error: error.message }, { status: 500 });
  }

  return NextResponse.json(flows);
}
```

### Week 4: Authentication & RBAC

**Goal**: Add login, user roles (municipality admin, stakeholder viewer, DIGO facilitator)

**Use**: Supabase Auth + Row Level Security (RLS)

**RLS Policy Example**:

```sql
-- Only municipality admins can edit flows
CREATE POLICY "Municipality admins can update flows"
ON flows FOR UPDATE
USING (
  auth.jwt() ->> 'role' = 'municipality_admin'
  AND destination = (auth.jwt() ->> 'municipality')
);

-- All authenticated users can view
CREATE POLICY "Authenticated users can view flows"
ON flows FOR SELECT
USING (auth.role() = 'authenticated');
```

### Week 5: Municipal Action Plans

**Tasks**:
1. Create action plans table
2. Link action plans to flows
3. Build action plan UI
4. Add PDF export

**New Page**: `src/app/municipal/page.tsx`

### Week 6: Analytics & Charts

**Tasks**:
1. Add Recharts visualizations (pressure analysis, economics, inclusivity)
2. Build analytics dashboard
3. Add export functionality

---

## Phase 3: Production Deployment (Weeks 7-8)

### Week 7: Testing & Optimization

**Tasks**:
1. Write tests (Jest + React Testing Library)
2. Lighthouse audit & performance optimization
3. Mobile responsiveness
4. Error handling & edge cases

### Week 8: Deployment

**Stack**: Vercel (Frontend) + Supabase (Backend)

```bash
# 1. Deploy to Vercel
npm install -g vercel
vercel login
vercel

# 2. Set environment variables in Vercel dashboard
# NEXT_PUBLIC_SUPABASE_URL
# NEXT_PUBLIC_SUPABASE_ANON_KEY
# SUPABASE_SERVICE_ROLE_KEY

# 3. Configure custom domain (optional)
# digo.yourdomain.com
```

---

## Learning Resources

### Must-Read Documentation
1. **Next.js App Router**: https://nextjs.org/docs/app
2. **React Leaflet**: https://react-leaflet.js.org/
3. **Supabase**: https://supabase.com/docs
4. **TypeScript**: https://www.typescriptlang.org/docs/

### Recommended Tutorials
1. **Next.js + Leaflet**: "Building a Next.js Map Application" (YouTube)
2. **Supabase Auth**: Official Supabase Auth Guide
3. **TypeScript Best Practices**: "TypeScript Do's and Don'ts"

---

## Code Quality Standards

### TypeScript Rules
- ✅ All components must have TypeScript types
- ✅ No `any` types (use `unknown` if unsure)
- ✅ Use interfaces for data shapes
- ✅ Use enums or literal types for fixed values

### React Best Practices
- ✅ Use Server Components by default (mark Client Components with `'use client'`)
- ✅ Keep components small (<200 lines)
- ✅ Extract reusable logic to custom hooks
- ✅ Use proper loading and error states

### Git Workflow
```bash
# Feature branch naming
git checkout -b feature/map-legend
git checkout -b fix/marker-icons

# Commit messages
git commit -m "feat: add destination selector component"
git commit -m "fix: resolve Leaflet icon rendering issue"
git commit -m "docs: update setup instructions"

# Before PR
npm run lint
npm run build
npm test
```

---

## Weekly Check-ins

**Every Friday**:
1. Demo what you built this week
2. Show any blockers or questions
3. Plan next week's tasks
4. Code review (I'll review your PRs)

**Communication**:
- Slack/Email for async questions
- GitHub Issues for bugs/features
- Weekly 1:1 video call (30 min)

---

## Success Criteria

### Phase 1 Complete When:
- ✅ Map loads without errors
- ✅ Pokhara & Madi both work
- ✅ Flow lines render correctly
- ✅ Popups show accurate data
- ✅ Legend is clear and accurate
- ✅ Code is on GitHub
- ✅ README has setup instructions

### Phase 2 Complete When:
- ✅ Data loads from Supabase
- ✅ Users can log in (email/password)
- ✅ Municipality admins can edit flows
- ✅ Action plans display correctly
- ✅ Analytics charts work

### Phase 3 Complete When:
- ✅ App deployed to Vercel
- ✅ Custom domain configured
- ✅ Lighthouse score >90
- ✅ No console errors in production
- ✅ Documentation complete

---

## Phase 4: AI Intelligence Layer (Advanced - Weeks 9-12)

**Note**: This is an advanced feature. Complete Phases 1-3 first before attempting.

### Overview: RAG-Powered Workshop Intelligence

Add AI-powered retrieval features to help facilitators prepare faster and smarter. See **`RAG_INTELLIGENCE_GUIDE.md`** for complete implementation details.

### Three Intelligence Features

#### 1. Workshop Preparation Intelligence 📊
**What it does**: Natural language queries over tourism statistics and evidence sources.

**Example**: 
```
Query: "What does NTB data show about Lakeside visitor volumes in Q3 2025?"

Answer: "According to NTB statistics for Q3 2025, Lakeside recorded 45,000 
tourist arrivals, a 12% increase from Q3 2024. Hotel occupancy averaged 68%..."

Sources: [NTB Q3 2025 Report, Pokhara Hotel Association Data]
```

**Tech Stack**: OpenAI GPT-4 + Embeddings + Supabase pgvector

**Time Saved**: 2+ hours per workshop on data gathering

#### 2. Policy Alignment Checking ✅
**What it does**: Automatically validate if action plans align with policy frameworks.

**Example**:
```
Query: "Does this homestay initiative align with Nepal Tourism Strategy?"

Answer: "Yes, strongly aligns with Strategic Pillar 3 (Community-Based Tourism) 
and gender equity goals. Recommendation: Add monitoring indicators..."

Sources: [Nepal Tourism Strategy 2022-2032, DIGO Criteria Document]
```

**Document Corpus**: National Tourism Strategy, MoCTCA policies, SDC frameworks, DIGO criteria

#### 3. SDC Reporting Assistance 📝
**What it does**: Auto-generate SDC progress report drafts from platform data.

**Example**:
```
Query: "Draft Q2 2026 progress report for Madi RM"

Output: [Full formatted report with metrics, outcomes, evidence quality scores]

Sources: [Workshop records, flow data, action plan tracking, stakeholder logs]
```

**Time Saved**: 6+ hours per quarterly report

### Implementation Timeline

**Week 9-10: Document Ingestion**
- Set up Supabase pgvector
- Build document upload API with PDF parsing
- Create embedding generation pipeline
- Upload initial document corpus (20-50 policy docs, statistics)

**Week 11: RAG Query System**
- Implement vector similarity search
- Build RAG query API with GPT-4
- Add source citation extraction
- Test retrieval accuracy

**Week 12: User Interface**
- Create RAG query interface component
- Build document uploader UI
- Add evidence quality dashboard integration
- User testing with DIGO facilitators

### Tech Stack Decision

**Option A: OpenAI (Recommended for Production)**
```bash
npm install openai
# Cost: ~$2-5/month for active DIGO usage
# Pros: Best quality, fast, reliable
# Cons: Requires API key, small ongoing cost
```

**Option B: Open Source (For Learning/Testing)**
```bash
npm install ollama @xenova/transformers
# Cost: $0
# Pros: Free, runs locally, privacy
# Cons: Slower, needs GPU, lower quality
```

### Success Criteria

- ✅ 80%+ of facilitator queries answered with sources
- ✅ Query response time < 10 seconds
- ✅ Policy alignment checks catch 95%+ of conflicts
- ✅ SDC report generation reduces time from 8h → 2h
- ✅ Zero errors in source citations

### Files to Create

```
src/
├── app/
│   ├── api/
│   │   └── rag/
│   │       ├── query/route.ts         # RAG query endpoint
│   │       ├── upload/route.ts        # Document upload
│   │       └── generate-report/route.ts
│   └── rag/
│       └── page.tsx                    # RAG interface page
├── components/
│   └── rag/
│       ├── RAGQueryInterface.tsx      # Main query UI
│       ├── DocumentUploader.tsx       # Upload documents
│       └── ReportGenerator.tsx        # SDC report drafts
└── lib/
    └── rag/
        ├── embeddings.ts              # Embedding utils
        ├── chunking.ts                # Text chunking
        └── prompts.ts                 # System prompts
```

### Database Schema

```sql
-- See RAG_INTELLIGENCE_GUIDE.md for full schema

CREATE TABLE rag_documents (
  id UUID PRIMARY KEY,
  title TEXT NOT NULL,
  source_type TEXT NOT NULL,
  source_org TEXT NOT NULL,
  file_url TEXT,
  metadata JSONB
);

CREATE TABLE rag_chunks (
  id UUID PRIMARY KEY,
  document_id UUID REFERENCES rag_documents(id),
  chunk_text TEXT NOT NULL,
  embedding vector(1536),  -- OpenAI embeddings
  metadata JSONB
);
```

### Learning Resources

- **RAG Fundamentals**: "What is Retrieval-Augmented Generation?" (Pinecone blog)
- **pgvector Guide**: Supabase pgvector documentation
- **OpenAI Embeddings**: OpenAI embeddings API guide
- **LangChain Tutorial**: Building RAG applications with LangChain

**Read the full guide**: `RAG_INTELLIGENCE_GUIDE.md`

---

## Questions?

**Before coding**:
1. Read this document fully
2. Set up your development environment
3. Run the prototype and understand how it works
4. Ask clarifying questions about architecture decisions

**When stuck**:
1. Check documentation first
2. Google the error message
3. Ask ChatGPT/Claude for debugging help
4. If still blocked after 1 hour, ping me

---

## 📚 Additional Documentation

This project has comprehensive documentation to guide you:

### Core Guides
- **INTERN_PROJECT.md** (this file) - Your main onboarding guide with week-by-week instructions
- **DATA_EXPORT_GUIDE.md** - Sample data files, evidence quality examples, and data quality workflow
- **RAG_INTELLIGENCE_GUIDE.md** - AI-powered workshop intelligence implementation (Phase 4)
- **IMPLEMENTATION_SUMMARY.md** - High-level overview of complete feature set and architecture
- **ARCHITECTURE.md** - Detailed system architecture diagrams, data flows, and component structure

### Quick Navigation

**Just starting?** → Read this file (INTERN_PROJECT.md) from top to bottom, start with Week 1

**Need sample data?** → Check DATA_EXPORT_GUIDE.md for ready-to-use JSON files

**Building RAG features?** → See RAG_INTELLIGENCE_GUIDE.md for complete implementation

**Want big picture?** → Read IMPLEMENTATION_SUMMARY.md for full feature overview

**Understanding architecture?** → Check ARCHITECTURE.md for system diagrams and data flows

---

**Good luck! 🚀**

Looking forward to seeing your map visualizer next week.

*Remember: You're building something that helps real communities manage tourism sustainably. Every line of code you write contributes to Nepal's tourism development and the UN Sustainable Development Goals!* 🌍
