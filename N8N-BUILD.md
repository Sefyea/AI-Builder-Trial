# Ghost Revenue — n8n build, node by node

Three files, two n8n workflows, one page.

| File | Where it lives | Job |
| --- | --- | --- |
| `index.html` | GitHub Pages | the UI — form, canvas, six card renderers |
| `data.json` | GitHub Pages, next to the page | the dataset with every derived column precomputed |
| this guide | your desk | wiring |

**Flow:** page → `POST /ask` webhook → AI Agent → calls `run_query` tool → gets real numbers → returns a JSON render spec → validator → page draws cards.

**The rule that makes it defensible:** the model never calculates. It chooses a query and chooses cards. Every number on screen comes out of a Code node.

---

## Where the dataset actually enters n8n

This is the part that isn't obvious from the diagram, so: **the dataset is not stored in n8n and it is not in the agent's prompt.** It arrives through one node — the `HTTP Request` node in the `run_query` sub-workflow (Workflow 1, Node 2), which GETs `data.json` off GitHub Pages on every tool call.

```
question
   ↓
AI Agent          knows the schema and the legal values, from the system message.
   │              has never seen a single row.
   │  calls run_query with a query spec
   ↓
run_query ─→ HTTP Request ─→ data.json on GitHub Pages   ← the dataset enters here
   │                              ↓
   │                         Code node computes the answer
   ↓  returns real numbers
AI Agent          wraps the numbers in cards. Adds no arithmetic of its own.
```

**Why not put the data in the prompt?** 283 rows across four tables is 134 KB. Pasting it into every request would be slow, would cost real money per question, and — worst — would let the model add up figures in its head. Then you cannot defend a single number on screen. Schema in the prompt, facts from the tool, always.

**If you'd rather not fetch over HTTP,** any of these swap in for Node 2 without touching anything else:

| Instead of HTTP Request | When it makes sense | Cost |
| --- | --- | --- |
| A `Code` node returning the JSON inline | fully self-contained, no external dependency | a 134 KB paste in the editor; awkward to update |
| `Google Sheets` node reading three tabs | the client wants to edit the data | needs OAuth; you must redo the derived columns in n8n |
| `Postgres` / Supabase | this becomes a real product | a schema to maintain, credentials to manage |

The HTTP fetch wins here because it is credential-free, version-controlled next to the page, and takes about 80 ms. It also demos well: the data visibly lives beside the UI.

**One caveat to state on camera:** every tool call re-fetches the file. Fine at this size, but for a production dataset you'd cache it — n8n has no built-in cache, so that would mean a static data node, a Redis node, or a real database.

---

## Step 0 — publish the page first

```bash
git init
git add index.html data.json
git commit -m "generative UI shell"
git branch -M main
git remote add origin https://github.com/<you>/<repo>.git
git push -u origin main
```

Repo → Settings → Pages → Source: `main`, folder `/root`. Wait for the green check, then open the URL.

The page works immediately — `WEBHOOK_URL` is empty so it answers from a built-in demo responder. Confirm `https://<you>.github.io/<repo>/data.json` loads in the browser before going further. n8n will fetch that exact URL.

Note the origin (e.g. `https://<you>.github.io`). You need it for CORS in step 2.

---

## Workflow 1 — `run_query`

Build this first; the agent can't reference a tool that doesn't exist. Four nodes.

### Node 1 · Execute Sub-workflow Trigger
- Add node → search **"When Executed by Another Workflow"**
- **Input data mode:** `Define using fields below`
- Add one field: name `spec`, type `String`

The agent will pass the query spec as a JSON string. Taking it as a string and parsing it yourself is more reliable than letting n8n coerce a nested object.

### Node 2 · HTTP Request — fetch the dataset
- **Method:** GET
- **URL:** `https://<you>.github.io/<repo>/data.json`
- **Response → Format:** `JSON`
- Options → **Response → Never Error:** off

`data.json` also carries a `schema` block listing every column and the legal values for each categorical field. You don't need a node for it — those values are already pasted into the system message. Keep the block anyway: when the agent writes a bad filter, you can diff the two and see instantly which one drifted.

