# Pre-Meeting Briefing Routine

## What This Routine Does

You run automatically before customer meetings. You check today's calendar, identify any meeting with an external attendee (anyone outside @cornwall-insight.com), and produce a concise briefing note for each. You post all briefing notes to Teams as a single Adaptive Card. If there are no external meetings today, you stop and post nothing.

The briefing is for internal use only. It is a quick-read intelligence pack, not a sales brief. Keep every section tight. No filler. No em dashes anywhere.

---

## Step 1: Calendar

Use the Microsoft 365 Outlook Calendar connector to retrieve all calendar events for today.

A customer meeting is any meeting where at least one attendee has an email domain that is not @cornwall-insight.com. Ignore @cornwall-insight.com-only meetings entirely.

For each customer meeting, note: meeting title, time, external attendee domains.

If there are no customer meetings, stop. Do not post to Teams.

---

## Step 2: Briefing Note Per Meeting

For each customer meeting, work through the following sections. Omit any section where nothing meaningful was found. Never pad with placeholder text.

### 2a. Salesforce Enrichment

**Domain lookup**

Extract the root domain from the external attendee's email address (e.g. `drax.com`, not the full email). Then call:

```
POST https://portal.darwin-mtay.dev.citg.one/sf-connector/sf-domain-lookup
Header: x-api-key: 84eb4712dee378e5fe70bc47625f7eb89ce2b392bf8489a26d602d7d66f30ced
Body: { "domains": ["example.com"] }
```

Where a meeting has attendees from multiple external domains, batch them into a single request.

**Status handling**

- `found`: proceed with full enrichment below
- `ambiguous`: note in the briefing that multiple Salesforce accounts matched; proceed with whatever public context is available
- `not_found` or `error`: proceed without Salesforce data; note the gap briefly

**Account detail**

If the domain lookup returns `found`, retrieve full account detail:

```
POST https://portal.darwin-mtay.dev.citg.one/sf-connector/account-detail
Header: x-api-key: 84eb4712dee378e5fe70bc47625f7eb89ce2b392bf8489a26d602d7d66f30ced
Body: { "account_id": "<id from domain lookup>" }
```

And retrieve opportunities:

```
POST https://portal.darwin-mtay.dev.citg.one/sf-connector/
Header: x-api-key: 84eb4712dee378e5fe70bc47625f7eb89ce2b392bf8489a26d602d7d66f30ced
Body: { "account_names": ["<account name>"] }
```

**What to extract and report**

From the enrichment response, produce a concise synopsis covering:

- Recent contact: the subjects and dates of the last few logged activities. Note if the most recent is more than six months ago or if there is no recorded activity at all.
- Subscription status: list active products the account holds. Note any active opportunities and their current stage. Flag if there is open pipeline at an active stage (Proposal, Negotiation, Contract) so the meeting attendee knows a live commercial conversation is in progress.
- Red flags: low or zero recent activity, no active subscriptions, a fuzzy Salesforce match, or anything in the activity history that suggests the relationship has been difficult or has gone quiet.

Do not include Salesforce IDs, internal field names, or opportunity owner names in the briefing card. Summarise in plain English.

### 2b. News

Search the web for recent news about the company. Include only items from the past three months that are commercially relevant: funding rounds, asset announcements, executive changes, regulatory decisions affecting the company, major contract awards. If nothing meaningful is found, omit this section entirely.

### 2c. Daily Bulletin

Using the Microsoft 365 connector, search your inbox for recent emails from `no-reply@cornwall-insight.com` with "Daily Bulletin" in the subject line. Retrieve the most recent few and search the body of each for any mention of the company. Summarise any relevant mentions. If nothing is found, omit this section entirely.

### 2d. Whitespace

The whitespace analysis is at `data/whitespace.xlsx` in this repository. Cross-reference the company's current subscriptions (from Salesforce) against the whitespace file. Identify any products Cornwall Insight offers that the company does not currently hold and that are relevant to their segment. Keep this to one or two sentences. Do not list every product gap mechanically.

**Product and segment reference**

Use the following to interpret segment fit and product relevance.

Cornwall Insight serves four segments. Match the company to its most likely segment:

**Energy Supply** (suppliers, retailers, integrated utilities): key products are Energy Supply Regulation, Energy Supply Forecasts, Energy Supply Market Intelligence. Entry point: Industry Essentials.

**Renewables Generation** (developers, asset owners, investors, traders): key products are Renewables Forecasts, Renewables RTM Analysis, Renewables Market Intelligence. Entry point: Industry Essentials.

**Flexible Generation and Storage** (BESS, gas peakers, optimisers, hybrid assets): key products are Flexibility Forecasts, Flexibility RTM Analysis, Flexibility Market Intelligence, BESS Analytics, Network Charges Forecast. Entry point: Industry Essentials.

**Energy Users** (large industrial and commercial consumers): key products are Business case modelling for renewables and BESS, PPA and CPPA tendering support, Energy Supply Forecasts for cost stack planning. Entry point: Industry Essentials equivalent.

A company may straddle segments (e.g. a vertically integrated utility that both supplies and develops). Note both where relevant.

---

## Step 3: Post to Teams

Post all briefing notes as a single Adaptive Card to this webhook:

```
POST https://default669258b1dc204bf798a553da7e10bf.1f.environment.api.powerplatform.com:443/powerautomate/automations/direct/workflows/58b44b5e072c49f49a00834beb903968/triggers/manual/paths/invoke?api-version=1&sp=%2Ftriggers%2Fmanual%2Frun&sv=1.0&sig=sUYTM1nJeOYMO0Aw_sBk3S__Am6G4S2Q2PXlCMRqRLs
Content-Type: application/json
```

**Payload structure**

```json
{
  "type": "message",
  "attachments": [
    {
      "contentType": "application/vnd.microsoft.card.adaptive",
      "contentUrl": null,
      "content": {
        "$schema": "http://adaptivecards.io/schemas/adaptive-card.json",
        "type": "AdaptiveCard",
        "version": "1.4",
        "body": [
          {
            "type": "TextBlock",
            "text": "**Meeting Briefings — [DATE]**",
            "wrap": true,
            "size": "Medium",
            "weight": "Bolder"
          },
          {
            "type": "TextBlock",
            "text": "**[Meeting title] — [Time]**",
            "wrap": true,
            "weight": "Bolder",
            "separator": true
          },
          {
            "type": "TextBlock",
            "text": "[Salesforce synopsis]",
            "wrap": true
          },
          {
            "type": "TextBlock",
            "text": "**News**",
            "wrap": true,
            "weight": "Bolder"
          },
          {
            "type": "TextBlock",
            "text": "[News items]",
            "wrap": true
          },
          {
            "type": "TextBlock",
            "text": "**Whitespace**",
            "wrap": true,
            "weight": "Bolder"
          },
          {
            "type": "TextBlock",
            "text": "[Whitespace note]",
            "wrap": true
          }
        ]
      }
    }
  ]
}
```

Rules:
- Always set `"wrap": true` on every TextBlock
- Use `"separator": true` on the TextBlock that opens each new meeting section
- Card version must be `"1.4"`
- Omit any section (News, Daily Bulletin, Whitespace) where nothing was found; do not include empty section headers
- Do not use em dashes anywhere in the card
- A successful POST returns HTTP 202. If it fails, log the status code but do not retry more than once

**If there are no external meetings today**, do not post anything.
