# Job Portal / Careers Finder Bounty

## Goal

Build a TinyFish-powered job and internship finder that pulls live openings
from company career pages and job portals, matches them to user preferences,
and returns fresh, actionable results.

The tool should reduce repeated career-page searches. Results should be clean
enough for a person to use immediately and structured enough for another tool
to consume.

## Submission Requirements

The build must:

1. Include a working demo.
2. Let users set preferences:
   - Role or title
   - Location or remote preference
   - Keywords
   - Seniority
   - Visa or work-authorization needs
3. Pull live openings from multiple company career pages or job portals.
4. Return structured, deduplicated listings with direct application links.
5. Include a short explanation of how TinyFish Search, Fetch, and Agent are used.

## Approval Criteria

Every criterion must pass:

1. TinyFish contributes meaningful discovery and extraction work.
2. Results come from live web data, not hardcoded listings.
3. The workflow supports multiple companies or portals.
4. The workflow accepts different roles, locations, keywords, and preferences.
5. Results are filtered, matched, or ranked instead of dumping raw pages.
6. Each listing includes enough evidence to verify title, company, location,
   freshness, and application URL.
7. Missing or uncertain fields are explicit. The tool never invents job facts.

## TinyFish Usage

Use the lightest endpoint that can answer each step:

1. **Search** discovers official company career pages and relevant job portals
   from user input or a company list.
2. **Fetch** reads discovered listing pages, search-result pages, and job detail
   pages. Fetch should provide clean text, links, metadata, and source URLs.
3. **Agent** handles dynamic portals that require filters, pagination, location
   selection, or interaction before listings appear.

Do not call Agent when Search or Fetch can provide the required data.

## Suggested Workflow

```text
User preferences
        |
        v
Search official career pages and portals
        |
        v
Fetch listing and detail pages
        |
        +--> extract title, company, location, seniority, visa, date, URL
        |
        +--> deduplicate by canonical application URL and stable job ID
        |
        v
Match and rank against preferences
        |
        v
Return structured listings with source evidence
```

## Expected Input

```json
{
  "roles": ["software engineer", "backend engineer"],
  "locations": ["remote", "New York", "London"],
  "keywords": ["TypeScript", "Node.js", "distributed systems"],
  "seniority": ["entry", "mid", "senior"],
  "visa": "sponsorship preferred",
  "companies": ["Vercel", "Stripe", "Notion"],
  "sources": ["company careers", "LinkedIn", "Greenhouse", "Lever"]
}
```

User may provide only some fields. Empty preferences must not produce a
misleading match score.

## Expected Output

```json
{
  "search": {
    "roles": [],
    "locations": [],
    "keywords": [],
    "seniority": [],
    "visa": null,
    "searchedAt": "2026-10-04T00:00:00.000Z"
  },
  "results": [
    {
      "id": "company-job-id-or-canonical-url-hash",
      "title": "Backend Engineer",
      "company": "Example Company",
      "location": "Remote - United States",
      "workMode": "remote",
      "employmentType": "full-time",
      "seniority": "mid",
      "visa": {
        "status": "supported",
        "evidence": "Visa sponsorship available"
      },
      "salary": {
        "min": 120000,
        "max": 160000,
        "currency": "USD",
        "period": "year"
      },
      "postedAt": "2026-10-01",
      "description": "Short source-backed summary.",
      "match": {
        "score": 0.86,
        "reasons": [
          "Role matches backend engineer preference",
          "Remote location matches preference",
          "Node.js appears in requirements"
        ],
        "gaps": []
      },
      "applyUrl": "https://jobs.example.com/jobs/123",
      "source": {
        "url": "https://jobs.example.com/jobs/123",
        "type": "company careers",
        "retrievedAt": "2026-10-04T00:00:00.000Z"
      },
      "evidence": [
        {
          "field": "location",
          "quote": "Remote - United States",
          "url": "https://jobs.example.com/jobs/123"
        }
      ]
    }
  ],
  "coverage": {
    "status": "complete",
    "sourcesChecked": 3,
    "listingsFound": 24,
    "listingsReturned": 8,
    "missingFields": []
  }
}
```

## Matching and Ranking

Matching must be transparent and deterministic enough to explain.

Suggested ranking factors:

- Role or title match
- Required keyword match
- Location and work-mode match
- Seniority match
- Visa fit
- Freshness
- Completeness of listing data

Do not treat an unknown field as a positive match. Use `gaps` when a listing
cannot verify a preference.

Recommended score range: `0.0` to `1.0`.

## Deduplication

Deduplicate listings using this priority:

1. Canonical application URL.
2. Stable external job ID from the source.
3. Normalized company, title, and location when no ID or URL is available.

Keep the strongest source record and preserve alternate source URLs when they
refer to the same opening.

## Freshness and Failure Behavior

- Include retrieval timestamps for every source.
- Preserve source URLs and direct apply URLs.
- Mark stale or undated listings explicitly.
- Continue when one source fails, but expose source-level errors.
- Return a partial result when at least one source provides usable listings.
- Fail clearly when no source returns usable listing content.
- Never invent salary, visa support, location, seniority, or posting dates.

## Demonstration Plan

The demo should use at least three different source shapes, for example:

1. A company career page with direct job detail pages.
2. A Greenhouse or Lever board.
3. A job portal with filters, pagination, or dynamic loading.

Demonstrate at least two preference sets:

- A software-engineering search with remote and keyword preferences.
- An internship or non-engineering search with a different location and
  seniority preference.

For every demonstration, verify:

