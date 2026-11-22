# Website email scraper: emails, phones and social links from a list of websites

How to turn a list of company websites into contact rows without an API key or a login. Each row has the emails the site publishes, its phone numbers and social profiles, the contact page and whether it has a form, and the platform the site runs on (Shopify, WordPress, HubSpot and others).

The data comes from the Apify Actor [Website Email Scraper & Contact Finder](https://apify.com/renzomacar/website-contact-finder). It reads the home page first, then the contact, about, imprint and team pages, and stops after a few pages per site.

## Why read the website instead of an email database

Email-finder databases sell addresses that are guessed or out of date, and charge per lookup. The addresses a company publishes on its own website are the ones it actually answers: the wholesale inbox, the press contact, the support desk. Reading them from the site also tells you where each one was found, so you can check it.

## Example: wholesale contacts of 12 specialty coffee roasters

The use case is a café or grocery buyer looking for wholesale coffee suppliers. Input:

```json
{
  "domains": [
    "counterculturecoffee.com", "onyxcoffeelab.com", "vervecoffee.com", "ritualcoffee.com",
    "heartroasters.com", "georgehowellcoffee.com", "sightglasscoffee.com", "coavacoffee.com",
    "madcapcoffee.com", "equatorcoffees.com", "temple-coffee.com", "olympiacoffee.com"
  ],
  "maxPagesPerDomain": 4
}
```

Run on 2026-11-22: 39 seconds. 11 websites were read over plain HTTP, with no browser needed. The twelfth did not answer and got no row.

| Roaster | Emails | Wholesale inbox | Phones | Social profiles | Contact form | Runs on |
|---|---|---|---|---|---|---|
| Onyx Coffee Lab | 5 | wholesale@onyxcoffeelab.com | 1 | 5 | no | Shopify |
| George Howell Coffee | 4 | - | 1 | 4 | yes | Shopify |
| Verve Coffee Roasters | 3 | wholesale@vervecoffee.com | 13 | 2 | yes | Shopify |
| Heart Coffee Roasters | 2 | - | 2 | 3 | no | Shopify |
| Counter Culture Coffee | 2 | - | 0 | 3 | no | Shopify, WordPress |
| Coava Coffee Roasters | 1 | - | 0 | 4 | no | - |
| Sightglass Coffee | 1 | - | 0 | 4 | no | Shopify |
| Madcap Coffee Company | 1 | - | 1 | 5 | yes | Shopify, HubSpot |
| Equator Coffees | 1 | - | 1 | 3 | yes | Shopify |
| Ritual Coffee Roasters | 0 | - | 4 | 4 | yes | WordPress |
| Olympia Coffee Roasting Company | 0 | - | 0 | 6 | no | Shopify |

What the table shows:

- 9 of the 11 publish at least one email. Ritual and Olympia publish none, but they can still be reached by phone, contact form or social profile.
- 9 of the 11 run on Shopify, which matters if you sell a Shopify app or service to this niche.
- Verve lists 13 phone numbers, one per café.
- Onyx's support page links to `info@onyxcoffelab.com`, which is missing an "e". The row marks that address with `"onCompanyDomain": false`, so you can catch typos like this before they reach your outreach list.

All 11 rows are in [sample-output.json](sample-output.json). One personal address was removed; every other value is as the Actor returned it. One row:

```json
{
  "domain": "heartroasters.com",
  "companyName": "Heart Coffee Roasters",
  "emails": ["coffee@heartroasters.com", "work@heartroasters.com"],
  "emailDetails": [
    { "email": "coffee@heartroasters.com", "type": "other", "onCompanyDomain": true, "foundIn": "page", "sourceUrl": "https://www.heartroasters.com/", "hasMxRecord": true },
    { "email": "work@heartroasters.com", "type": "other", "onCompanyDomain": true, "foundIn": "page", "sourceUrl": "https://www.heartroasters.com/pages/contact", "hasMxRecord": true }
  ],
  "phones": ["+1 503 206 6602", "+1 503 208 2710"],
  "socialLinks": {
    "twitter": "https://twitter.com/heartroasters",
    "facebook": "https://www.facebook.com/HeartRoasters",
    "instagram": "https://www.instagram.com/heartroasters"
  },
  "contactPageUrl": "https://www.heartroasters.com/pages/contact",
  "hasContactForm": false,
  "techStack": ["Cloudflare", "Shopify", "jQuery"],
  "pagesCrawled": 4,
  "method": "http",
  "status": "found"
}
```

- `emailDetails` gives, for each address:
  - whether it is a shared mailbox (`type: "role"`, such as info@ or sales@);
  - whether it is on the company's own domain (`onCompanyDomain`);
  - where it was found (`foundIn`: the page text or a `mailto` link) and on which page (`sourceUrl`);
  - whether that domain accepts mail (`hasMxRecord`).
- `method` is `http` or `browser`. Most sites are read with plain requests; a browser is opened only for the sites that need one.
- `status` is `found` when the site publishes at least one contact detail.

## Call it from code

You need an Apify account and its API token.

```bash
curl -X POST "https://api.apify.com/v2/acts/renzomacar~website-contact-finder/run-sync-get-dataset-items?token=$APIFY_TOKEN" \
  -H "content-type: application/json" \
  -d '{"domains":["heartroasters.com","madcapcoffee.com"]}'
```

Python: read a list of domains and write a CSV with one row per email, keeping only addresses on the company's own domain:

```python
import csv
from apify_client import ApifyClient

domains = [line.strip() for line in open("domains.txt") if line.strip()]
client = ApifyClient("YOUR_APIFY_TOKEN")
run = client.actor("renzomacar/website-contact-finder").call(run_input={"domains": domains})

with open("emails.csv", "w", newline="", encoding="utf-8") as f:
    w = csv.writer(f)
    w.writerow(["company", "domain", "email", "type", "source"])
    for site in client.dataset(run["defaultDatasetId"]).iterate_items():
        for e in site["emailDetails"]:
            if e["onCompanyDomain"]:
                w.writerow([site["companyName"], site["domain"], e["email"], e["type"], e["sourceUrl"]])
```

JavaScript: print only the sites that run on Shopify:

```js
import { ApifyClient } from "apify-client";

const client = new ApifyClient({ token: process.env.APIFY_TOKEN });
const run = await client.actor("renzomacar/website-contact-finder").call({
  domains: ["onyxcoffeelab.com", "ritualcoffee.com", "vervecoffee.com"],
});
const { items } = await client.dataset(run.defaultDatasetId).listItems();
for (const s of items.filter((s) => s.techStack.includes("Shopify"))) {
  console.log(s.domain, s.emails.join(", ") || s.contactPageUrl);
}
```

## Use it from Claude, ChatGPT or another AI agent (MCP)

```json
{
  "mcpServers": {
    "apify": {
      "url": "https://mcp.apify.com/?tools=renzomacar/website-contact-finder",
      "headers": { "Authorization": "Bearer YOUR_APIFY_TOKEN" }
    }
  }
}
```

Then ask, for example: "Find the contact emails of these 20 websites and tell me which ones have a wholesale or partnerships inbox."

## Input fields

| Field | What it does |
|---|---|
| `domains` | Websites, as domains or URLs. A URL is reduced to its host. |
| `maxPagesPerDomain` | Pages read per site (default 4): home page first, then the contact, about, imprint and team pages. |
| `includeGenericEmails` | Turn off to leave out shared mailboxes (info@, support@, noreply@ and similar). |
| `detectTechStack` | Detects the platform the site runs on. |
| `onlyWithContacts` | Returns only the sites that publish a contact detail. |
| `includeUnreachable` | Also returns a row for sites that did not answer or sit behind an anti-bot check. |
| `maxConcurrency` | How many sites are opened in a browser at the same time, for the few that need one. |

## Limits

- It returns what each website publishes. It does not guess addresses from names or look people up.
- Sites that do not answer, or that sit behind an anti-bot check, get no row unless `includeUnreachable` is on. The run log lists them.
- Contact details shown only as images, or only after a form is filled in, are not read.
- Use the addresses within the law that applies to you (CAN-SPAM, GDPR and similar).

## Price

You pay per website, at the price shown on the Actor page.
