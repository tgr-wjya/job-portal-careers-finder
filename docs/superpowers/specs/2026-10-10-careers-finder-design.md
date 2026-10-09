# Job Portal / Careers Finder Specification

## 1. Goal & Overview
Build a high-performance, verifiable job and internship discovery platform powered by the **TinyFish Web Automation Suite** (Search, Fetch, Agent) and executed on **Bun + Hono + Vite/React + Tailwind CSS**.

The system addresses the [Job Portal / Careers Finder Bounty](file:///home/tgr-wjya/Documents/Projects/job-portal-careers-finder/JOB_PORTAL_CAREERS_FINDER_BOUNTY.md), achieving Tier 3 status (200 points) through purposeful, validated usage of all three TinyFish endpoints:
1. **TinyFish Search (`GET https://api.search.tinyfish.ai`)**: Discovery of official career pages, boards, and autonomous open-web job hunting via domain allowlisting.
2. **TinyFish Fetch (`POST https://api.fetch.tinyfish.ai`)**: Sub-second batch Markdown extraction for static career pages and ATS platforms (Greenhouse, Lever).
3. **TinyFish Agent (`POST https://agent.tinyfish.ai/v1/automation/run` & `run-sse`)**: Goal-directed navigation for dynamic, filter-heavy, or client-rendered hiring portals.

---

## 2. Architecture & Pipeline

```text
User Preferences (Role, Location, Keywords, Seniority, Visa, Companies/Open Discovery)
        │
        ▼
   [Intent Parser]
        │
        ├─────────────────────────────────────────┐
        ▼                                         ▼
[Specific Company Discovery]             [Autonomous Open-Web Discovery]
TinyFish Search: official career URLs    TinyFish Search: domain allowlist
                                         (greenhouse.io, lever.co, ashbyhq.com)
        │                                         │
        └──────────────────┬──────────────────────┘
                           ▼
                  [Extraction Router]
                 /                   \
                ▼                     ▼
     [Static & ATS Boards]      [Dynamic Portals]
       TinyFish Fetch            TinyFish Agent
     (Batch Markdown POST)     (Goal-based Browser)
                \                     /
                 ▼                   ▼
               [Normalizer & Evidence Anchor]
                 • Map to standard JobListing schema
                 • Exact word-for-word quote extraction
                 • Strict unknown/null distinction
                           │
                           ▼
                     [Deduplicator]
                 • Primary: Canonical apply URL hash
                 • Secondary: External source job ID
                 • Tertiary: Slugified company + title + location
                           │
                           ▼
                [Matcher & Deterministic Ranker]
                 • Score (0.0 to 1.0)
                 • Explicit positive reasons backed by quotes
                 • Explicit gaps for missing/unverified criteria
                           │
                           ▼
                  [Delivery Interfaces]
                 • Server-Sent Events (SSE) Stream to Web UI
                 • Standalone Bun CLI Runner (`bin/careers-finder.ts`)
```

---

## 3. Verified TinyFish Endpoint Contracts

### TinyFish Search
- **Endpoint**: `GET https://api.search.tinyfish.ai`
- **Method**: Synchronous HTTP GET.
- **Parameters**: `query`, `purpose`, `location`, `language`, `include_domains`, `recency_minutes`.
- **Purpose**: Discovers target career URLs and performs open-web hiring searches against trusted ATS domains (`greenhouse.io,lever.co,jobs.ashbyhq.com`).

### TinyFish Fetch
- **Endpoint**: `POST https://api.fetch.tinyfish.ai`
- **Method**: Synchronous HTTP POST (up to 10 URLs per batch).
- **Body**: `{ urls: string[], format: "markdown", links: true }`.
- **Latency**: Sub-second (100ms - 1500ms observed in live verification).
- **Purpose**: High-speed, low-cost extraction of job descriptions, compensation details, and direct application links from static and ATS pages.

### TinyFish Agent
- **Endpoint**: `POST https://agent.tinyfish.ai/v1/automation/run` (sync) or `/v1/automation/run-sse` (live events).
- **Method**: HTTP POST with natural language goal and strict JSON schema.
- **Allowed Schema Keywords**: `type`, `properties`, `required`, `items`, `nullable`, `enum`, `format`, `minimum`, `maximum`, `anyOf`.
- **Purpose**: Automated browser navigation for portals with dynamic filters, JavaScript pagination, or anti-bot protections.

---

## 4. Data Schemas

### `JobListing` (Normalized Item)
```typescript
export interface JobListing {
  id: string; // sha256 hash of canonical applyUrl
  title: string;
  company: string;
  location: string;
  workMode: "remote" | "hybrid" | "on-site" | "unknown";
  employmentType: "full-time" | "part-time" | "internship" | "contract" | "unknown";
  seniority: "entry" | "mid" | "senior" | "lead" | "unknown";
  visa: {
    status: "supported" | "not_supported" | "unknown";
    evidence?: string;
  };
  salary: {
    min: number | null;
    max: number | null;
    currency: string | null;
    period: "year" | "month" | "hour" | null;
  } | null;
  postedAt: string | null;
  description: string;
  match: {
    score: number; // 0.0 - 1.0
    reasons: string[];
    gaps: string[];
  };
  applyUrl: string;
  source: {
    url: string;
    type: "company careers" | "Greenhouse" | "Lever" | "Ashby" | "job portal";
    retrievedAt: string;
  };
  evidence: Array<{
    field: "location" | "salary" | "visa" | "seniority" | "role" | "requirements";
    quote: string;
    url: string;
  }>;
}
```

### `SearchCoverage` (Summary Output)
```typescript
export interface SearchCoverage {
  status: "complete" | "partial" | "failed";
  sourcesChecked: number;
  listingsFound: number;
  listingsReturned: number;
  missingFields: string[];
}
```

---

## 5. Matching & Ranking Algorithm
1. **Role / Title Alignment (35%)**: Evaluated using normalized token matching and title taxonomy.
2. **Location & Work Mode (25%)**: Exact match on remote, hybrid, or target geography.
3. **Keyword Overlap (20%)**: Required technologies and tools in requirements. Score = `matched / total_requested`.
4. **Seniority Alignment (10%)**: Level match (`entry`, `mid`, `senior`).
5. **Visa Fit (10%)**: Positive match if visa sponsorship explicitly stated; neutral gap if unmentioned.
6. **Zero Hallucination / Evidence Gate**: A reason is emitted only when backed by an exact quote in the `evidence` array. Missing criteria become explicit entries in `match.gaps`.

---

## 6. UI & Frontend Design Specifications
Guided by [`frontend-design`](file:///home/tgr-wjya/.agents/skills/frontend-design/SKILL.md) and patterns from the [TinyFish Cookbook](https://github.com/tinyfish-io/tinyfish-cookbook):

- **Aesthetic Direction**: Functional telemetry cockpit. High-contrast slate background (`#090d16` / `#0f172a`), deep indigo/cyan accents (`#38bdf8`, `#6366f1`), sharp border definitions (`#1e293b`), and clean tabular layout. Banned: generic rounded AI cards with blurry drop shadows or pastel cream backgrounds.
- **Layout Architecture**:
  - **Left Telemetry Panel (380px)**:
    - Search intent configuration (Roles, Locations, Keywords, Seniority, Visa).
    - Mode toggle: Targeted preset companies vs Autonomous Open-Web Discovery.
    - Live Pipeline Telemetry: Real-time badges indicating active TinyFish endpoint (`SEARCH`, `FETCH`, `AGENT`), request latencies, and extracted listing counts.
  - **Right Results Feed**:
    - Real-time streaming cards rendered immediately as SSE events arrive.
    - Card header: Title, Company, Work Mode tag, Seniority badge, Deterministic Match Score badge (e.g. `92% MATCH`).
    - Evidence Inspector: Collapsible pane displaying word-for-word source quotes for salary, visa, and location.
    - Application Action: Primary button directing straight to the canonical application form.
    - JSON Export: Raw bounty-compliant JSON export modal/download.

---

## 7. CLI Architecture (`bin/careers-finder.ts`)
- Standalone execution using Bun: `bun run bin/careers-finder.ts --role "Backend Engineer" --location "Remote" --keywords "Go,Kubernetes"`
- Options:
  - `--role <titles>` (comma-separated)
  - `--location <locations>`
  - `--keywords <tags>`
  - `--companies <names>`
  - `--open-discovery` (enables autonomous ATS search)
  - `--output <path>` (writes structured JSON file)
- Outputs formatted ANSI terminal cards with evidence quotes and match explanations.

---

## 8. Testing & Verification Plan
1. **Unit Tests (`bun test`)**:
   - `normalizer.test.ts`: Validates salary extraction, work-mode classification, and quote binding across edge cases.
   - `deduplicator.test.ts`: Verifies URL normalization, parameter stripping, and canonical deduplication.
   - `matcher.test.ts`: Verifies deterministic scoring math, reasons generation, and gap identification.
2. **Live Integration Tests**:
   - Test Preset 1: Stripe & Vercel career pages (Search + Fetch).
   - Test Preset 2: Ashby / dynamic portal (Search + Agent).
   - Test Preset 3: Open-Web Discovery for Rust/Go remote roles across Greenhouse.
3. **Build & Quality Gates**:
   - `bun run build`: Verifies zero TypeScript or Vite bundle errors.
   - Lighthouse check: Confirms accessibility, contrast, and performance standards.
