---
title: "How I Built a WhatsApp AI CRM in Google Sheets for ~$100/Month (Twilio + OpenAI)"
seoTitle: "WhatsApp AI CRM in Google Sheets (Twilio + OpenAI)"
seoDescription: "Build a WhatsApp AI CRM in Google Sheets with Apps Script, Twilio & GPT-4 for ~$100/mo — full architecture, tool design & cost breakdown."
datePublished: 2026-07-10T21:54:51.827Z
cuid: cmrfh4jyq000009ko2nue4n99
slug: how-i-built-a-whatsapp-ai-crm-in-google-sheets-for-100-month-twilio-openai
canonical: https://magesheet.com/blog/whatsapp-ai-crm-google-sheets
cover: https://cdn.hashnode.com/uploads/covers/69f77ba076c1469ba4cc3db6/191fd123-24ac-4986-aa83-8cbf80d8c420.jpg
ogImage: https://cdn.hashnode.com/uploads/og-images/69f77ba076c1469ba4cc3db6/2ad8cfeb-fb28-4356-98ed-774efe5aae9e.jpg
tags: automation, google-sheets, google-apps-script, openai, magesheet

---

Most businesses overpay for their CRM.

You're paying $200–600 per seat, per month, for HubSpot or Salesforce — and your customers still wait hours for a WhatsApp reply because the "integration" is a rigid, keyword-based chatbot that can't actually *do* anything.

So we built the opposite: a **WhatsApp AI CRM that lives entirely inside Google Sheets**, powered by Google Apps Script, the Twilio WhatsApp Business API, and OpenAI's GPT-4. Total cost: around $100/month for 2,000–3,000 conversations. No SaaS subscription. No database migration. Your sales data stays in the spreadsheet your team already knows.

Here's the part most tutorials get wrong.

## A chatbot answers. An AI agent acts.

The unlock isn't GPT generating nice text — it's **OpenAI function calling (tools)**. When a customer says "can I book a demo Thursday?", GPT-4 doesn't just reply politely. It recognizes the intent and *calls a function* inside your Apps Script: it checks the calendar, writes the appointment to a Sheet, and confirms — in seconds.

That single capability turns a spreadsheet into an autonomous CRM.

## The architecture that keeps it from blowing up

Naively wiring GPT straight to your data is how you get duplicate charges and corrupted rows. The pattern that works is a layered flow:  
Customer → Twilio (WhatsApp API) → Apps Script doPost (webhook)  
→ Context loader (reads Customers + Conversations + Orders)  
→ AI brain (GPT-4 + tool library)  
→ Action executor (the ONLY layer with write access)  
  
The webhook returns `200` fast to dodge the timeout. The AI *decides* but never writes directly. A single action-executor layer handles all writes — with idempotency keys and an audit log — so the AI can't accidentally refund the same order twice.

## The tool library is where the magic hides

You give the model three tiers of tools:

*   **Read tools** (safe, free to call): `get_customer`, `get_orders`, `get_inventory`, `search_faq`
    
*   **Write tools** (logged, idempotent): `create_lead`, `update_status`, `book_appointment`
    
*   **Escalation tools** (human-gated): `request_human`, `process_refund` — *never* auto-executed
    

One hard-won lesson: the `description` field on each tool schema matters more than the code. It's what teaches the model *when* to reach for each function.

## The trick that cuts the AI bill 5–6×

Running every message through GPT-4 is what makes people say "AI is too expensive." The fix is **model routing**: a tiny GPT-4o-mini classifier (~$0.0001/message) sorts each message into FAQ / order\_status / new\_lead / complaint / escalation. The cheap 80% never touches GPT-4.

Result: 3,000 messages/month drops from ~$90–150 to ~$15–25.

## What's in the full guide

The complete build on the MageSheet blog covers the parts I couldn't fit here: the stateless-memory problem (and the conversation-summarization fix), per-customer profile memory, the real cost breakdown vs. HubSpot, data-compliance notes (SOC 2 / GDPR), and where Apps Script's ~50–100 concurrent-conversation ceiling forces a move to Cloud Run.

**Read the full guide with the complete architecture and tool schemas here:** 👉 [https://magesheet.com/blog/whatsapp-ai-crm-google-sheets](https://magesheet.com/blog/whatsapp-ai-crm-google-sheets)

If you've built something similar in Sheets, I'd love to hear how you handled memory — drop a comment.