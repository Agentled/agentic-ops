# 10 — Person research: pick the lookup by the signal you have

**Problem**: Developers reach for a single favourite enrichment tool (LinkedIn scraper, Hunter, Clearbit) regardless of what input they have. The result: wasted credits on low-signal inputs, missing data when the "go-to" provider doesn't have the record, and no fallback when the first call returns null.

**Why it fails silently**: Enrichment APIs return `null` or an empty object for "not found" — not an error. An agent sees `email: null`, treats it as a terminal failure, and moves on. The real problem is that the wrong lookup was picked for the input signal, and a better fallback exists one tier down.

---

## The input signal determines the right lookup

Every person-research task starts with some subset of: name, company domain, company name, LinkedIn URL, email, or job title. The strongest signal you have determines which lookup has the best hit rate. Picking by preference instead of by signal is how you burn credits.

| You have | Best first lookup | What it returns | Typical hit rate |
|---|---|---|---|
| LinkedIn profile URL | LinkedIn profile scraper | full profile: name, headline, company, experience, education | 90%+ |
| Name + company **domain** | Email-finder API (name + domain) | verified email, score | 60–80% |
| Name + company **name** (no domain) | Web search to resolve domain → email-finder | domain first, then email | 40–60% |
| Company domain only | Domain-wide email search | list of public emails with name + role | varies (5–50 rows) |
| Email only | Email verification + reverse lookup | name, company, social profile | 30–50% |
| Name only | Search + disambiguation (LLM + web_search) | candidate list — requires user confirmation | low — ambiguous |
| Job title + company | LinkedIn people search by company + title | candidate profiles | 40–70% |

**Rule**: Route the workflow by input signal at the top. Don't pick the lookup based on which API you like best.

---

## Anti-pattern 1 — LinkedIn-first for everything

Using a LinkedIn profile scraper as the first step regardless of input:

```yaml
# Wrong: LinkedIn scrape when all you have is name + domain
steps:
  - id: find-person
    action: linkedin.get-profile-from-search
    input:
      query: "{{input.name}} {{input.company}}"
```

Problems:
- LinkedIn search by name is ambiguous — returns the wrong person when the name is common
- LinkedIn scrapers are the most expensive tier (rate-limited, sometimes blocked)
- If all you needed was an email, a direct name+domain email-finder would have been 10× cheaper and higher hit rate

LinkedIn scraping is the right tool when you **already have** the profile URL, or when you need deep profile detail (experience, bio, connections). Not for email-finding.

---

## Anti-pattern 2 — Email-finder with name only

```yaml
# Wrong: email-finder without a domain
- id: find-email
  action: hunter.find-email
  input:
    firstName: "{{input.firstName}}"
    lastName: "{{input.lastName}}"
    # no domain — this is guessing
```

Email-finders need `firstName + lastName + domain`. Without a domain they either return nothing or guess at public domains (`gmail.com`, `yahoo.com`) with near-zero accuracy. Always resolve the domain first.

---

## Anti-pattern 3 — LLM-as-lookup

```yaml
# Wrong: asking the LLM to return an email
- id: find-email
  type: ai-action
  prompt: "What is the email address of {{input.name}} at {{input.company}}?"
```

The model will hallucinate a plausible email (`firstname.lastname@company.com`) with no verification. LLMs are good at **routing** the lookup and **disambiguating** candidates — not at returning verified contact data. Use them as the orchestrator, not the database.

---

## Correct pattern — Signal-based routing with a fallback ladder

