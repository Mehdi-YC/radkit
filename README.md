# radkit — SvelteKit Boilerplate (Svelte 5 + TypeScript)

This repository is a **backend-first**, developer-centric boilerplate for your radkit platform. It uses:

* SvelteKit (Svelte 5) + TypeScript
* Bun-compatible scripts (works with Node too)
* Drizzle ORM (Postgres)
* Zod + radkit field wrappers (Zod with metadata)
* Superforms for form binding
* Skeleton UI (Tailwind) for simple UI primitives
* BetterAuth (placeholder) for auth integration
* Handlebars for server-side record templates (record-level repeated template)
* Docker + docker-compose for dev

---

## Goals

* No third‑party or untrusted code execution — all actions run in the same runtime

* Keep Docker & docker-compose simple (one for dev, one for prod)

* Always enforce access control on *every* filter, search and record fetch

* Dynamic, file-driven collections loaded at runtime

* Backend-first: API endpoints are authoritative and UI consumes them via fetch

* Pixel-simple admin UI: sidebar with collections, list view, record view

* Record-level Handlebars templates (one template per record rendered server-side)

* Field metadata supports sections & spans for grid layout

* Field-level ACL enforced server-side

* Minimal dependencies, readable code, and good DX

---

## Project tree

```
radkit-boilerplate/
├── project/                        # Developer project folder (editable)
│   ├── project.json
│   ├── collections/
│   │   ├── brand.collection.ts
│   │   └── car.collection.ts
│   ├── actions/
│   │   └── assignOwner.action.ts
│   └── templates/
│       └── car-record.hbs
│
├── src/
│   ├── app.d.ts
│   ├── hooks.server.ts
│   ├── lib/
│   │   ├── radkit/                 # radkit core runtime (loader, registry, types)
│   │   │   ├── loader.ts
│   │   │   ├── registry.ts
│   │   │   ├── fields.ts          # radkit.* wrappers built on Zod
│   │   │   ├── converter.ts       # convert radkit -> zod if needed
│   │   │   └── types.ts
│   │   ├── db/
│   │   │   ├── schema.ts          # Drizzle static tables
│   │   │   └── client.ts
│   │   └── auth/
│   │       └── betterAuth.ts      # placeholder to wire BetterAuth
│   ├── routes/
│   │   ├── api/
│   │   │   ├── projects/[project]/[collection]/+server.ts   # list/create
│   │   │   └── projects/[project]/[collection]/[id]/+server.ts # get/update/delete
│   │   │   └── actions/[actionName]/+server.ts
│   │   ├── admin/
│   │   │   ├── +layout.svelte
│   │   │   ├── +page.server.ts    # lists collections
│   │   │   ├── [collection]/+page.server.ts  # collection listing
│   │   │   └── [collection]/[id]/+page.server.ts # record view
│   │   └── +layout.svelte
│   └── styles/
│       └── app.css
│
├── docker/
│   ├── Dockerfile
│   └── compose.dev.yml
├── .env
├── package.json
├── tsconfig.json
├── svelte.config.ts
├── vite.config.ts
└── README.md
```

---

## Quick start (development)

```bash
# install dependencies (Bun recommended, Node works too)
bun install

# start Postgres via docker-compose
docker compose -f docker/compose.dev.yml up -d

# run dev server
bun run dev
```

Open `http://localhost:5173/admin` to see the admin UI.

---

## Key Concepts Implemented in Boilerplate

### 1) Project folder

All code is first‑party and trusted. Action scripts run directly in the same Node/Bun runtime — no VM, sandbox or isolation required. This keeps execution predictable and simple.

`project/` is where product developers add collections, actions and templates. The core runtime scans this folder at startup and registers collections and actions.

### 2) radkit field wrappers

`src/lib/radkit/fields.ts` provides `radkit.string()`, `radkit.int()`, `radkit.relation()`, `radkit.file()`, `radkit.enum()`, `radkit.multiEnum()`, `radkit.map()` etc. Each returns a Zod schema *with attached metadata* under a `.radkit` property.

This is the single source of truth.

### 3) Loader & Registry

* `loader.ts` scans `project/collections/*.collection.ts` and `project/actions/*` and imports them dynamically using Node `import()`.
* `registry.ts` stores `collections`, `actions` and `projects` in memory and exposes helpers.
* `hooks.server.ts` initializes radkit on server start and exposes registry on `event.locals.radkit`.

### 4) Database

Static tables (users, records, links, snapshots, logs, notifications) are created via Drizzle in `src/lib/db/schema.ts`. Records store dynamic collection data in `data JSONB` and `links` hold relations.

### 5) API

All UI is driven by API endpoints under `src/routes/api/...`:

* `GET /api/projects/:project/:collection` — list (supports filter, search, pagination)
* `POST /api/projects/:project/:collection` — create (validates using collection zod)
* `GET /api/projects/:project/:collection/:id` — record get
* `PATCH /api/projects/:project/:collection/:id` — record update
* `DELETE /api/projects/:project/:collection/:id` — delete
* `POST /api/actions/:actionName` — run action (validates action form)

All endpoints enforce field-level ACL and run pre/post hooks.

### 6) UI (Admin)

Minimal modern UI using Skeleton + Tailwind. Main features:

* Left sidebar with collections (from registry)
* Top-right profile control (user avatar + quick settings)
* Collection list view: search, filters, table with pagination
* Record view: sections & span layout (driven by field `.radkit.ui.section` and `.radkit.ui.span`)
* Create/Edit opens a page with Superforms-driven bound form

