# AI Real Estate Lead Qualifier

An n8n workflow that automatically classifies real estate leads by intent and urgency, then routes each one to the right next step — agent notification, customer follow-up, or live property matches.

## What it does

Leads submitted through a Google Form are classified by an AI agent as **hot**, **warm**, or **cold** based on budget, timeline, and pre-approval status. Each classification triggers a different automated path:

- 🔴 **Hot** → Urgent internal email notifies the agent immediately, with the AI's reasoning included
- 🟡 **Warm** → Customer receives a follow-up email letting them know an agent will be in touch
- 🔵 **Cold** → Workflow queries the Rentcast API and emails the customer 3 curated property matches (rent or buy pricing, based on stated intent)

Every lead is logged to Google Sheets with a status update once processing completes.

## Architecture

```
Google Form → Webhook → AI Lead Qualifier → Switch (hot / warm / cold)
                                                ├─ HOT  → Gmail (Agent)    → Sheets
                                                ├─ WARM → Gmail (Customer) → Sheets
                                                └─ COLD → Rentcast API → Rent/Buy branch → Gmail (Customer) → Sheets
```

## Tools

n8n (self-hosted, Docker) · Groq & OpenRouter · Rentcast API · Gmail · Google Sheets · Google Forms

## Notes

- Full write-up, including bugs encountered and how they were fixed, is in [`documentation.md`](./documentation.md)
- Known limitations: no fallback for low/zero search results yet, no error handling on failed API calls, Rentcast free tier caps at 50 requests/month