*Optional:* if you'd rather the agent look values up than be told them, add a second Call n8n Sub-Workflow Tool called `get_schema` pointing at a two-node workflow (trigger → HTTP Request → return `data.schema`). Costs an extra round trip per question; buys self-correction. I'd skip it for a trial.

### Node 3 · Code — the executor
- **Mode:** `Run Once for All Items`
- **Language:** JavaScript

```js
const DATA = $input.first().json;

let spec;
try {
  const raw = $('When Executed by Another Workflow').first().json.spec;
  spec = typeof raw === 'string' ? JSON.parse(raw) : raw;
} catch (e) {
  return [{ json: { ok: false, error: 'spec was not valid JSON: ' + e.message } }];
}

function executeQuery(DATA, spec) {
  const TABLES = ['invoices','estimates','clients','bundles'];
  const AGGS   = ['sum','count','avg','median','min','max'];
  const OPS    = ['eq','ne','gt','gte','lt','lte','in','contains','is_true','is_false','not_null','is_null'];

  const err = m => ({ ok:false, error:m });
  if (!spec || typeof spec !== 'object') return err('spec must be an object');
  const table = String(spec.table || '');
  if (!TABLES.includes(table)) return err('unknown table: ' + table + '. allowed: ' + TABLES.join(', '));

  let rows = Array.isArray(DATA[table]) ? DATA[table].slice() : [];
  const cols = rows.length ? Object.keys(rows[0]) : [];

  for (const f of (Array.isArray(spec.filters) ? spec.filters : [])) {
    const c = String(f.col || '');
    if (!cols.includes(c)) return err('unknown column: ' + table + '.' + c);
    const op = String(f.op || 'eq');
    if (!OPS.includes(op)) return err('unknown op: ' + op);
    const v = f.value;
    rows = rows.filter(r => {
      const a = r[c];
      switch (op) {
        case 'eq':  return a === v;
        case 'ne':  return a !== v;
        case 'gt':  return Number(a) >  Number(v);
        case 'gte': return Number(a) >= Number(v);
        case 'lt':  return Number(a) <  Number(v);
        case 'lte': return Number(a) <= Number(v);
        case 'in':  return Array.isArray(v) && v.includes(a);
        case 'contains': return String(a ?? '').toLowerCase().includes(String(v ?? '').toLowerCase());
        case 'is_true':  return a === true;
        case 'is_false': return a === false;
        case 'not_null': return a !== null && a !== undefined;
        case 'is_null':  return a === null || a === undefined;
      }
      return true;
    });
  }

  const med = a => { if(!a.length) return null; const s=a.slice().sort((x,y)=>x-y), m=s.length>>1;
    return s.length%2 ? s[m] : (s[m-1]+s[m])/2; };
  const roll = (arr, agg, metric) => {
    const nums = arr.map(r => Number(r[metric])).filter(Number.isFinite);
    switch (agg) {
      case 'count':  return arr.length;
      case 'sum':    return nums.reduce((a,b)=>a+b,0);
      case 'avg':    return nums.length ? nums.reduce((a,b)=>a+b,0)/nums.length : null;
      case 'median': return med(nums);
      case 'min':    return nums.length ? Math.min(...nums) : null;
      case 'max':    return nums.length ? Math.max(...nums) : null;
    }
    return null;
  };

  const agg    = spec.agg ? String(spec.agg) : null;
  const metric = spec.metric ? String(spec.metric) : null;
  if (agg && !AGGS.includes(agg)) return err('unknown agg: ' + agg);
  if (agg && agg !== 'count') {
    if (!metric) return err('agg "' + agg + '" needs a metric column');
    if (!cols.includes(metric)) return err('unknown metric column: ' + table + '.' + metric);
  }
  const limit = Math.min(Math.max(parseInt(spec.limit ?? 25, 10) || 25, 1), 200);

  if (spec.group_by) {
    const gb = String(spec.group_by);
    if (!cols.includes(gb)) return err('unknown group_by column: ' + table + '.' + gb);
    const buckets = new Map();
    for (const r of rows) {
      const k = r[gb] === null || r[gb] === undefined ? '(blank)' : String(r[gb]);
      if (!buckets.has(k)) buckets.set(k, []);
      buckets.get(k).push(r);
    }
    let out = [...buckets.entries()].map(([k, arr]) => ({
      key: k, value: roll(arr, agg || 'count', metric), n: arr.length
    }));
    const dir = (spec.sort && spec.sort[0] && spec.sort[0][1] === 'asc') ? 1 : -1;
    const by  = (spec.sort && spec.sort[0] && spec.sort[0][0] === 'key') ? 'key' : 'value';
    out.sort((a,b) => by === 'key' ? dir * String(a.key).localeCompare(String(b.key))
                                   : dir * ((a.value ?? 0) - (b.value ?? 0)));
    return { ok:true, shape:'grouped', group_by:gb, agg:agg||'count', metric:metric||null,
             matched_rows:rows.length, rows: out.slice(0, limit) };
  }

  if (agg && !spec.select) {
    return { ok:true, shape:'scalar', agg, metric:metric||null,
             matched_rows:rows.length, value: roll(rows, agg, metric) };
  }

  let sel = Array.isArray(spec.select) && spec.select.length ? spec.select : cols;
  const bad = sel.find(c => !cols.includes(c));
  if (bad) return err('unknown select column: ' + table + '.' + bad);
  if (Array.isArray(spec.sort) && spec.sort.length) {
    const [sc, sd] = spec.sort[0];
    if (!cols.includes(sc)) return err('unknown sort column: ' + table + '.' + sc);
    const d = sd === 'asc' ? 1 : -1;
    rows.sort((a,b) => {
      const x=a[sc], y=b[sc];
      if (typeof x === 'number' && typeof y === 'number') return d*(x-y);
      return d*String(x??'').localeCompare(String(y??''));
    });
  }
  return { ok:true, shape:'rows', columns:sel, matched_rows:rows.length,
           rows: rows.slice(0, limit).map(r => sel.map(c => r[c])) };
}

return [{ json: executeQuery(DATA, spec) }];
```