```yaml
steps:
  # 1. Triage input: what do we actually have?
  - id: classify-input
    type: ai-action
    prompt: |
      Given input {{input}}, determine the strongest identity signal.
      Return exactly one of:
        linkedin_url
        name_and_domain
        name_and_company
        company_domain_only
        email_only
        job_title_and_company
        name_only
    responseStructure:
      signal: "string"
      extracted: "object — fields you pulled out"

  # 2a. LinkedIn URL → direct profile scrape
  - id: scrape-linkedin
    entryConditions:
      criteria:
        - variable: "{{steps.classify-input.signal}}"
          operator: "=="
          value: "linkedin_url"
    action: linkedin.get-profile-from-url
    input:
      profileUrl: "{{steps.classify-input.extracted.linkedinUrl}}"

  # 2b. Name + domain → email-finder directly
  - id: find-email-direct
    entryConditions:
      criteria:
        - variable: "{{steps.classify-input.signal}}"
          operator: "=="
          value: "name_and_domain"
    action: email-finder.find
    input:
      firstName: "{{steps.classify-input.extracted.firstName}}"
      lastName: "{{steps.classify-input.extracted.lastName}}"
      domain: "{{steps.classify-input.extracted.domain}}"

  # 2c. Name + company → resolve domain first, then email-finder
  - id: resolve-domain
    entryConditions:
      criteria:
        - variable: "{{steps.classify-input.signal}}"
          operator: "=="
          value: "name_and_company"
    type: ai-action
    tools: [web_search]
    prompt: "Find the official website domain for company {{steps.classify-input.extracted.company}}"
    responseStructure:
      domain: "string"

  - id: find-email-resolved
    entryConditions:
      criteria:
        - variable: "{{steps.resolve-domain.domain}}"
          operator: "isNotNull"
    action: email-finder.find
    input:
      firstName: "{{steps.classify-input.extracted.firstName}}"
      lastName: "{{steps.classify-input.extracted.lastName}}"
      domain: "{{steps.resolve-domain.domain}}"

  # 2d. Company domain only → domain-wide email search, then filter
  - id: domain-wide-emails
    entryConditions:
      criteria:
        - variable: "{{steps.classify-input.signal}}"
          operator: "=="
          value: "company_domain_only"
    action: email-finder.find-by-domain
    input:
      domain: "{{steps.classify-input.extracted.domain}}"

  # 2e. Email only → verify + reverse lookup
  - id: reverse-lookup
    entryConditions:
      criteria:
        - variable: "{{steps.classify-input.signal}}"
          operator: "=="
          value: "email_only"
    action: email-finder.verify-and-enrich
    input:
      email: "{{steps.classify-input.extracted.email}}"

  # 2f. Job title + company → LinkedIn people search
  - id: people-search
    entryConditions:
      criteria:
        - variable: "{{steps.classify-input.signal}}"
          operator: "=="
          value: "job_title_and_company"
    action: linkedin.search-people
    input:
      company: "{{steps.classify-input.extracted.company}}"
      title: "{{steps.classify-input.extracted.title}}"

  # 2g. Name only → disambiguation step; don't guess — ask or stop
  - id: disambiguate
    entryConditions:
      criteria:
        - variable: "{{steps.classify-input.signal}}"
          operator: "=="
          value: "name_only"
    type: ai-action
    tools: [web_search]
    prompt: |
      "{{steps.classify-input.extracted.name}}" is ambiguous.
      Search and return up to 5 candidate profiles. Stop here —
      the workflow must not enrich a single candidate without
      user confirmation.
    responseStructure:
      candidates: "array of { name, company, url }"
      needsConfirmation: "boolean"
```

Every declared signal has a branch. Missing a branch means the workflow silently produces no enrichment for that signal — the one exact failure this pattern exists to prevent.

---

## When no native action exists — use computer use or scraping

The ladder above assumes a native action exists for each step (email-finder API, LinkedIn scraper connector, directory API). In practice you'll hit data sources with no connector: an industry association directory, a regional regulatory database, a conference attendee page, a niche talent platform. **Do not skip the lookup** just because there's no matching action.

Two fallback tools cover this:

| Source type | Tool | When to use |
|---|---|---|
| Static HTML page (team page, about page, directory listing) | Web scraper (`web-scraping.scrape`) | Content is in the initial HTML, no auth/JS required |
| Dynamic / authenticated page (LinkedIn without a scraper connector, logged-in dashboard) | Computer use / browser automation (`browser-use.run-task`, `anthropic-computer-use.run-task`) | Content requires clicks, scrolling, login, JS execution |

```yaml
# Example: no native connector for this directory — use scraping
- id: association-directory
  action: web-scraping.scrape
  input:
    url: "https://some-industry-association.example.com/members/{{currentItem.slug}}"

# Example: no LinkedIn scraper connector in this workspace — use computer use
- id: linkedin-via-browser
  action: browser-use.extract-data
  input:
    url: "{{currentItem.linkedinUrl}}"
    extractionGoal: "Return the person's current title, company, and email if visible"
```

Rules of thumb:
- **Scrape first**, computer use second. Scraping is ~10× cheaper and faster when it works.
- Feed the scraped HTML / markdown into an AI extraction step — don't try to regex it.
- Computer use is the last tier before giving up. Budget for it explicitly — each `run-task` costs real credits and takes seconds-to-minutes.
- If a source blocks scraping (Cloudflare, JS-only, login wall), jump straight to computer use — retrying the scraper won't help.

---

## Fallback ladder — fall *down*, not *across*

When a lookup returns null, the mistake is to retry the **same tier** with the same input: re-querying Hunter with slight input variations, calling a second email-finder with the same name+domain. That's falling across. It rarely helps — the providers share similar data sources.

Fall **down** the ladder to a different source type:

```
1. Direct API lookup           (email-finder, LinkedIn-by-URL, connector action)
         ↓ null
2. Structured directory        (Crunchbase, Apollo, Specter, company team page API)
         ↓ null
3. Web search + extraction     (LLM + web_search over the open web)
         ↓ null
4. Web scraping                (web-scraping.scrape on a static team/about page)
         ↓ null or blocked
5. Computer use / browser auto (browser-use, anthropic-computer-use on dynamic / auth pages)
         ↓ null
6. Stop — ask for more input signal; do not hallucinate
```

Each tier below is heavier (slower, more credits, more failure modes) but accesses a different data source. Never retry the same tier more than once. Tiers 4 and 5 specifically exist for sources with no native connector — use them rather than declaring the research impossible.

---

## One-line rule

> Pick the lookup by the strongest signal in the input; when a lookup fails, fall down the ladder to a different source — never retry the same tier hoping for a different answer.
