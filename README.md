# SellSmart

Price a resale item from a photo or a keyword, using real eBay market data.

SellSmart pulls two independent pricing signals for an item — **live listings** from the eBay Browse API and **completed/sold listings** scraped from eBay search — and exposes them behind a single GraphQL endpoint. Sold comps are the honest signal: they reflect what buyers actually paid, not what sellers are asking.

> **Status: early prototype.** The eBay integration (keyword + reverse-image search) and the sold-comps scraper are working. The AI valuation layer is scaffolded but not yet implemented — `searchByKeyword` and `searchByImage` currently return listings plus a placeholder where the analysis will go. See [Roadmap](#roadmap).

---

## What it does

- **Reverse-image search** — send a base64 image, get matching eBay listings back via the Browse API's `search_by_image` endpoint. No manual titling or category picking.
- **Keyword search** — text query with optional price-range filtering and pagination.
- **Sold-comp pricing** — scrapes completed and sold eBay listings (`LH_Complete=1&LH_Sold=1`) and extracts a clean list of realized prices, filtering out zero-price and promotional rows.
- **One GraphQL surface** — both search paths are mutations on a single `/graphql` endpoint, so a client (web or mobile) talks to one API.

---

## Architecture

```
          photo / keyword
                 │
                 ▼
     ┌───────────────────────┐
     │  server.js            │   Express + graphql-http
     │  POST /graphql :4000  │
     └───────────┬───────────┘
                 │
        ┌────────┴────────┐
        ▼                 ▼
┌───────────────┐  ┌──────────────────┐
│ ebay_client   │  │ scraper_client   │
│ Browse API    │  │ sold/completed   │
│ OAuth2        │  │ via ScrapeOps    │
│ keyword+image │  │ BeautifulSoup    │
└───────────────┘  └──────────────────┘
      live asks          realized prices
```

The Node layer is the API surface. The Python modules are the data clients — each is independently runnable and testable, which keeps the eBay auth flow and the scraping logic out of the request path while they're still changing.

---

## Stack

| Layer | Tech |
|---|---|
| API | Node.js, Express 5, `graphql` + `graphql-http` |
| Marketplace data | eBay Browse API (OAuth2 client-credentials), `httpx` |
| Comps scraping | BeautifulSoup4 + lxml, ScrapeOps proxy |
| Planned AI layer | `google-generativeai` (Gemini), FastAPI + Strawberry GraphQL |

---

## Getting started

### Prerequisites

- Node.js 18+
- Python 3.9+
- An [eBay developer account](https://developer.ebay.com/) with production keys
- A [ScrapeOps](https://scrapeops.io/) API key (used as the scraping proxy)

### 1. Clone and install

```bash
git clone https://github.com/AdityaDREXEL/Sell-Smart.git
cd Sell-Smart

npm install
pip install -r requirements.txt
```

### 2. Configure environment

Create a `.env` file in the project root:

```env
# eBay Browse API (production keys)
EBAY_PROD_APP_ID=your_app_id
EBAY_PROD_CERT_ID=your_cert_id

# ScrapeOps proxy — required by scraper_client.py
SCRAPEOPS_API_KEY=your_scrapeops_key

# Gemini — used by the planned AI valuation layer
GOOGLE_API_KEY=your_google_api_key

PORT=4000
```

Verify your keys are loading:

```bash
python test_env.py
```

### 3. Run the API

```bash
node server.js
```

The GraphQL endpoint is now at `http://localhost:4000/graphql`.

---

## API

Health check:

```graphql
query {
  hello
}
```

Search by keyword, with a price floor and ceiling:

```graphql
mutation {
  searchByKeyword(
    query: "Sony WH-1000XM4"
    minPrice: 80
    maxPrice: 250
    limit: 20
  ) {
    items {
      title
      price
      imageUrl
      link
    }
    analysis
  }
}
```

Search by image — pass a base64-encoded photo:

```graphql
mutation {
  searchByImage(image: "<base64-encoded-image>", limit: 20) {
    items {
      title
      price
      imageUrl
      link
    }
    analysis
  }
}
```

The `analysis` field is the seam for the valuation layer. It currently returns a placeholder string.

---

## Project structure

```
server.js             Express + GraphQL server, search mutations
ebay_client.py        eBay Browse API: OAuth2 token, keyword search, image search
scraper_client.py     Sold/completed listing scraper, returns realized prices
messenger_client.py   Messaging client
test_env.py           Environment variable check
requirements.txt      Python dependencies
package.json          Node dependencies
```

---

## Roadmap

- [ ] **Valuation model** — combine live asks and sold comps into a single price estimate with a confidence range, replacing the `analysis` placeholder
- [ ] **Retrieval over comps** — ground the model's reasoning in the scraped sold listings rather than model priors alone
- [ ] **Parallel comp fetching** — the scraper is currently sequential; move to concurrent requests to cut latency
- [ ] **Marketplace recommendation** — compare expected net across eBay, Mercari, Poshmark
- [ ] **Fee and shipping math** — subtract platform fees and shipping to produce net profit, not gross price
- [ ] **Persistence** — store lookups and price history so repeat items resolve instantly
- [ ] **Frontend** — photo capture and results UI

---

## Notes

`package.json` declares `index.js` as its entry point, but the server lives in `server.js` — run it directly until that's reconciled.

The scraper depends on eBay's current search markup and is proxied through ScrapeOps. Treat it as best-effort and expect to adjust selectors when eBay changes their pages.

---

## License

Not yet licensed. Add one before reuse.