### Node 4 · Code — shrink the reply
- **Mode:** `Run Once for All Items`

```js
// Keep the tool response small. Big payloads burn tokens and slow the agent.
const r = $input.first().json;
return [{ json: { result: JSON.stringify(r).slice(0, 6000) } }];
```

Save as **`run_query`**. Test it: Execute Workflow, and in the trigger's pinned input put

```json
{ "spec": "{\"table\":\"invoices\",\"filters\":[{\"col\":\"true_state\",\"op\":\"eq\",\"value\":\"never_sent\"}],\"agg\":\"sum\",\"metric\":\"amount\"}" }
```

Expected: `value: 138142.23`, `matched_rows: 14`. If you get that, the hard part is done.

Two more to verify by hand — they're the numbers you'll quote on camera:

| spec | expected |
| --- | --- |
| `true_state = past_due_mislabelled`, sum amount | `172042.43`, 23 rows |
| `estimates` where `won_not_invoiced is_true`, sum amount | `84252.54`, 2 rows |
| `invoices` where `line_items contains "Server maintenance"`, count | `18` |
| `clients` where `industry_mismatch is_true`, count | `26` |

Then deliberately break one, so you know what failure looks like: change `never_sent` to `Never_Sent`. You should get `matched_rows: 0` with `ok: true` — a silent empty result, not an error. That is exactly why the legal values are pinned in the system message, and it's worth a sentence in the Loom.

---

## Workflow 2 — `ask`

Five nodes plus two attachments.

### Node 1 · Webhook
- **HTTP Method:** `POST`
- **Path:** `ask`
- **Respond:** `Using 'Respond to Webhook' Node`
- Options → add **Allowed Origins (CORS)** → `https://<you>.github.io`

That CORS field is the single most common failure point. If it isn't there in your n8n version, see the CORS section below.

