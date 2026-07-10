# AI Real Estate Lead Qualifier

## 1. Problem

Real estate agents receive inbound leads of widely varying quality — some are ready to transact within weeks and pre-approved for financing, while others are just browsing with no timeline or budget clarity. Manually reading and prioritizing every inquiry wastes an agent's time on low-intent leads while high-intent leads risk going cold waiting for a response.

This project automates that triage. It reads incoming leads from a form submission, classifies each one by intent and urgency using AI, and routes it to the appropriate next step automatically — whether that's an urgent notification to an agent, a follow-up email to the customer, or a curated list of matching properties pulled live from a real estate data API.

## 2. Architecture

```
Google Form → Webhook → AI Lead Qualifier → Switch (hot / warm / cold)
                                                ├─ HOT  → AI Email Assistant → Gmail (Agent) → Sheets (log)
                                                ├─ WARM → AI Email Assistant → Gmail (Customer) → Sheets (log)
                                                └─ COLD → Rentcast API → Rent/Buy branch → Gmail (Customer) → Sheets (log)
```

**Flow breakdown:**

The workflow starts with a **Webhook** node connected to a Google Form. On submission, an **AI Lead Qualifier** agent reads the lead's details — budget, timeline, pre-approval status, preferred location, and property type — and classifies the lead as **hot**, **warm**, or **cold**, along with a written reason for that classification. A **Switch** node then routes the lead down one of three paths based on that classification.

**If HOT:**
- An AI Email Assistant checks the classification and recipient type, and generates a subject line.
- Since the recipient is the agent, the email is sent through a dedicated Gmail node as an "Urgent" internal notification containing the lead's full details and the AI's reasoning for the hot classification.
- Once sent, a Google Sheets node updates the lead's STATUS column to "Done."

**If WARM:**
- The AI Email Assistant checks that the recipient is the customer and generates a subject line.
- A static, templated email is sent to the customer via a second Gmail node, letting them know an agent will follow up within 2–3 business days.
- Once sent, the Google Sheets STATUS column is updated to "Done."

**If COLD:**
- An HTTP Request node queries the Rentcast API for property listings matching the lead's preferred city, state, and property type.
- An If node checks whether the lead's stated purpose is renting or buying.
- If **buying**, a second HTTP Request retrieves current sale/value data for matching properties.
- If **renting**, a second HTTP Request retrieves monthly rent data for matching properties instead.
- A Set node extracts and formats the relevant fields per property (address, bedrooms, bathrooms, square footage, price/rent, and price range).
- One of two Gmail nodes (depending on rent vs. buy) sends the customer a curated list of 3 property matches with pricing details.
- The Google Sheets STATUS column is updated to "Done."

## 3. Tools Used

- **n8n** (self-hosted via Docker on Windows, exposed publicly through ngrok)
- **Groq** and **OpenRouter** (AI lead classification and email drafting)
- **Rentcast API** (property listing, rent, and sale price data)
- **Gmail** (sending agent notifications and customer quotes)
- **Google Sheets** (lead logging and status tracking)
- **Google Forms** (lead intake)

## 4. Problems Solved

**Problem:** Rentcast property searches returned zero results despite correct city/state spelling.
**Cause:** Rentcast's API matching is case-sensitive — a lowercase or inconsistently cased city/state value silently returned an empty result set instead of an error.
**Fix:** Added a proper-case transform on the city field (`.trim().split(' ').map(...).join(' ')`) and switched the state field from free text to a dropdown of valid two-letter abbreviations, eliminating the casing issue at the source.

**Problem:** Even with correct casing, some cities still returned zero results (e.g. "Salt Lake City").
**Cause:** Rentcast's internal dataset stores some city names differently than their common form — "Salt Lake City" is stored simply as "Salt Lake."
**Fix:** Added a suffix-strip transform (`.replace(/\s+City$/i, '')`) to normalize common naming mismatches before querying the API.

**Problem:** Customer emails were displaying literal `\n` characters instead of line breaks.
**Cause:** The Gmail node's Email Type was set to HTML, which ignores plain newline characters and only respects HTML tags.
**Fix:** Replaced `\n` with `<br>` tags in the email templates so line breaks rendered correctly under HTML mode.

**Problem:** Attempting to list multiple properties in a single email by indexing (`$json.formattedAddress[0]`, `[1]`, `[2]`) returned individual characters instead of separate property records.
**Cause:** `$json` always refers to the single current item being processed — it has no awareness of sibling items in the same execution, so indexing into it indexes into the string itself rather than an array of items.
**Fix:** Switched to `$input.all()[index].json.fieldName`, which correctly accesses the full array of items flowing into the node, allowing each of the 3 properties to be referenced individually within one combined email.

## 5. Limitations & Next Steps

- **No fallback handling yet** for searches that return fewer than 3 properties (or zero). Currently the email simply sends with whatever is available; a "no exact matches, an agent will follow up" fallback message is planned but not yet built.
- **No error catching** around failed API calls (e.g. Rentcast rate limits, malformed responses) — planned as a follow-up once the core workflow is fully polished.
- **Property type coverage is limited to what Rentcast supports** (Single Family, Condo, Townhouse, Manufactured, Multi-Family, Apartment, Land) — leads requesting commercial space (office, retail, industrial) fall outside what this workflow can currently fulfill, since Rentcast has no equivalent listings data.
- **Rentcast free-tier API limit is 50 requests/month**, which constrains how much live testing and real-world usage this workflow can currently support without a paid plan.
