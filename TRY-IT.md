# Try it yourself

Live page: `https://<you>.github.io/<repo>/`

Type anything you like. The seven suggested chips are there for speed, not because the box only accepts those. Below are six scenarios worth trying in order, because each one tests a different thing — and three of them are designed to make the tool fail in a specific way.

Every figure you see was computed in a code node from the dataset in this repo. The model chose the query and chose the cards. It never did arithmetic. Hit **show the spec the agent returned** under any answer to see the exact JSON it produced.

---

### 1. A question with a clean answer

> **Which drafts were never sent?**

You should get a 14-row table totalling $138,142, sorted by age. The oldest has been sitting 466 days.

*Testing:* the basic path — intent → query → real numbers → the right card type for the shape of the answer.

---

### 2. A question where the honest answer is "none"

> **Which invoices are over $50,000?**

This is a perfectly valid query that matches zero of 187 rows. You should be told the result set was empty, plus the five largest invoices for context — the biggest is $24,809.96.

*Testing:* whether an empty result gets reported as empty. The common failure here is a tool that quietly shows you the next-closest thing and lets you assume it answered the question.

---

### 3. A question the data cannot support

> **Show revenue by industry**

You should get a refusal, not a chart. The `industry` column disagrees with the company name on 26 of 30 clients — Apex Dental Group is filed as Legal, Northstar Legal LLP as Manufacturing, and the only client filed as Nonprofit is a resort. The refusal names the column, shows evidence rows, and offers a derived alternative.

*Testing:* whether it will decline to produce a plausible-looking chart from a field it knows is wrong. This is the behaviour I most wanted in the build.

---

### 4. A question with an answer that needs a caveat

> **Who's our oldest client?**

You get an answer — Northstar Legal LLP at 95 months — immediately followed by why that number is soft. Transaction history spans only 20 months, so no tenure above 20 is verifiable, and Northstar shows 10.4 months of actual activity against a claimed 95. Note the confidence value drops to 0.4.

*Testing:* the middle ground between answering and refusing. Most tools only have the two extremes.

---

### 5. Something absurd

> **What's our churn rate by moon phase?**

You should get a graceful explanation that no column could answer it — there is no cancellation date, no contract end date, no subscription record — followed by what it *can* answer. No crash, no empty chart, no invented number.

*Testing:* graceful degradation. Try your own nonsense too; `asdfgh qwerty` also lands somewhere sensible.

---

### 6. Break it on purpose

Ask the same question twice. The second answer returns instantly and is badged **from cache · no agent call**.

Then ask several different questions quickly. After a few seconds' gap you'll hit a cooldown note, and there's a 25-call daily cap per browser.

*Testing:* that a public page with a paid model behind it cannot be made expensive by refreshing or spamming it. Reloading the page costs nothing at all — the opening view is precomputed and badged as such.

---

## What's in the dataset

`data.json` in this repo is the three source tabs normalised into one file, with derived columns added. 187 invoices, 66 estimates, 30 clients, 10 service bundles. Every derived field is arithmetic on the source except four, which are inferred and labelled as such: `industry_derived`, `is_nonprofit`, `health_norm`, `service_type`.

It also carries a `column_verdicts` block — twelve fields with a plain-English verdict on how far each can be trusted. That is what the agent cites when it refuses.

## One thing to know about the source data

The amounts in the original file are randomly generated. They're safe to add up, but nothing correlates with anything — deal size doesn't predict win rate, client health doesn't predict payment behaviour. So this tool will sum, rank and compare, and it will refuse to tell you what *drives* anything. That constraint shaped the whole build.