### Node 2 · AI Agent
- Add node → **AI Agent**
- **Agent:** `Tools Agent`
- **Source for Prompt:** `Define below`
- **Prompt:**
  ```
  {{ $json.body.question }}
  ```
- Options → **System Message:** paste the full prompt from the next section
- Options → **Max Iterations:** `5`
- Settings → **Always Output Data:** on
- Settings → **On Error:** `Continue (using error output)`

Attach to the agent:

- **Chat Model** — OpenAI Chat Model or Anthropic Chat Model. Pick a mid-tier reasoning model; set **Temperature** to `0.2`.
- **Memory** — Simple Memory. **Session ID:** `Define below` → `{{ $json.body.session_id }}`, **Context Window Length:** `6`.
- **Tool** — **Call n8n Sub-Workflow Tool**
  - **Name:** `run_query`
  - **Description:**
    ```
    Runs a read-only query against the dataset and returns real numbers. Input is a JSON string
    matching the query spec. Tables: invoices, estimates, clients, bundles. Never do arithmetic
    yourself — always call this. If it returns ok:false, read the error, fix the spec, and retry once.
    ```
  - **Workflow:** select `run_query`
  - **Workflow Inputs:** map `spec` ← `Defined automatically by the model`
- **Output Parser** — **Structured Output Parser**, with the schema from below. Also switch on the agent's **Require Specific Output Format**.

### Node 3 · Code — the validator
- **Mode:** `Run Once for All Items`

```js
const ALLOWED = ['kpi','bar','table','strip','refusal','note'];
const src = $input.first().json;
const s = src.output ?? src.text ?? src ?? {};

let cards = Array.isArray(s.cards) ? s.cards : [];
cards = cards
  .filter(c => c && typeof c === 'object')
  .map(c => ALLOWED.includes(c.type) ? c : ({
    type: 'note',
    title: 'Unsupported card type: ' + String(c.type).slice(0, 40),
    body: String(c.title || 'The agent asked for a card the page cannot draw.').slice(0, 400)
  }))
  .slice(0, 6);

if (!cards.length) {
  cards = [{ type: 'note', title: 'I could not build that view',
    body: String(s.narration || 'No renderable result.').slice(0, 400) }];
}

return [{ json: {
  v: 1,
  intent: String(s.intent || 'unknown').slice(0, 60),
  confidence: typeof s.confidence === 'number' ? Math.max(0, Math.min(1, s.confidence)) : null,
  narration: String(s.narration || '').slice(0, 400),
  cards
} }];
```

### Node 4 · Respond to Webhook
- **Respond With:** `JSON`
- **Response Body:** `{{ JSON.stringify($json) }}`
- Options → **Response Headers** → add:
  - `Access-Control-Allow-Origin` : `https://<you>.github.io`
  - `Content-Type` : `application/json`

### Node 5 · Code — error branch (optional but do it)

Wire the agent's **error output** to a second Code node, then into the same Respond to Webhook:

```js
return [{ json: { v:1, intent:'agent_error', confidence:0, narration:'',
  cards:[{ type:'note', title:'The agent failed on that one',
    body:'Try rephrasing. Detail: ' + String($json.error?.message || 'unknown').slice(0,200) }] } }];
```

Now the page always gets a valid spec, even when the model falls over mid-demo.

---

## The system message

