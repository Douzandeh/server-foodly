# Recipes API (JSON Server on Vercel)

A zero-config mock REST API serving a collection of 15 world dishes (recipes, ingredients, prices, discounts). Built with [JSON Server](https://github.com/typicode/json-server) and deployed on [Vercel](https://vercel.com) as a serverless function, so front-end projects can consume realistic data without standing up a backend.

![Node](https://img.shields.io/badge/node-%3E%3D12-339933?logo=node.js&logoColor=white)
![json-server](https://img.shields.io/badge/json--server-0.17-black)
![Deploy](https://img.shields.io/badge/deploy-Vercel-000?logo=vercel)
![License](https://img.shields.io/badge/license-MIT-blue)

---

## Table of Contents

- [Features](#features)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [API Reference](#api-reference)
- [Data Model](#data-model)
- [Consuming the API](#consuming-the-api)
- [Deployment](#deployment)
- [Known Limitations](#known-limitations)
- [Contributing](#contributing)
- [License](#license)

---

## Features

- Full REST routes generated automatically from `db.json`
- Filtering, full-text search, sorting, pagination and slicing out of the box
- CORS enabled by default, so it works straight from any front-end origin
- One-command local dev, one-command deploy
- Custom `server.js` exporting the Express app, which is what Vercel's Node runtime expects

## Tech Stack

| Layer      | Tool                                                      |
| ---------- | --------------------------------------------------------- |
| Runtime    | Node.js                                                   |
| API        | [json-server](https://github.com/typicode/json-server) `^0.17.0` |
| Hosting    | Vercel (`@vercel/node` serverless function)               |
| Data store | Flat JSON file (`db.json`)                                |

## Project Structure

```
.
├── db.json          # Data source; top-level key "data" becomes the /data resource
├── server.js        # JSON Server entry point (exports the app for Vercel)
├── vercel.json      # Build + routing config; bundles db.json with the function
├── package.json
├── LICENSE
└── .gitignore
```

## Getting Started

### Prerequisites

- Node.js 12 or newer (the latest LTS is recommended)
- npm or pnpm

### Installation

```bash
git clone <your-repo-url>
cd <your-repo-folder>
npm install
```

### Run locally

```bash
npm start
```

The server starts on **http://localhost:3000**. Try it:

```bash
curl http://localhost:3000/data/1
```

## API Reference

All data lives under the `/data` resource (the top-level key in `db.json`).

| Method | Endpoint     | Description                  |
| ------ | ------------ | ---------------------------- |
| GET    | `/data`      | List all dishes              |
| GET    | `/data/:id`  | Get a single dish            |
| POST   | `/data`      | Create a dish (see [limitations](#known-limitations)) |
| PUT    | `/data/:id`  | Replace a dish               |
| PATCH  | `/data/:id`  | Partially update a dish      |
| DELETE | `/data/:id`  | Remove a dish                |

### Query examples

```bash
# Pagination (the X-Total-Count header contains the total)
GET /data?_page=1&_limit=5

# Sorting
GET /data?_sort=price&_order=desc

# Filtering by exact value
GET /data?discount=0

# Range filters
GET /data?price_gte=10&price_lte=20

# Partial match on a field
GET /data?name_like=chicken

# Full-text search across all fields
GET /data?q=noodle

# Slicing
GET /data?_start=0&_end=3
```

See the [json-server docs](https://github.com/typicode/json-server/tree/v0.17.4#routes) for the complete list of operators.

## Data Model

```jsonc
{
  "id": 1,
  "name": "Beef Wellington",
  "price": 13,                 // number, base price
  "discount": 30,              // number, percentage (0 = no discount)
  "introduction": "…",         // string, short description of the dish
  "details": [                 // array of single-key objects
    { "Cuisine": "British" },
    { "Recipe Type": "Beef" },
    { "Difficulty": "Medium" },
    { "Preparation Time": "50 mins" },
    { "Cooking Time": "45 mins" },
    { "Serves": "8" }
  ],
  "ingredients": ["1.5 kg beef fillet, trimmed and tied", "…"],
  "recipe": ["Season the beef with black pepper…", "…"]
}
```

> **Tip for consumers:** `details` is an array of single-key objects rather than a flat object. Normalize it once on the client:
>
> ```js
> const details = Object.assign({}, ...dish.details);
> // { Cuisine: "British", "Recipe Type": "Beef", ... }
> ```

## Consuming the API

### Fetch

```js
const BASE_URL = "https://<your-project>.vercel.app";

export async function getDishes({ page = 1, limit = 6 } = {}) {
  const res = await fetch(`${BASE_URL}/data?_page=${page}&_limit=${limit}`);
  if (!res.ok) throw new Error(`Request failed: ${res.status}`);

  return {
    dishes: await res.json(),
    total: Number(res.headers.get("X-Total-Count")),
  };
}
```

### Discounted price helper

```js
export const finalPrice = ({ price, discount }) =>
  +(price * (1 - discount / 100)).toFixed(2);
```

### React example

```jsx
import { useEffect, useState } from "react";

function DishList() {
  const [dishes, setDishes] = useState([]);

  useEffect(() => {
    const controller = new AbortController();

    fetch(`${BASE_URL}/data`, { signal: controller.signal })
      .then((res) => res.json())
      .then(setDishes)
      .catch((err) => err.name !== "AbortError" && console.error(err));

    return () => controller.abort();
  }, []);

  return (
    <ul>
      {dishes.map((d) => (
        <li key={d.id}>{d.name}: ${finalPrice(d)}</li>
      ))}
    </ul>
  );
}
```

## Deployment

The project is preconfigured for Vercel.

1. Push the repository to GitHub, GitLab or Bitbucket.
2. Import it in the [Vercel dashboard](https://vercel.com/new), or deploy from the CLI:

   ```bash
   npm i -g vercel
   vercel        # preview deployment
   vercel --prod # production deployment
   ```

How it works:

- `vercel.json` builds `server.js` with `@vercel/node` and uses `includeFiles` to bundle `db.json` with the function.
- All routes (`/(.*)`) are forwarded to `server.js`.
- `server.js` exports the Express server (`module.exports = server`), which Vercel invokes per request. The `listen(3000)` call is only used for local development.

## Known Limitations

- **Writes do not persist on Vercel.** Serverless functions have a read-only, ephemeral filesystem, so `POST`/`PUT`/`PATCH`/`DELETE` will either fail or be lost between invocations. Treat the deployed API as **read-only**. For persistent writes, run it locally or swap the data layer for a real database.
- **No authentication or rate limiting.** Don't put sensitive data in `db.json`.
- **Cold starts** may add latency to the first request after a period of inactivity.

## Contributing

1. Fork the repo and create a feature branch: `git checkout -b feat/my-change`
2. Keep `db.json` valid JSON and keep `id` values unique and sequential
3. Test locally with `npm start`
4. Open a pull request describing your change

## License

Released under the [MIT License](./LICENSE). © 2026 Hossein
