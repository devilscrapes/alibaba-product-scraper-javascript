<p align="center"><a href="https://apify.com/DevilScrapes/alibaba-products-scraper"><img src="assets/banner.svg" alt="Alibaba Products Scraper — JavaScript example" width="100%"></a></p>

# Alibaba Product Scraper for JavaScript and Node.js: supplier search to JSON and CSV

<p align="center">
<a href="https://apify.com/DevilScrapes/alibaba-products-scraper"><img alt="Run on Apify" src="https://img.shields.io/badge/Run%20on-Apify-DC2626?logo=apify&logoColor=white"></a>
<img alt="JavaScript" src="https://img.shields.io/badge/Node.js-20%2B-339933?logo=nodedotjs&logoColor=white">
<a href="LICENSE"><img alt="MIT license" src="https://img.shields.io/badge/license-MIT-0A0606"></a>
<a href="https://github.com/devilscrapes/alibaba-product-scraper-javascript/actions/workflows/ci.yml"><img alt="CI" src="https://github.com/devilscrapes/alibaba-product-scraper-javascript/actions/workflows/ci.yml/badge.svg"></a>
</p>

Use this repository to call the **[Alibaba Products Scraper](https://apify.com/DevilScrapes/alibaba-products-scraper)** from Node.js. It is a single ES module built on the official `apify-client` package: pass in search phrases, and you get back structured Alibaba.com offers with unit price, minimum order quantity, supplier name, image and product link. Apify runs the scraper in the cloud and handles the hard parts of collecting search pages, so your backend, serverless function or script only has to start a run and read the rows. Results land in `results.json` and `results.csv`, or stay in memory if you import the client call into your own app.

> **Actor:** [Alibaba Products Scraper](https://apify.com/DevilScrapes/alibaba-products-scraper) · **Publisher:** [Devil Scrapes](https://apify.com/DevilScrapes) · **Also in Python:** [alibaba-product-scraper-python](https://github.com/devilscrapes/alibaba-product-scraper-python)

## 🔥 What you get

- **One row per unique product**: product ID, title, price or price range, minimum order quantity, supplier name, image URL and product URL.
- **Many keywords per run**: every search term is paginated on its own, up to the limit you set.
- **No repeat rows**: sponsored offers that reappear on later result pages are removed across the whole run, so you are not billed twice for one product.
- **One bad term never sinks the run**: a term that fails is skipped and logged while the other terms finish.
- **Blocking handled for you**: the Actor detects Alibaba's challenge pages and retries on a fresh session instead of returning an empty page as "no results".
- **Honest zero results**: a term with no matching products succeeds with zero rows instead of failing the run.

## 🎯 Use cases

### Add supplier search to a sourcing app

Wire the core call into an Express route or a queue worker. A user types a product, your service starts a run and streams the offers back once it finishes. The rows are plain objects, so they drop straight into a React table or your API response.

### Refresh a product database on a schedule

Run `node index.js` from a GitHub Actions schedule, a cron container or a serverless timer and upsert each row by `product_id`. New suppliers and changed price ranges show up without anyone browsing Alibaba by hand.

### Compare wholesale cost with retail price

Pair these wholesale offers with marketplace prices from your other data sources to estimate margin per product before you contact a supplier. The search term on every row makes joins by keyword simple.

### Alert on new or cheaper offers

Keep yesterday's results, diff them against today's by `product_id` and post the new or cheaper offers to Slack or email. It is a small amount of JavaScript for a sourcing team's daily feed.

## ⚡ Quick start

You need an Apify account and an API token from [Console → Settings → Integrations](https://console.apify.com/settings/integrations). Apify's free plan includes monthly usage credit; this Actor's event prices are listed under Pricing below.

```bash
git clone https://github.com/devilscrapes/alibaba-product-scraper-javascript.git
cd alibaba-product-scraper-javascript
npm install
export APIFY_TOKEN=your_token_here
node index.js                       # uses input.json
node index.js my-input.json --out data --max-charge 2
```

The script starts the [Alibaba Products Scraper](https://apify.com/DevilScrapes/alibaba-products-scraper), waits for it to finish, prints a link to the run in Apify Console and saves every result to `results.json` and `results.csv`. It exits with code 1 and the run's status message if the run fails.

## 🧩 The core call

The integration itself is four statements. `call()` starts the Actor and resolves when the run finishes; `listItems()` fetches the rows from the run's default dataset.

```javascript
import { ApifyClient } from 'apify-client';

const client = new ApifyClient({ token: 'YOUR_APIFY_TOKEN' });
const run = await client.actor('DevilScrapes/alibaba-products-scraper').call({"searchTerms": ["bluetooth speaker"], "maxResultsPerTerm": 20, "proxyConfiguration": {"useApifyProxy": true}});
const { items } = await client.dataset(run.defaultDatasetId).listItems();
console.log(items);
```

## 📥 Input

`index.js` passes `input.json` to the Actor as it is. The example asks for one phrase and up to 20 products, a cheap way to see real output. List more phrases in `searchTerms` to collect several categories in one run.

```json
{
  "searchTerms": [
    "bluetooth speaker"
  ],
  "maxResultsPerTerm": 20,
  "proxyConfiguration": {
    "useApifyProxy": true
  }
}
```

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `searchTerms` | array | no | `["bluetooth speaker", "led strip light"]` | One or more free-text Alibaba search queries (e.g. bluetooth speaker, led strip light). Each term is fetched as its own paginated trade/search run. |
| `maxResultsPerTerm` | integer | no | `100` | Cap on unique products collected per search term. Values above ~50 force multi-page pagination with run-wide duplicate removal. |
| `proxyConfiguration` | object | no | `{"useApifyProxy": true}` | Apify Proxy spec — mandatory field. The Actor defaults to a RESIDENTIAL proxy pinned to the US when this is left at {"useApifyProxy": true} — the default (datacenter/auto) pool hit a served anti-bot challenge in cloud QA (2026-09-17). Override apifyProxyGroups/apifyProxyCountry here to change it. |

## 📤 Output

You get one object per product offer. Price strings are left as Alibaba displays them, including ranges and currency symbols, so your code decides how to parse and convert them.

| Field | Type | Description |
|---|---|---|
| `product_id` | string | Alibaba's internal product ID. |
| `search_term` | string | The search query that produced this row. |
| `title` | string | Product title as listed on the search results page. |
| `price` | string or null | Raw price or price-range string as served (mixed currency formats preserved). |
| `moq` | string or null | Minimum order quantity string, when present. |
| `supplier_name` | string or null | Supplier / company name, when present. |
| `image_url` | string or null | Primary listing image URL. |
| `product_url` | string | Absolute alibaba.com product detail page URL. |
| `page_number` | integer | 1-indexed search results page this row was found on. |
| `scraped_at` | string | UTC ISO-8601 timestamp captured when the page was fetched. |

## 💰 Pricing

| Event | Price (USD) | When it is charged |
|---|---|---|
| Actor start | $0.2 | Charged once when the Actor begins a run. |
| Result item | $0.006 | Charged once per validated product row pushed to the dataset. |

Prices as configured on 2026-09-17. The [Actor page](https://apify.com/DevilScrapes/alibaba-products-scraper) always shows the current price. Runs are billed to the Apify account that owns the token. Use `--max-charge` to cap what a single run can cost.

## 🛠️ Working with the data in JavaScript

This snippet groups the saved offers by supplier, keeps each supplier's cheapest listing and prints a ranked table. It uses only Node.js built-ins.

```javascript
import { readFile } from 'node:fs/promises';

const rows = JSON.parse(await readFile('results.json', 'utf8'));
const firstNumber = (text) => {
    const match = /\d[\d,]*(?:\.\d+)?/.exec(text ?? '');
    return match ? Number(match[0].replaceAll(',', '')) : null;
};

const bySupplier = new Map();
for (const row of rows) {
    const price = firstNumber(row.price);
    if (price === null) continue;
    const key = row.supplier_name ?? 'Unknown supplier';
    const best = bySupplier.get(key);
    if (!best || price < best.price) bySupplier.set(key, { price, moq: row.moq, url: row.product_url });
}

const ranked = [...bySupplier].sort((a, b) => a[1].price - b[1].price).slice(0, 20);
console.table(ranked.map(([supplier, offer]) => ({ supplier, ...offer })));
```

## ❓ FAQ

### Is there an official Alibaba API I should use instead?

Alibaba's official APIs are aimed at approved partners and sellers. This Actor works from public search results with nothing more than an Apify token, which suits research, sourcing tools and price monitoring.

### Can I use it with TypeScript?

Yes. `apify-client` ships its own type definitions. Copy the core call into a `.ts` file and the run and dataset types come with it, so your editor autocompletes fields like `defaultDatasetId`.

### How do I keep runs cheap while testing?

Keep `maxResultsPerTerm` low and pass a spending cap, for example `node index.js --max-charge 0.5`. Apify will not charge more than that amount for the run, whatever the input asks for.

### What happens when a search phrase finds nothing?

The run still succeeds. That term simply contributes zero rows, so a quiet niche never turns into a failed job in your pipeline.

## 🧯 Troubleshooting

- **`Set APIFY_TOKEN first`**: export the token in the same shell, or put it in a local `.env` file your shell loads. Never commit it.
- **Run finished with `FAILED` or `ABORTED`**: open the printed Console link; the run log says why. Input validation errors name the field.
- **Fewer rows than expected**: a `--max-charge` cap also limits how many results can be charged. Raise the cap or lower the limits in your input.
- **Something looks wrong in the data**: open an issue on the [Actor page](https://apify.com/DevilScrapes/alibaba-products-scraper) so the maintainers see it.

## 🔗 More from Devil Scrapes

- [Alibaba Products Scraper on Apify Store](https://apify.com/DevilScrapes/alibaba-products-scraper)
- [Python version of this example](https://github.com/devilscrapes/alibaba-product-scraper-python)
- [All Devil Scrapes Actors](https://apify.com/DevilScrapes)
- [Apify web scraping cookbook](https://github.com/devilscrapes/apify-web-scraping-cookbook)

Related Actors:

- [AliExpress Products Scraper](https://apify.com/DevilScrapes/aliexpress-products-scraper) — retail prices and orders for the same products on AliExpress.
- [ImportYeti Alternative Scraper](https://apify.com/DevilScrapes/importyeti-alternative-scraper) — check which suppliers actually ship to US importers.
- [DirectIndustry Industrial Suppliers Scraper](https://apify.com/DevilScrapes/directindustry-industrial-suppliers-scraper) — industrial equipment suppliers outside the Alibaba ecosystem.

---

MIT licensed example code. The Actor itself runs on [Apify](https://apify.com/DevilScrapes/alibaba-products-scraper) and is maintained by [Devil Scrapes](https://apify.com/DevilScrapes). Not affiliated with Alibaba.com or Alibaba Group.