```
You are the query and presentation layer for a finance data explorer. You never calculate.
You choose what to query and how to draw it.

DATA
Four tables, reachable only through the run_query tool.
  invoices   187 rows. invoice_id, client_id, client_name, issue_date, due_date, amount, status,
             line_items, notes, net_terms, days_past_due, age_days, days_in_draft, weekend_issued,
             is_past_due, service_type, price_index, true_state, note_intent, note_conflict
  estimates   66 rows. estimate_id, client_id, client_name, issue_date, amount, status, notes,
             days_pending, first_inv_after, first_inv_amount, est_to_inv_ratio, won_not_invoiced
  clients     30 rows. client_id, name, industry, account_age_months, primary_contact, health,
             health_norm, industry_derived, industry_mismatch, is_nonprofit, first_activity,
             months_of_activity, tenure_gap, tenure_impossible, n_invoices, invoiced_total,
             past_due_total, past_due_pct, n_estimates
  bundles     10 rows. line_items, n, min, p25, median, p75, max, spread_x, service_type
Today is 2026-08-31. Every figure is already as-of that date.

QUERY SPEC — pass as a JSON string to run_query
{ "table": "invoices|estimates|clients|bundles",
  "filters": [{"col":"status","op":"eq","value":"Draft"}],
  "group_by": "client_name",
  "agg": "sum|count|avg|median|min|max",
  "metric": "amount",
  "select": ["invoice_id","amount"],
  "sort": [["value","desc"]],
  "limit": 10 }
ops: eq ne gt gte lt lte in contains is_true is_false not_null is_null
Use group_by for rankings, agg alone for a single number, select for row listings.

LEGAL VALUES — filters are case-sensitive and exact. Never guess a value that is not on this list.
  invoices.status         Draft | Sent | Paid | Overdue
  invoices.true_state     collected | overdue_flagged | past_due_mislabelled | never_sent
  invoices.service_type   project | recurring | emergency
  invoices.note_intent    payment_risk | dispute | process_failure | blocked_admin | deferral |
                          bundling | concession | admin | none
  invoices.net_terms      15 | 30 | 45 | 60   (number, not string)
  invoices.weekend_issued, is_past_due, note_conflict   booleans — use is_true / is_false
  estimates.status        Won | Lost | Pending
  estimates.won_not_invoiced   boolean
  clients.health          At Risk | Good | WATCH | at risk | good | watch | null   (7 variants — do
                          not filter on this; filter on health_norm)
  clients.health_norm     at risk | good | watch | unrated
  clients.industry and clients.industry_derived
                          Construction | Education | Healthcare | Hospitality | Legal |
                          Manufacturing | Nonprofit | Professional Services | Real Estate | Retail
  clients.is_nonprofit, industry_mismatch, tenure_impossible   booleans
  bundles.line_items      10 exact strings. Do not retype them — filter with
                          {"col":"line_items","op":"contains","value":"Server maintenance"}
The same 10 strings appear in invoices.line_items. Always use "contains", never "eq", on that column.

FIELDS YOU MUST NOT TREAT AS TRUE
  clients.industry            wrong on 26 of 30 rows. Never group by it. Refuse and offer
                              industry_derived instead, labelled inferred.
  clients.account_age_months  impossible for 9 of 30 clients. History spans only 20 months, so no
                              tenure above 20 is verifiable. Answer if asked, then undermine it.
  clients.health              7 casing variants, 7 unrated, and unrated clients pay best. Not a
                              risk signal. Use health_norm and say so.
  invoices.status             all 23 rows marked Sent are already past due. Use true_state.
  invoices.amount             randomly generated in the source file. Safe to sum. Never correlate
                              it with anything and never claim a driver or a trend.

EVERY REPLY IS EXACTLY ONE OF THREE
  answer   — call run_query, then draw cards from what it returned.
  refuse   — the data cannot support the question. Use a refusal card, name the column, give
             evidence rows, and say what you can do instead.
  clarify  — genuinely ambiguous. One note card asking one question.

CARDS — the only six types that exist
  kpi      headline figures.       items: [{v, l, tone}]              tone: ok|warn|bad
  bar      rankings.               rows: [[label, number]], unit "$" or "%", tone
  table    row listings.           columns: [], rows: [[]], money_columns: [indexes]
  strip    one bundle's spread.    values: [], p25, med, p75, tone
  refusal  why you won't answer.   title, body, evidence: [[label, detail]]
  note     caveats and context.    title, body

RULES
  Never invent a card type. If you cannot build a valid card, return a note explaining why.
  Never state a number that did not come back from run_query in this turn.
  Two to three cards is the right answer for most questions: the figures, then the caveat.
  Set confidence below 0.5 whenever the answer leans on a field in the untrusted list.
  If run_query returns ok:false, read the error, correct the spec, retry once, then refuse.
  Keep narration to one or two sentences. The cards carry the content.
  If previous_spec is present in the request, treat the question as a follow-up and modify it.
```

