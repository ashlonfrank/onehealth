# 1health.ai Architecture

## System Overview

1health.ai is a localhost-only health dashboard with an AI chat agent. All data stays on the user's machine.

```
┌─────────────────────────────────────────────────────────────────┐
│                        USER'S MACHINE                           │
│                                                                 │
│  ┌─────────────┐    ┌──────────────────────────────────────┐   │
│  │             │    │           Next.js App                 │   │
│  │  /data      │    │                                      │   │
│  │  folder     │───▶│  ┌─────────┐    ┌────────────────┐   │   │
│  │             │    │  │ Parsers │───▶│ HealthDataStore│   │   │
│  │ - PDFs      │    │  └─────────┘    └───────┬────────┘   │   │
│  │ - CSVs      │    │                         │            │   │
│  │ - XLSX      │    │           ┌─────────────┴──────┐     │   │
│  └─────────────┘    │           │                    │     │   │
│                     │           ▼                    ▼     │   │
│  ┌─────────────┐    │  ┌─────────────┐    ┌────────────┐   │   │
│  │             │    │  │  Dashboard  │    │    Chat    │   │   │
│  │  .env       │    │  │  (React)    │    │   (React)  │   │   │
│  │             │    │  └─────────────┘    └─────┬──────┘   │   │
│  │ ANTHROPIC_  │    │                           │          │   │
│  │ API_KEY     │    │                           ▼          │   │
│  │             │    │                    ┌────────────┐    │   │
│  └──────┬──────┘    │                    │  /api/chat │    │   │
│         │           │                    └─────┬──────┘    │   │
│         │           └──────────────────────────┼───────────┘   │
│         │                                      │               │
│         │           ┌──────────────────────────▼───────────┐   │
│         └──────────▶│         Claude Agent SDK             │   │
│                     │                                      │   │
│                     │  Tools: Read, Bash, WebSearch,       │   │
│                     │         WebFetch                     │   │
│                     └──────────────────┬───────────────────┘   │
│                                        │                       │
└────────────────────────────────────────┼───────────────────────┘
                                         │
                                         ▼
                              ┌─────────────────────┐
                              │   Anthropic API     │
                              │   (External)        │
                              │                     │
                              │   + WebSearch:      │
                              │   - NIH/PubMed      │
                              │   - Blueprint       │
                              └─────────────────────┘
```

## Data Flow

### 1. Startup / Sync Flow

```
User runs `npm run dev`
         │
         ▼
┌─────────────────────────┐
│ Scan /data folder       │
│ Detect: PDF, CSV, XLSX  │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│ Parse each file         │
│ pdf-parse / papaparse   │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│ Extract biomarkers      │
│ Pattern matching on     │
│ parsed text             │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│ Calculate PhenoAge      │
│ Levine formula          │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│ Store in memory         │
│ HealthDataStore         │
│ (singleton)             │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│ Dashboard renders       │
│ with real data          │
└─────────────────────────┘
```

### 2. Chat Flow

```
User types message
         │
         ▼
┌─────────────────────────┐
│ POST /api/chat          │
│ { message: "..." }      │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│ Build context:          │
│ - User message          │
│ - Health data summary   │
│ - System prompt         │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│ Claude Agent SDK        │
│ query()                 │
│                         │
│ Agent may:              │
│ - Read files            │
│ - WebSearch NIH/PubMed  │
│ - Calculate values      │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│ Return response         │
│ to frontend             │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│ Chat UI displays        │
│ agent response          │
└─────────────────────────┘
```

## Component Breakdown

### Parsers (`lib/parsers/`)

| File | Input | Output |
|------|-------|--------|
| `pdf.ts` | PDF file path | Extracted text (string) |
| `csv.ts` | CSV file path | Array of row objects |
| `xlsx.ts` | XLSX file path | Array of row objects per sheet |

### Extractors (`lib/extractors/`)

| File | Input | Output |
|------|-------|--------|
| `biomarkers.ts` | Raw text | `{ glucose?: number, ldl?: number, ... }` |

