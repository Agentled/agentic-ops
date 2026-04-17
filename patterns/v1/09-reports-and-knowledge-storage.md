# 09 — Reports, sharing, and knowledge-graph storage: closing the loop

**Problem**: Agents produce analysis as free-form prose in AI step output, then either never display it or drop it into a raw JSON view. Results aren't shareable with stakeholders, aren't comparable across runs, and don't feed back into the Knowledge Graph for trend analysis or future scoring calibration.

**Why it fails silently**: Without a `renderer`, an AI step's structured output is unreadable in the workspace UI. Without a `share` step, the output has no URL that can be sent to a stakeholder. Without a `knowledgeSync` step, each run's data is discarded — the second run can't learn from the first, and KPIs have no history to trend against.

---

## The three-part closing loop

```
... upstream steps → AI report step (with Config renderer)
                   → [optional] share step (public URL)
                   → [optional] composed email step (delivery)
                   → knowledgeSync (persist to KG for trending)
                   → milestone
```

The three pieces are independent and optional, but they compound. A report with a renderer is readable. A report with a renderer + share step is forwardable. A report with all three + knowledgeSync is how KPI dashboards actually get built.

## 1. Report AI step with Config renderer

The Config renderer turns structured AI output into a KPI dashboard view. Key blocks: `kpiRow`, `markdown`, `table`, `signalList`.

```json
{
  "id": "generate-report",
  "type": "aiAction",
  "name": "Generate Report",
  "pipelineStepPrompt": {
    "template": "Analyze the data and produce a structured report.\n\nData: {{steps.evaluate.items}}",
    "responseStructure": {
      "summary": "string — executive summary",
      "kpis": "object { total: number, qualified: number, avgScore: number }",
      "items": "array of { name, score, decision }",
      "insights": "array of strings"
    }
  },
  "renderer": {
    "type": "Config",
    "config": {
      "layout": {
        "title": "Run Report",
        "blocks": [
          { "blockType": "kpiRow", "kpis": [
            { "label": "Total", "valuePath": "kpis.total", "icon": "Hash" },
            { "label": "Qualified", "valuePath": "kpis.qualified", "icon": "CheckCircle" },
            { "label": "Avg Score", "valuePath": "kpis.avgScore", "format": "score" }
          ]},
          { "blockType": "markdown", "contentPath": "summary" },
          { "blockType": "table", "arrayPath": "items", "searchable": true, "columns": [
            { "header": "Name", "field": "name" },
            { "header": "Score", "field": "score", "display": "score", "sortable": true },
            { "header": "Decision", "field": "decision", "display": "badge" }
          ]},
          { "blockType": "signalList", "title": "Insights", "arrayPath": "insights", "variant": "signal" }
        ]
      }
    }
  },
  "creditCost": 10,
  "next": { "stepId": "share-report" }
}
```

**Rule**: the `responseStructure` keys must match the renderer's `valuePath`/`arrayPath`/`contentPath` references. Drift between the two silently produces empty KPI tiles and blank tables.

## 2. Share step for a public URL

```json
{
  "id": "share-report",
  "type": "share",
  "name": "Create Public Report Link",
  "shareConfig": {
    "outputSteps": ["generate-report"],
    "expiresInDays": 30,
    "visibility": "public"
  },
  "next": { "stepId": "send-report-email" }
}
```

Outputs `{ shareId, shareUrl, expiresAt }`. Downstream steps use `{{steps.share-report.shareUrl}}` to reference the public URL (e.g., in an email body).

**When to add a share step**:

- Anyone outside the workspace needs to see the report.
- The report should remain accessible after the execution's detail page expires.
- The workflow delivers the report via email and the body should link to the full report.

**When to skip it**: internal-only dashboards where viewers already have workspace access.

## 3. Knowledge-graph storage for trending

Without this, each run's data vanishes. With it, you can:

- Compare "today's MRR" against last month's.
- Calibrate future AI scoring on past outcomes (see `kg.retrieve-scoring-memory`).
- Build a timeline view of any metric.

```json
{
  "id": "store-history",
  "type": "knowledgeSync",
  "name": "Store to KPI History",
  "knowledgeSync": {
    "source": { "stepId": "generate-report" },
    "listKey": "kpi_history",
    "fieldMapping": {
      "mrr": "mrr",
      "burn": "burn",
      "runway_months": "runway_months",
      "executedAt": "executedAt"
    }
  },
  "next": { "stepId": "done" }
}
```

If you're inside a loop (scoring many items per run), set `source.resultsPath: "items"` so each loop iteration produces one KG row.

## Anti-patterns

**AI step with no renderer:**

```yaml
type: aiAction
responseStructure:
  summary: string
  kpis: { total, avgScore }
# ❌ No renderer — the workspace UI shows raw JSON. Nobody reads it.
```

**Share step without `outputSteps`:**

```yaml
type: share
shareConfig:
  visibility: public
  # ❌ Missing outputSteps — share URL renders nothing
```

**`knowledgeSync` without a clear schema:**

If `listKey` doesn't already exist, the KG creates an implicit schema from the first write. Subsequent writes with different field shapes fail to index. For trending, pre-create the list with a typed schema.

**Reusing `fieldMapping` values with source-side renames:**

`fieldMapping` keys are source field names (from the prior step's output), values are target field names (in the KG). Flipping them silently stores the wrong data.

```yaml
# ✅ Correct: source → target
fieldMapping:
  mrr: monthly_revenue          # steps.extract.mrr → row.monthly_revenue
  burn: monthly_expenses
```

## Checklist for any "report" workflow

- [ ] `responseStructure` keys match every renderer `valuePath` / `arrayPath` / `contentPath`
- [ ] Renderer `blocks` cover at least one KPI + one tabular view
- [ ] Share step exists if report is forwardable to non-workspace users
- [ ] If delivered via email, the email template embeds `{{steps.share.shareUrl}}`
- [ ] `knowledgeSync` persists to a list with a typed schema (not implicit)
- [ ] `executedAt` (or a similar timestamp) is stored so rows can be trended