---

## The Structured Output Parser schema

Set the parser to **Define using JSON Schema** and paste:

```json
{
  "type": "object",
  "required": ["intent", "narration", "cards"],
  "additionalProperties": false,
  "properties": {
    "intent": { "type": "string", "maxLength": 60 },
    "confidence": { "type": "number", "minimum": 0, "maximum": 1 },
    "narration": { "type": "string", "maxLength": 400 },
    "cards": {
      "type": "array", "minItems": 1, "maxItems": 6,
      "items": {
        "type": "object",
        "required": ["type", "title"],
        "properties": {
          "type": { "type": "string", "enum": ["kpi","bar","table","strip","refusal","note"] },
          "title": { "type": "string", "maxLength": 120 },
          "body": { "type": "string", "maxLength": 700 },
          "unit": { "type": "string", "enum": ["$","%",""] },
          "tone": { "type": "string", "enum": ["ok","warn","bad","accent"] },
          "items": { "type": "array", "maxItems": 8, "items": {
            "type": "object", "required": ["v","l"], "properties": {
              "v": { "type": "string" }, "l": { "type": "string" },
              "tone": { "type": "string", "enum": ["ok","warn","bad","accent"] } } } },
          "rows": { "type": "array", "maxItems": 60, "items": { "type": "array" } },
          "columns": { "type": "array", "items": { "type": "string" } },
          "money_columns": { "type": "array", "items": { "type": "integer" } },
          "values": { "type": "array", "maxItems": 60, "items": { "type": "number" } },
          "p25": { "type": "number" }, "med": { "type": "number" }, "p75": { "type": "number" },
          "evidence": { "type": "array", "maxItems": 12, "items": { "type": "array" } }
        }
      }
    }
  }
}
```

`additionalProperties: false` on the root and the `enum` on `type` are what stop the model inventing a `pieChart`.

---

## CORS

The browser blocks the fetch before your workflow ever runs, so this looks like "nothing happens".

1. Set **Allowed Origins (CORS)** on the Webhook node to your Pages origin.
2. Set the `Access-Control-Allow-Origin` response header on **Respond to Webhook**.
3. If preflight still fails, add a second Webhook node on the same path `ask` with method `OPTIONS`, wired straight to a Respond to Webhook returning status `204` with headers `Access-Control-Allow-Origin`, `Access-Control-Allow-Methods: POST, OPTIONS`, `Access-Control-Allow-Headers: content-type`.
4. Test from the published Pages URL, not from a local file. `file://` origins behave differently and will mislead you.

Use `*` for the origin only while debugging, and put the real origin back before you record.

---

## Test sequence

Do it in this order. Each step isolates one failure.

1. `data.json` loads in a browser tab.
2. `run_query` returns `138142.23` for the pinned draft spec.
3. Workflow `ask` — use the Webhook's **Test URL** and n8n's built-in test, question `"how much money is stuck?"`. Check the agent actually called the tool (click the node, look at the tool call log).
4. Activate the workflow. Copy the **Production URL**.
5. Paste it into `WEBHOOK_URL` in `index.html`, push, reload the page. The pill should read `live · n8n`.
6. Ask `"show revenue by industry"`. You want a refusal card. This is the demo moment.
7. Ask something absurd — `"what's our churn rate by moon phase"`. You want a graceful note, not a broken page.

---

## Budget

One question is 1–3 LLM calls and one or two tool calls: roughly two to five seconds and a fraction of a cent. The whole thing runs on n8n's free tier and a few dollars of model credit for the trial.

---

## What to show in the Loom

1. Ask a normal question. Cards appear.
2. Hit **show the spec** — this is the moment a technical reviewer leans in. The agent returned typed JSON, not HTML, and the page owns the pixels.
3. Ask for revenue by industry. It refuses, with evidence.
4. Cut to the n8n canvas and click into the `run_query` tool call so they can see the model chose a query and the Code node produced the number.
5. Say the line: the model never calculates.
