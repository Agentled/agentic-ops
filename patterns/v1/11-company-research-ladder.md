# 11 — Company research: match the source to the question

**Problem**: Developers default to one source (usually LinkedIn company scraper) for all company research, regardless of what they actually need. The result: team-centric data when the question was about positioning, positioning-centric data when the question was about revenue, and credits spent on a scrape that returns fields irrelevant to the decision downstream.

**Why it fails silently**: Each company-data source is biased toward a different dimension. LinkedIn is **people-centric** (team, headcount, hiring). Company websites are **positioning-centric** (products, messaging, target customer). Structured directories (Crunchbase, Specter, SEC) are **financial and structural** (funding, revenue, legal entity). Using the wrong source returns plausible-looking data that's wrong for the downstream use. There's no error — just a bad decision built on the wrong signal.

---

## The source determines the answer

Before writing the workflow, ask: **what is the report / decision / email downstream going to use this data for?**

| Downstream need | Right primary source | What it returns |
|---|---|---|
| "Is this company worth selling to?" | Website scrape + directory | product, target customer, funding stage |
| "Who should we target inside?" | LinkedIn company + people search | team size, titles, hiring signal |
| "Is this a real company?" | Directory (Crunchbase / registry) | legal entity, founded date, funding |
| "What do they actually do?" | Website homepage + /about + /pricing | product description, pricing, customer base |
| "Are they growing?" | LinkedIn headcount trend + news search | hiring velocity, press mentions |
| "Who uses this product?" | Website testimonials + case studies | logos, customer quotes |

Pick the source for the question. Pulling from LinkedIn when the question is "what do they do" gives you industry codes and team size — not an answer.

---

## The input signal determines the first lookup

| You have | Best first lookup | Fallback |
|---|---|---|
| LinkedIn company URL | LinkedIn company scraper | Resolve to website → website scrape |
| Company website domain | Homepage + `/about` scrape | Resolve domain → LinkedIn URL |
| Company name only | Web search → resolve domain + LinkedIn URL | Directory search (Crunchbase) |
| Stock ticker / legal name | Public-data API (Crunchbase / SEC) | Web search |
| Email domain | Reverse-resolve to company | Web search |

Same principle as person research: route by input signal, don't pick by preference.

---

## Anti-pattern 1 — LinkedIn-scrape for everything

```yaml
# Wrong: LinkedIn scrape to answer "what does this company sell?"
- id: research-company
  action: linkedin.get-company-from-url
  input:
    profileUrl: "{{input.linkedinUrl}}"
```

LinkedIn company pages describe the company in the company's own recruiting voice — "we're transforming the future of X" — optimized for hiring, not for understanding what they sell. If the question is "what do they sell, to whom, at what price?", you'll get a generic industry label and need a website scrape anyway. Skip the LinkedIn hop.

---

## Anti-pattern 2 — Website scrape when you need people data

```yaml
# Wrong: website scrape to answer "who's the head of growth?"
- id: find-head-of-growth
  action: web-scraping.scrape
  input:
    url: "https://{{input.domain}}/about"
```

Most company `/about` pages don't list the full team. The ones that do are out of date. LinkedIn's people search (filtered by company + title) is the right primary source for people-at-company questions.

---

## Anti-pattern 3 — One lookup, then stop

```yaml
# Wrong: single-source research
- id: research
  action: linkedin.get-company-from-url
- id: generate-report
  # runs with whatever LinkedIn returned, even if critical fields are empty
```

Even with the right primary source, one lookup rarely gives you enough. The strong pattern is **primary source + one enrichment** from an orthogonal source — e.g. LinkedIn (team) + website scrape (product), merged before the report step.

---

## Correct pattern — Scope the research, then layer sources