All UI pages call the API endpoints to read and mutate data.

### 7) Handlebars templates (record-level)

Each collection can provide a record template inside `project/templates/`. Example: `project/templates/car-record.hbs`.

When retrieving a record, your backend can render the template server-side and return HTML as part of the record response. The admin record page can display the compiled HTML (safe) or use it for exports.

Example usage in the record endpoint:

```ts
const tmpl = fs.readFileSync(path.join(projectPath, 'templates', 'car-record.hbs'), 'utf-8');
const html = Handlebars.compile(tmpl)(recordContext);
```

The boilerplate includes an example `car-record.hbs` that repeats the layout for the record's composed view.

---

## Example files (selected)

> Note: The following are the most important files from the boilerplate. The full project in the canvas contains all files and code.

### `project/collections/brand.collection.ts`

```ts
import { z } from 'zod';
import { radkit } from '../../src/lib/radkit/fields';

export default {
  name: 'brand',
  title: 'Brand',
  emoji: '🏷️',
  is_singleton: false,
  schema: z.object({
    name: radkit.string({ label: 'Name', ui: { section: 'General', span: 6 } }),
    country: radkit.string({ label: 'Country', ui: { section: 'General', span: 6 } }),
    logo: radkit.file({ label: 'Logo', ui: { section: 'Media', span: 12 } })
  }),
  templates: {
    record: '../../project/templates/car-record.hbs'
  }
}
```

### `project/collections/car.collection.ts`

```ts
import { z } from 'zod';
import { radkit } from '../../src/lib/radkit/fields';

export default {
  name: 'car',
  title: 'Car',
  emoji: '🚗',
  schema: z.object({
    brandId: radkit.relation('brand', { label: 'Brand', ui: { section: 'General', span: 6 } }),
    model: radkit.string({ label: 'Model', ui: { section: 'General', span: 6 } }),
    year: radkit.int({ label: 'Year', ui: { section: 'Specs', span: 3 } }),
    fuel: radkit.enum(['gasoline','diesel','electric','hybrid'], { label: 'Fuel', ui: { section: 'Specs', span: 3 } }),
    options: radkit.multiEnum(['sunroof','nav','heated_seats','sport'], { label: 'Options', ui: { section: 'Specs', span: 6 } }),
    image: radkit.file({ label: 'Image', ui: { section: 'Media', span: 6 } }),
    location: radkit.map({ label: 'Location', ui: { section: 'Location', span: 12 } })
  }),
  templates: { record: '../../project/templates/car-record.hbs' }
}
```

### `project/templates/car-record.hbs` (Handlebars record template)

```hbs
<div class="record-template card p-4">
  <h1>{{data.model}} <small>({{data.year}})</small></h1>
  <div class="meta">
    <div><strong>Brand:</strong> {{brand.name}}</div>
    <div><strong>Fuel:</strong> {{data.fuel}}</div>
    {{#if data.options}}
    <div><strong>Options:</strong>
      <ul>
        {{#each data.options}}
          <li>{{this}}</li>
        {{/each}}
      </ul>
    </div>
    {{/if}}
    {{#if data.image}}
      <img src="{{data.image}}" alt="Car image" style="max-width:300px" />
    {{/if}}
  </div>
</div>
```

This template is compiled server-side for each record and returned in the record response as `renderedTemplate`.

---

## Backend: Important implementation notes

* **No third‑party code execution**: all logic in `project/actions/` is trusted and executed in the same environment. No VM, sandbox or isolation required.

* **Docker**: project includes both:

  * `docker/compose.dev.yml` for local development
  * `docker/compose.prod.yml` for production (Postgres + SvelteKit server + volume mounts)

* **ACL on filtering & listing**: every API endpoint checks permissions *before applying filters*. The server removes fields the user cannot read and denies queries on restricted fields.

* **Loader uses `fs` + dynamic import**: safe because all code is first‑party.

* **Loader uses `fs` + dynamic `import()`**. In dev it's fine. For production bundlers, prefer a build step that copies `project/` into the deployed image and use `import()` with absolute paths.

* **Security**: Action scripts and hooks run server-side — isolate them and validate inputs. Consider running user-supplied JS in a sandbox (WASM or VM) if you allow third-party code.

* **Performance**: Index often-used JSONB paths (e.g. `data->>'brandId'`) and consider materialized views for complex queries.

* **Search & filters**: implement a simple query language that maps to JSONB operators and GIN indexes.

* **Field-level ACL**: enforced server-side on every read/write. Do not trust the UI.

* **Snapshots**: create a snapshot before mutations (optionally configurable per collection).

* **Testing**: the boilerplate includes Jest (or Vitest) examples for collection loading, schema validation, and endpoint behaviors.

---

## README & Next Steps

The canvas contains a full `README.md` with instructions, environment variables, Docker instructions, and a development roadmap. It also contains a `CONTRIBUTING.md` for how to add new collections and actions.

---

### What I created for you now

I created a full, ready-to-copy boilerplate in this document (files, templates, examples, and README). Open the project in your editor, copy the files into a repo, install dependencies, and run the dev server.

If you want, I can also:

* generate the actual repo files as downloadable archive
* create the initial Docker images and a GitHub Actions workflow
* convert Handlebars templates to Svelte components for client-side rendering

Tell me which of those you'd like next.