- Live source pages are used.
- Direct application links work.
- Duplicate listings are removed.
- Results explain why they match.
- Unknown visa, salary, or location data stays unknown.
- The same schema works across all source types.

## Scoring Tiers

The bounty awards points by meaningful endpoint usage:

| Tier | Endpoint count | Points |
|---|---:|---:|
| 1 | 1 endpoint | 50 |
| 2 | 2 endpoints | 100 |
| 3 | 3 endpoints | 200 |

Endpoint count alone is not enough. Each endpoint must materially improve
source discovery, extraction, or matching. Do not add calls only to increase
endpoint count.

## Short TinyFish Explanation

The final README or demo must explain:

- Why Search was needed.
- Which URLs Fetch inspected.
- Which dynamic steps required Agent.
- How source evidence is preserved.
- How failures and stale listings are represented.
- How matching and deduplication work.

## Cookbook and Example References

Use these TinyFish Cookbook projects as implementation references. They are
patterns to adapt, not sources of hardcoded job data.

### Competitor Scout CLI

[competitor-scout-cli](https://github.com/tinyfish-io/tinyfish-cookbook/tree/main/competitor-scout-cli)

Relevant pattern:

- Search discovers relevant URLs.
- Fetch reads multiple URLs in a batch.
- A separate evidence check decides whether the collected material is enough.
- Agent is a fallback for cases where Search and Fetch do not provide enough
  evidence.
- Research runs, errors, and cancellation are persisted.

Adaptation: represent each career source as a research target, retain source
evidence, and keep Agent fallback explicit.

### Research Sentry

[research-sentry](https://github.com/tinyfish-io/tinyfish-cookbook/tree/main/research-sentry)

Relevant pattern:

- One extraction task runs per independent source.
- Sources run concurrently with `Promise.allSettled`.
- Results stream back as each source completes.
- An aggregator deduplicates and ranks results.
- Typed failures remain visible instead of becoming silent empty results.

Adaptation: run one source worker per company career page or portal, stream
partial results, then deduplicate by canonical application URL and rank by
preference match.

### Scholarship Finder

[scholarship-finder](https://github.com/tinyfish-io/tinyfish-cookbook/tree/main/scholarship-finder)

Relevant pattern:

- User query is converted into structured search intent.
- Relevant source URLs are discovered before scraping.
- Multiple browser agents scrape independent sources in parallel.
- Results stream to the interface as agents complete.
- Each result card exposes live source-derived fields.

Adaptation: parse role, location, keywords, seniority, and visa preferences
before discovery. Return job cards with direct apply URLs and evidence.

### Tutor Finder

[tutor-finder](https://github.com/tinyfish-io/tinyfish-cookbook/tree/main/tutor-finder)

Relevant pattern:

- Search discovers source sites from user criteria.
- One Agent runs per discovered site.
- `EventType.PROGRESS` exposes source status.
- `EventType.COMPLETE` delivers structured records.
- Results are fetched live in memory, without stale database records.
- The UI supports comparison after collection.

Adaptation: show source-by-source search progress, then support shortlist and
comparison views for matched jobs. Do not copy its hardcoded discovery fallback
as a substitute for live job data.

### General TinyFish Examples

[TinyFish Agent examples](https://agent.tinyfish.ai/examples)

The examples reinforce these rules:

- Give Agent one focused source task.
- State the exact JSON shape in the goal.
- Ask for evidence URLs with extracted values.
- Keep independent sources in separate runs.
- Do not combine unrelated websites into one Agent goal.
- Use Fetch for clean page reading and Agent for interaction-heavy pages.

## Recommended Job Finder Architecture

Combine the cookbook patterns into this flow:

```text
Preferences
  |
  v
Intent parser
  |
  v
TinyFish Search: discover official career boards and portals
  |
  v
Source planner: select independent URLs and source types
  |
  +--> Fetch workers for static listing pages
  |
  +--> Agent workers for dynamic filters, pagination, or portals
  |
  v
Normalizer: common listing schema + evidence
  |
  v
Deduplicator: canonical apply URL, source job ID, normalized identity
  |
  v
Matcher and ranker: role, location, keywords, seniority, visa, freshness
  |
  v
Streaming results, source errors, and final coverage summary
```

Recommended implementation details:

- Use SDK streaming events when building a web demo.
- Use `Promise.allSettled` so one broken portal does not hide other results.
- Emit source status and partial results while workers run.
- Validate every Agent result against a schema before ranking it.
- Keep `unknown` distinct from `false` for visa, salary, seniority, and
  work-mode fields.
- Keep matching reasons tied to observed listing fields.
- Never use a generic summary as evidence for a job listing.
- Never present a hardcoded source fallback as live availability.

## Submission Checklist

- [ ] Working demo link or runnable repository.
- [ ] User preference controls work.
- [ ] Multiple live sources are demonstrated.
- [ ] Structured listings include direct application links.
- [ ] Duplicate listings are removed.
- [ ] Match scores include reasons and gaps.
- [ ] Search, Fetch, and Agent usage is documented.
- [ ] Source evidence and retrieval timestamps are retained.
- [ ] No hardcoded job listings or fake source data.
- [ ] No passwords, API keys, or personal data included.
- [ ] Demo shared in the TinyFish Discord `#showcase` channel.
- [ ] LinkedIn post about the build tags TinyFish.

## Quality Bar

Prioritize:

- Fresh, actionable listings over large result counts.
- Direct application links over aggregator redirects.
- Clear match explanations over opaque scores.
- Evidence-backed fields over complete-looking records.
- Reliable behavior across different career-page architectures.
- A useful workflow someone would use this week.