```yaml
steps:
  # 1. Determine what the downstream step actually needs
  - id: scope-research
    type: ai-action
    prompt: |
      Given the research goal "{{input.goal}}", list which dimensions we need:
      - team_and_headcount (people-centric)
      - product_and_positioning (website-centric)
      - funding_and_structure (directory-centric)
      - growth_signals (news + LinkedIn trend)
    responseStructure:
      dimensions: "array of strings"

  # 2a. Team dimension → LinkedIn
  - id: linkedin-fetch
    entryConditions:
      criteria:
        - variable: "{{steps.scope-research.dimensions}}"
          operator: "contains"
          value: "team_and_headcount"
    action: linkedin.get-company-from-url
    input:
      profileUrl: "{{input.linkedinUrl}}"

  # 2b. Product dimension → website
  - id: website-scrape
    entryConditions:
      criteria:
        - variable: "{{steps.scope-research.dimensions}}"
          operator: "contains"
          value: "product_and_positioning"
    action: web-scraping.scrape
    input:
      url: "https://{{input.domain}}"

  # 2c. Funding dimension → directory
  - id: directory-lookup
    entryConditions:
      criteria:
        - variable: "{{steps.scope-research.dimensions}}"
          operator: "contains"
          value: "funding_and_structure"
    action: crunchbase.get-company
    input:
      name: "{{input.companyName}}"

  # 3. Merge into a single context for the report
  # Branches are declared sequentially, so each either runs or self-skips
  # before this step. No explicit sync gate is needed. For platforms that
  # run branches in parallel, add a group_completion entry condition or
  # use the platform's equivalent join step.
  - id: merge-context
    type: code
    code: |
      return {
        team: steps["linkedin-fetch"]?.output ?? null,
        product: steps["website-scrape"]?.output ?? null,
        funding: steps["directory-lookup"]?.output ?? null,
      }
```

Note the null-safe reads: each branch returns `null` when its entry condition fails, and the report step downstream must handle missing dimensions (`team_and_headcount` skipped → no team data) rather than treating null as an error.

---

## When no native action exists — use computer use or scraping

Every company has at least a website, and most have a LinkedIn page. But for structured data (legal entity, parent company, funding history, board members), you'll frequently hit sources with no connector: regional business registries, industry associations, niche databases. The default reaction — "no action for this, skip it" — drops information you could have fetched.

| Source type | Tool | When to use |
|---|---|---|
| Company website (homepage, `/about`, `/team`, `/pricing`) | Web scraper (`web-scraping.scrape`) | Positioning, products, visible team |
| Public registry or directory (static HTML) | Web scraper | Legal entity, registration data |
| Dynamic site (JS-only, paywall, login wall) | Computer use (`browser-use.run-task`, `anthropic-computer-use.run-task`) | LinkedIn if no scraper connector, Crunchbase/Pitchbook pages behind auth, regional regulator portals |
| Multi-step research task ("find the latest funding round and CEO") | Computer use with an extraction goal | Requires navigation + synthesis across pages |

```yaml
# Scrape a company /about page as a cheap positioning source
- id: about-scrape
  action: web-scraping.scrape
  input:
    url: "https://{{input.domain}}/about"

# Computer use for a dynamic registry with no API
- id: registry-lookup
  action: browser-use.extract-data
  input:
    url: "https://some-business-registry.example.com/search?q={{input.companyName}}"
    extractionGoal: "Return the legal entity name, incorporation date, and registered address"
```

Rules of thumb:
- **Scrape websites directly** rather than rely on LinkedIn's second-hand description. Scraping the homepage is nearly always cheaper and richer for product/positioning than a LinkedIn company scrape.
- Use computer use when scraping is blocked or the source is JS-heavy. Don't retry scraping against a Cloudflare wall.
- Pipe extracted HTML / markdown into an AI step for structured extraction — don't pattern-match by hand.
- Set a credit ceiling on computer-use steps (`maxSteps`, timeout). Runaway browser sessions are the most expensive failure mode in this pattern.

---

## Fallback ladder

When the primary source is empty or blocked, fall to a different source type:

```
1. Primary connector for the dimension   (LinkedIn / Crunchbase / directory API)
         ↓ null or blocked
2. Secondary source for the dimension    (website /team for people; /about for product; press for funding)
         ↓ null
3. Web search + LLM extraction           (open web)
         ↓ null
4. Web scraping                          (web-scraping.scrape on static pages)
         ↓ null or blocked
5. Computer use / browser automation     (browser-use, anthropic-computer-use for JS/auth pages)
         ↓ null
6. Mark field as unknown — don't hallucinate
```

Same rule as person research: fall down the ladder to a **different source type**, not across to another provider in the same tier. Tiers 4 and 5 are the "no native connector" tiers — reach for them before giving up on a dimension.

---

## One-line rule

> Match the data source to the dimension of the question — LinkedIn for people, website for positioning, directories for financials — and layer sources rather than trusting a single one.
