# Data Export Guide: Prototype → Standalone Project

## Quick Export for Intern

Your intern needs clean JSON files. Here's how to extract data from the current prototype.

---

## Option 1: Use These Ready-to-Copy Files

### Pokhara Locations (`pokhara-locations.json`)

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
    "id": "bus-park",
    "name": "Tourist Bus Park",
    "lat": 28.2096,
    "lng": 83.9856,
    "type": "entry",
    "visitors": 3800,
    "category": "Entry Point",
    "evidenceQuality": {
      "tier": "tier2",
      "sources": ["Bus operator association estimates", "Ticket counter surveys"],
      "validated_by": "Transport Stakeholder Workshop",
      "validation_date": "2026-04-18"
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
      "tier": "tier1",
      "sources": ["Payment gateway transactions from local businesses", "Hotel occupancy records", "Government tourism survey data"],
      "validated_by": "Multi-stakeholder validation (Hotels, Restaurants, Municipal Office)",
      "validation_date": "2026-04-22"
    }
  },
  {
    "id": "peace-pagoda",
    "name": "World Peace Pagoda",
    "lat": 28.1994,
    "lng": 83.9408,
    "type": "attraction",
    "visitors": 5400,
    "category": "Cultural Site",
    "evidenceQuality": {
      "tier": "tier3",
      "sources": ["Boat operator estimate"],
      "validated_by": "Single stakeholder interview",
      "validation_date": "2026-04-17",
      "notes": "PRIORITY: Needs validation through guide association and temple management records"
    }
  },
  {
    "id": "devi-falls",
    "name": "Devi's Falls",
    "lat": 28.1908,
    "lng": 83.9596,
    "type": "attraction",
    "visitors": 6200,
    "category": "Natural Attraction",
    "evidenceQuality": {
      "tier": "tier1",
      "sources": ["Ticket sales records from entry gate", "Municipal tourism office data", "Annual visitor log"],
      "validated_by": "Site Management & Municipality",
      "validation_date": "2026-04-16"
    }
  },
  {
    "id": "gupteshwor",
    "name": "Gupteshwor Mahadev Cave",
    "lat": 28.1906,
    "lng": 83.9602,
    "type": "attraction",
    "visitors": 4800,
    "category": "Cultural Site"
  },
  {
    "id": "hotel-zone",
    "name": "Hotel Zone (Lakeside)",
    "lat": 28.2101,
    "lng": 83.9601,
    "type": "accommodation",
    "visitors": 7200,
    "category": "Accommodation"
  },
  {
    "id": "old-bazaar",
    "name": "Old Bazaar (Shopping)",
    "lat": 28.2128,
    "lng": 83.9856,
    "type": "service",
    "visitors": 5100,
    "category": "Shopping/Dining"
  },
  {
    "id": "mahendra-cave",
    "name": "Mahendra Cave",
    "lat": 28.2456,
    "lng": 83.9719,
    "type": "attraction",
    "visitors": 3200,
    "category": "Natural Attraction"
  },
  {
    "id": "sarangkot",
    "name": "Sarangkot Viewpoint",
    "lat": 28.2446,
    "lng": 83.9497,
    "type": "attraction",
    "visitors": 4900,
    "category": "Viewpoint"
  },
  {
    "id": "trek-start",
    "name": "Annapurna Trek Base",
    "lat": 28.2142,
    "lng": 83.9512,
    "type": "service",
    "visitors": 3200,
    "category": "Trekking Service"
  }
]
```

### Pokhara Flows (`pokhara-flows.json`)

```json
[
  {
    "id": "flow-1",
    "name": "Airport Arrival → Lakeside",
    "paths": [
      { 
        "from": "airport", 
        "to": "hotel-zone", 
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
        "from": "hotel-zone", 
        "to": "lakeside", 
        "visitors": 3200, 
        "mode": "walk",
        "evidenceQuality": {
          "tier": "tier2",
          "sources": ["Hotel concierge observations", "Tour guide estimates"],
          "validated_by": "Hospitality Workshop",
          "validation_date": "2026-04-20"
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
    "id": "flow-2",
    "name": "Bus Park → Budget Circuit",
    "paths": [
      { "from": "bus-park", "to": "old-bazaar", "visitors": 2200, "mode": "walk" },
      { "from": "old-bazaar", "to": "lakeside", "visitors": 1800, "mode": "walk" },
      { "from": "lakeside", "to": "fewa-lake", "visitors": 1600, "mode": "walk" }
    ],
    "pressure": "low",
    "destination": "pokhara"
  },
  {
    "id": "flow-3",
    "name": "Lakeside → Caves Circuit",
    "paths": [
      { "from": "lakeside", "to": "devi-falls", "visitors": 4100, "mode": "taxi" },
      { "from": "devi-falls", "to": "gupteshwor", "visitors": 3800, "mode": "walk" },
      { "from": "gupteshwor", "to": "lakeside", "visitors": 3500, "mode": "taxi" }
    ],
    "pressure": "medium",
    "destination": "pokhara"
  },
  {
    "id": "flow-4",
    "name": "Sarangkot Sunrise Circuit",
    "paths": [
      { "from": "hotel-zone", "to": "sarangkot", "visitors": 2400, "mode": "taxi" },
      { "from": "sarangkot", "to": "lakeside", "visitors": 2200, "mode": "taxi" }
    ],
    "pressure": "low",
    "destination": "pokhara"
  },
  {
    "id": "flow-5",
    "name": "Peace Pagoda Trail",
    "paths": [
      { "from": "lakeside", "to": "peace-pagoda", "visitors": 3600, "mode": "hike" },
      { "from": "peace-pagoda", "to": "fewa-lake", "visitors": 3200, "mode": "boat" }
    ],
    "pressure": "high",
    "destination": "pokhara"
  },
  {
    "id": "flow-7",
    "name": "Fewa Lake Convergence (Overcrowded)",
    "paths": [
      { 
        "from": "hotel-zone", 
        "to": "fewa-lake", 
        "visitors": 5200, 
        "mode": "walk",
        "evidenceQuality": {
          "tier": "tier2",
          "sources": ["Multiple hotel manager estimates", "Lakeside guide observations"],
          "validated_by": "Cross-validated in stakeholder workshop",
          "validation_date": "2026-04-20"
        }
      },
      { 
        "from": "peace-pagoda", 
        "to": "fewa-lake", 
        "visitors": 2100, 
        "mode": "boat",
        "evidenceQuality": {
          "tier": "tier1",
          "sources": ["Boat operator ticket records", "Boat association daily logs"],
          "validated_by": "Boat Operators Association",
          "validation_date": "2026-04-19"
        }
      },
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
          "notes": "HIGH PRIORITY for verification given critical pressure rating - recommend pedestrian count"
        }
      }
    ],
    "pressure": "critical",
    "destination": "pokhara"
  }
]
```

### Madi Locations (`madi-locations.json`)

```json
[
  {
    "id": "highway-junction",
    "name": "Mahendra Highway Junction",
    "lat": 27.6289,
    "lng": 84.5342,
    "type": "entry",
    "visitors": 1200,
    "category": "Entry Point",
    "evidenceQuality": {
      "tier": "tier3",
      "sources": ["Local taxi driver estimate"],
      "validated_by": "Single stakeholder interview",
      "validation_date": "2026-04-10",
      "notes": "Needs validation with highway police or traffic monitoring data"
    }
  },
  {
    "id": "sauraha-entrance",
    "name": "Sauraha Entry (via Chitwan)",
    "lat": 27.5789,
    "lng": 84.4982,
    "type": "entry",
    "visitors": 2100,
    "category": "Entry Point",
    "evidenceQuality": {
      "tier": "tier2",
      "sources": ["Chitwan tour operator estimates", "Madi guide observations"],
      "validated_by": "Cross-validated at Madi-Chitwan Joint Workshop",
      "validation_date": "2026-04-12"
    }
  },
  {
    "id": "community-forest",
    "name": "Madi Community Forest",
    "lat": 27.6142,
    "lng": 84.5156,
    "type": "attraction",
    "visitors": 3200,
    "capacity": 2500,
    "category": "Ecotourism",
    "evidenceQuality": {
      "tier": "tier1",
      "sources": ["Community forest entry register", "Guide permit logs", "Municipal tourism office records"],
      "validated_by": "Community Forest Committee & Municipality",
      "validation_date": "2026-04-14"
    }
  },
  {
    "id": "river-point",
    "name": "Rapti River Wildlife Viewing",
    "lat": 27.6089,
    "lng": 84.5289,
    "type": "attraction",
    "visitors": 2800,
    "capacity": 2000,
    "category": "Wildlife Tourism",
    "evidenceQuality": {
      "tier": "tier2",
      "sources": ["Canoe operator daily logs", "Wildlife guide association estimates"],
      "validated_by": "River Tourism Stakeholder Meeting",
      "validation_date": "2026-04-13",
      "notes": "Canoe operators keep records but not systematically digitized"
    }
  },
  {
    "id": "homestay-cluster",
    "name": "Women-Led Homestay Cluster",
    "lat": 27.6198,
    "lng": 84.5289,
    "type": "accommodation",
    "visitors": 650,
    "category": "Community Homestay",
    "evidenceQuality": {
      "tier": "tier1",
      "sources": ["Homestay booking records", "Women's cooperative ledger", "Payment receipts"],
      "validated_by": "Women's Homestay Cooperative",
      "validation_date": "2026-04-15",
      "notes": "Excellent record-keeping by cooperative - model for other locations"
    }
  }
]
```

### Madi Flows (`madi-flows.json`)

```json
[
  {
    "id": "madi-flow-1",
    "name": "Chitwan Package Tour (External Operator)",
    "paths": [
      { 
        "from": "sauraha-entrance", 
        "to": "river-point", 
        "visitors": 1850, 
        "mode": "jeep",
        "evidenceQuality": {
          "tier": "tier2",
          "sources": ["Chitwan tour operator reports", "Jeep driver association logs"],
          "validated_by": "Cross-border stakeholder workshop (Chitwan-Madi)",
          "validation_date": "2026-04-12",
          "notes": "High leakage concern - external operators dominate this flow"
        }
      },
      { 
        "from": "river-point", 
        "to": "community-forest", 
        "visitors": 1700, 
        "mode": "walk",
        "evidenceQuality": {
          "tier": "tier3",
          "sources": ["Tour guide estimate"],
          "validated_by": "Single guide interview",
          "validation_date": "2026-04-11",
          "notes": "PRIORITY: Needs validation with community forest records to understand actual local benefit"
        }
      }
    ],
    "pressure": "critical",
    "destination": "madi",
    "leakage": 82
  },
  {
    "id": "madi-flow-4",
    "name": "Women-Led Homestay Experience",
    "paths": [
      { 
        "from": "sauraha-entrance", 
        "to": "homestay-cluster", 
        "visitors": 550, 
        "mode": "taxi",
        "evidenceQuality": {
          "tier": "tier1",
          "sources": ["Homestay booking records", "Taxi receipt logs", "Women's cooperative guest register"],
          "validated_by": "Women's Homestay Cooperative & Local Taxi Association",
          "validation_date": "2026-04-15"
        }
      },
      { 
        "from": "homestay-cluster", 
        "to": "community-forest", 
        "visitors": 480, 
        "mode": "walk",
        "evidenceQuality": {
          "tier": "tier1",
          "sources": ["Homestay activity logs", "Community forest entry register cross-check"],
          "validated_by": "Women's Cooperative & Forest Committee Joint Review",
          "validation_date": "2026-04-15",
          "notes": "Excellent collaboration between cooperative and forest management - model for data collection"
        }
      }
    ],
    "pressure": "low",
    "destination": "madi",
    "leakage": 12
  }
]
```

---

## Option 2: Generate from Current Code

If you have the prototype running, extract data programmatically:

### Extract Script

Create `scripts/export-data.ts`:

```typescript
import fs from 'fs';
import { pokharaLocations, visitorFlows } from '../src/app/components/VisitorFlowMap';

// Export locations
fs.writeFileSync(
  'public/data/pokhara-locations.json',
  JSON.stringify(pokharaLocations, null, 2)
);

// Export flows
const flowsData = visitorFlows.map(flow => ({
  id: flow.id,
  name: flow.name,
  paths: flow.paths,
  pressure: flow.pressure,
  destination: 'pokhara'
}));

fs.writeFileSync(
  'public/data/pokhara-flows.json',
  JSON.stringify(flowsData, null, 2)
);

console.log('✓ Data exported successfully');
```

Run: `npx tsx scripts/export-data.ts`

---

## Option 3: CSV Format (For Workshop Import)

If workshops use Excel/Google Sheets, provide CSV templates:

### Locations CSV

```csv
id,name,lat,lng,type,visitors,capacity,category
airport,Pokhara Airport,28.2009,83.9821,entry,4200,,Entry Point
fewa-lake,Fewa Lake Waterfront,28.2096,83.9586,attraction,12000,8000,Natural Attraction
lakeside,Lakeside (Baidam),28.2089,83.9595,service,9500,,Tourism Hub
```

### Flows CSV

```csv
flow_id,flow_name,from_location,to_location,visitors,transport_mode,pressure
flow-1,Airport Arrival → Lakeside,airport,hotel-zone,3500,taxi,medium
flow-1,Airport Arrival → Lakeside,hotel-zone,lakeside,3200,walk,medium
flow-7,Fewa Lake Convergence,lakeside,fewa-lake,4700,walk,critical
```

**Intern Task**: Write CSV importer that converts these to JSON.

---

## Data Validation Schema

For the intern to validate imported data:

```typescript
import { z } from 'zod';

const LocationSchema = z.object({
  id: z.string(),
  name: z.string(),
  lat: z.number().min(-90).max(90),
  lng: z.number().min(-180).max(180),
  type: z.enum(['entry', 'attraction', 'service', 'accommodation', 'challenge']),
  visitors: z.number().int().positive(),
  capacity: z.number().int().positive().optional(),
  category: z.string()
});

const FlowSchema = z.object({
  id: z.string(),
  name: z.string(),
  paths: z.array(z.object({
    from: z.string(),
    to: z.string(),
    visitors: z.number().int().positive(),
    mode: z.enum(['walk', 'taxi', 'boat', 'jeep', 'hike'])
  })),
  pressure: z.enum(['low', 'medium', 'high', 'critical']),
  destination: z.enum(['pokhara', 'madi'])
});

// Usage
const locations = LocationSchema.array().parse(jsonData);
```

---

## Real Workshop Data Integration

### When You Have Actual Workshop Data

1. **Receive from DIGO team** as Excel file
2. **Convert to CSV** using Excel export
3. **Validate coordinates** using Google Maps / OpenStreetMap
4. **Import to JSON** using conversion script
5. **Load into prototype** to visualize
6. **Stakeholder validation** in workshop
7. **Export validated data** for production

### Coordinate Validation

```typescript
// Verify coordinates are in Nepal
function isInNepal(lat: number, lng: number): boolean {
  return (
    lat >= 26.3 && lat <= 30.5 &&  // Nepal latitude range
    lng >= 80.0 && lng <= 88.3      // Nepal longitude range
  );
}

// Check if location exists on map
async function geocodeLocation(name: string): Promise<[number, number]> {
  const response = await fetch(
    `https://nominatim.openstreetmap.org/search?q=${encodeURIComponent(name + ', Pokhara, Nepal')}&format=json`
  );
  const data = await response.json();
  return [parseFloat(data[0].lat), parseFloat(data[0].lon)];
}
```

---

## Summary: What to Give Your Intern

**Minimal start package**:
```
📁 intern-data-package/
├── pokhara-locations.json      (12 locations)
├── pokhara-flows.json           (6 sample flows)
├── madi-locations.json          (5 locations)
├── madi-flows.json              (2 sample flows)
└── README.md                    (this guide)
```

**Full package for production**:
```
📁 digo-data-export/
├── data/
│   ├── pokhara-locations.json   (12 locations)
│   ├── pokhara-flows.json       (19 flows - full dataset)
│   ├── madi-locations.json      (9 locations)
│   ├── madi-flows.json          (8 flows - full dataset)
│   └── action-plans.json        (municipal action plans)
├── schemas/
│   ├── location.schema.json     (JSON Schema for validation)
│   └── flow.schema.json
├── scripts/
│   ├── validate-data.ts         (Data validation script)
│   └── csv-to-json.ts           (CSV converter)
└── README.md
```

**Give them**: The minimal package to start, full package as reference.

---

## Next Step for Intern

After receiving data files:

```bash
# In their standalone project
mkdir public/data
# Copy JSON files to public/data/
npm run dev
# Map should load with real Pokhara data
```

Then iterate:
1. Week 1: Display data correctly
2. Week 2: Add Madi switching
3. Week 3: CSV import for workshop data
4. Week 4: Connect to Supabase

---

## Evidence Quality Tier System

All data in DIGO must be tagged with evidence quality tiers. Your goal as an intern is to **move all Tier 3 data to Tier 2+** through validation.

### Tier Definitions

**Tier 1 - High Confidence** ✅
- **Validated by multiple independent sources**
- **Examples of sources**:
  - Official entry/exit records (airports, checkpoints, ticketed sites)
  - Hotel occupancy records from hotel management systems
  - Payment gateway transaction data
  - Government survey data (Nepal Tourism Board, municipality)
  - Cooperative/association ledgers with systematic record-keeping
- **When to use**: Data cross-verified by 3+ independent sources or from official government records
- **Color**: Green (#059669)

**Tier 2 - Medium Confidence** ⚠️
- **Stakeholder consensus, needs ongoing monitoring**
- **Examples of sources**:
  - Hotel/business association collective estimates
  - Tour guide direct records (not individual estimates)
  - Business owner observations validated in workshops
  - Cross-validated estimates from multiple workshop participants
- **When to use**: 2+ stakeholders agree on estimates, or single reliable source with track record
- **Color**: Amber (#d97706)

**Tier 3 - Low Confidence** 🚨
- **Single source, needs urgent verification**
- **Examples of sources**:
  - Individual business owner estimate (not cross-checked)
  - Seasonal observation from single guide
  - Anecdotal report
  - "I think about X tourists visit per day"
- **When to use**: Only have one source, or data is based on impression rather than records
- **Color**: Red (#dc2626)
- **Action**: FLAG for workshop validation

### Data Quality Workflow for Interns

```
1. COLLECT → Gather data from any available source (Tier 3 is okay to start!)
2. TAG → Mark with honest evidence tier
3. VALIDATE → Bring Tier 3 items to stakeholder workshops
4. UPGRADE → Update tier when validation improves confidence
5. DOCUMENT → Record all sources and validation process
```

### Example: Upgrading Data Quality

**Scenario**: Fewa Lake visitor count

**Initial State (Tier 3)**:
```json
{
  "id": "fewa-lake",
  "visitors": 12000,
  "evidenceQuality": {
    "tier": "tier3",
    "sources": ["Boat operator guess"],
    "validated_by": "Single interview",
    "validation_date": "2026-04-01",
    "notes": "Very rough estimate, needs validation"
  }
}
```

**After Workshop (Tier 2)**:
```json
{
  "id": "fewa-lake",
  "visitors": 11500,
  "evidenceQuality": {
    "tier": "tier2",
    "sources": [
      "Boat association collective estimate", 
      "Lakeside hotel manager observations",
      "Restaurant owner consensus"
    ],
    "validated_by": "Lakeside Stakeholder Workshop (15 participants)",
    "validation_date": "2026-04-20",
    "notes": "Adjusted from 12000 to 11500 based on workshop discussion"
  }
}
```

**After Cross-Check with Records (Tier 1)**:
```json
{
  "id": "fewa-lake",
  "visitors": 11200,
  "evidenceQuality": {
    "tier": "tier1",
    "sources": [
      "Boat ticket sales records (Boat Association)",
      "Hotel guest activity logs",
      "Municipal tourism office visitor survey"
    ],
    "validated_by": "Municipality & DIGO cross-verification",
    "validation_date": "2026-05-05",
    "notes": "Final validation with actual records, slight adjustment to 11200"
  }
}
```

### CSV Template with Evidence Quality

If collecting data in Excel/Google Sheets, use this format:

```csv
id,name,lat,lng,type,visitors,category,evidence_tier,evidence_sources,validated_by,validation_date,evidence_notes
airport,Pokhara Airport,28.2009,83.9821,entry,4200,Entry Point,tier1,"Official airport records; Immigration data",Civil Aviation Authority,2026-04-15,Cross-referenced with national stats
peace-pagoda,World Peace Pagoda,28.1994,83.9408,attraction,5400,Cultural Site,tier3,Boat operator estimate,Single interview,2026-04-17,PRIORITY: Needs guide association validation
```

### Quality Targets

**Before Phase 2 (Database Integration)**:
- ❌ No more than 20% Tier 3 data
- ✅ At least 30% Tier 1 data
- ✅ Document all validation workshops

**Before Phase 3 (Production Deployment)**:
- ❌ ZERO Tier 3 data in production
- ✅ At least 50% Tier 1 data
- ✅ Data quality score ≥ 80%

**Data Quality Score Formula**:
```
Score = (Tier1_count × 100 + Tier2_count × 60 + Tier3_count × 20) / (Total_count × 100) × 100
```

### Validation Checklist

Before marking data as Tier 1 or Tier 2:

- [ ] Can you name the specific sources (not just "workshop")?
- [ ] Do you have contact info for validators in case of questions?
- [ ] Are sources independent (not same person/org twice)?
- [ ] For Tier 1: Do you have documentary evidence (logs, records, receipts)?
- [ ] For workshop validation: Did 3+ stakeholders agree?
- [ ] Is the validation date recent (within 3 months)?

### Common Pitfalls

❌ **Don't do this**:
- Marking workshop estimates as Tier 1 (unless backed by records)
- Using same source twice (e.g., "Hotel A estimate" + "Hotel A owner opinion")
- Leaving old Tier 3 data in production
- Forgetting to document validation date

✅ **Do this**:
- Be honest about data limitations (Tier 3 is better than wrong Tier 1!)
- Prioritize upgrading critical pressure points first
- Document who to contact for follow-up validation
- Update tiers as new information comes in

---

## Summary for Intern

**Your data quality mission**:
1. Start collecting data honestly (Tier 3 is okay!)
2. Tag everything with evidence quality
3. Run workshops to upgrade Tier 3 → Tier 2
4. Find documentary sources to upgrade Tier 2 → Tier 1
5. Get to 80%+ quality score before production

**Remember**: The tier system exists to help municipalities make informed decisions. A honest Tier 3 rating that leads to validation is more valuable than an inflated Tier 1 that's actually a guess.

Good luck! 🗺️