Extraction uses regex patterns to find biomarker names + values in lab report text.

### Calculations (`lib/calculations/`)

| File | Input | Output |
|------|-------|--------|
| `phenoage.ts` | Biomarkers + chronological age | `{ phenoAge: number, delta: number }` |

Implements Levine PhenoAge formula. See `docs/HEALTH_KNOWLEDGE.md` for formula.

### Store (`lib/store/`)

| File | Purpose |
|------|---------|
| `health-data.ts` | Singleton store holding parsed + calculated health data |

Methods:
- `loadAllData()` — Scans /data, parses, extracts, calculates
- `getBiomarkers()` — Returns extracted biomarkers
- `getBodyComp()` — Returns DEXA data (body fat, lean mass)
- `getActivity()` — Returns activity data (HRV, RHR, sleep)
- `getPhenoAge()` — Returns calculated phenotypic age
- `getHealthSummary()` — Returns text summary for agent context

### Agent (`lib/agent/`)

| File | Purpose |
|------|---------|
| `health-agent.ts` | Claude Agent SDK configuration |

Configuration:
- System prompt with health expertise
- Allowed tools: Read, Bash, WebSearch, WebFetch
- Verified source list for WebSearch

### API Routes (`app/api/`)

| Route | Method | Purpose |
|-------|--------|---------|
| `/api/chat` | POST | Send message to health agent, get response |

### Components (`components/`)

| Component | Purpose |
|-----------|---------|
| `Dashboard.tsx` | Main dashboard layout with all cards |
| `AgeHeader.tsx` | Chronological age, PhenoAge, delta display |
| `BloodWorkCard.tsx` | Biomarkers with color-coded status |
| `DexaCard.tsx` | Body composition from DEXA |
| `ActivityCard.tsx` | HRV, RHR, sleep from activity tracker |
| `ImprovementsCard.tsx` | Areas to improve with recommendations |
| `Chat.tsx` | Chat interface with message history |
| `SyncButton.tsx` | Manual data resync trigger |

## Type Definitions (`lib/types/`)

```typescript
// health.ts

interface Biomarkers {
  // Levine PhenoAge inputs (required for calculation)
  glucose?: number;          // mg/dL
  creatinine?: number;       // mg/dL
  albumin?: number;          // g/dL
  alkalinePhosphatase?: number; // U/L
  crp?: number;              // mg/L
  lymphocytePercent?: number; // %
  mcv?: number;              // fL
  rdw?: number;              // %
  wbc?: number;              // 10^3/uL

  // Common additional markers
  ldl?: number;
  hdl?: number;
  triglycerides?: number;
  vitaminD?: number;
  // ... extend as needed
}

interface BodyComposition {
  bodyFatPercent?: number;
  leanMassLbs?: number;
  boneDensity?: number;
  visceralFat?: number;
}

interface ActivityData {
  hrv?: number;              // ms
  restingHeartRate?: number; // bpm
  sleepHours?: number;
  sleepScore?: number;
  strain?: number;
}

interface HealthData {
  biomarkers: Biomarkers;
  bodyComp: BodyComposition;
  activity: ActivityData;
  phenoAge: {
    chronologicalAge: number;
    biologicalAge: number;
    delta: number;
  } | null;
  lastSynced: Date;
}

interface Improvement {
  biomarker: string;
  currentValue: number;
  targetValue: number;
  status: 'low' | 'high';
  recommendation: string;
}
```

## Security Considerations

1. **API key in .env only** — Never committed, never logged
2. **No external data transmission** — Only Anthropic API calls
3. **No persistent storage** — Data re-parsed each session
4. **User owns their data** — Files stay in /data, never uploaded
5. **Localhost only** — Not designed for deployment

## Future Extensions (Out of Scope for V1)

- OAuth integrations (Whoop API, Apple HealthKit)
- Data persistence (SQLite or JSON cache)
- Multi-user support
- Trend tracking over time
- Export reports as PDF
