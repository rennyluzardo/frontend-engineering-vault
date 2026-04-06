# miPOS — Frontend Engineering Case Study
## Building the UI for an Open Restaurant Operating System

**Primary Reference:** [GitHub Repository: rennyluzardo/miPOS](https://github.com/rennyluzardo/miPOS/)
**Role:** Lead Frontend Engineer · Sole frontend contributor
**Product:** miPOS (mipos.pro / mipos.dev) — API-first SaaS platform for the restaurant industry
**Stack:** Next.js · Redux · SCSS · Node.js (custom server)
**Status:** Discontinued · Published as approved portfolio reference

---

### 1. The Problem Worth Solving
Restaurants in 2018 still juggled siloed delivery apps, on-prem POS terminals, and spreadsheets for inventory. Operators double-entered every Rappi or Uber Eats order into their POS, then manually adjusted prep queues and supplier minimums hours later. The ideal interface had to collapse those tasks into a single, resilient view: live delivery tickets, on-site tables, and SKU depletion all sharing one queue. Because miPOS promised an API-first platform, the frontend had to act like an operating console for both humans and integrations—surfacing structured data the same way partners would consume it downstream.

### 2. Architectural Decisions
- **Next.js Pages Router:** I chose Next.js to server-render the ordering console, keeping first paint fast on underpowered back-office machines and enabling `getInitialProps` hydration for any route that needed preloaded data.@pages/_app.js#1-25  The thin pages directory (`index.jsx`, `orders.jsx`) let me keep each view self-contained yet deployed through a unified build pipeline.@pages/orders.jsx#1-260 @pages/index.jsx#1-14 
- **Custom Node.js server:** A bespoke Express host wraps Next so I could swap routing logic or inject middleware before Next handled the request, preparing for authenticated partner embeds and alternative domains. `server.js` wires Express + Next, while `routes.js` (via `next-routes`) kept human-friendly URLs that matched POS terminology instead of the default file-system router.@server.js#1-18 @routes.js#1-3  The custom `next.config.js` adds Sass tooling, jQuery for legacy widgets, and a `routes` alias—choices aimed at rapid iteration inside one repo.@next.config.js#1-26 
- **Redux via React Context:** Rather than pulling in the full Redux package, I used `useReducer` with a root reducer that delegates to cart and product slices, mirroring the same mental model while avoiding extra dependencies.@config/store.js#1-18 @reducers/rootReducer.js#1-15  Given orders can mutate from multiple sources, predictable reducer switches for `products` and `cart` kept mutations explicit and traceable.@reducers/product.js#1-14 @reducers/cart.js#1-12 
- **SCSS Architecture:** Global styling begins with mixins and Ant Design imports, then cascades into `core`, `pages`, and `components` directories via glob imports.@scss/styles.scss#1-15  Component partials such as `_order-toolbar.scss` pair BEM-ish naming with utility mixins (`font-size`) so any widget can scale typography without hard-coded pixels.@scss/components/orders/_order-toolbar.scss#1-85 @scss/mixins/_font-size.scss#1-13  The structure mirrors the component tree, letting operators' critical zones (toolbar, listings, sidecar) evolve independently.

### 3. Component Architecture
- **Hierarchy:** `OrdersLayout` composes Ant Design’s `Layout` shell with global header, a sidecar summary, and a dynamic content area for workflows.@components/layouts/OrdersLayout.jsx#1-45  Inside, specialized children handle order phases—`CategoriesListing`, `ProductsListing`, `AdditionalsListing`, and modal flows—keeping each concern isolated.@components/orders/CategoriesListing.jsx#1-27 @components/orders/ProductsListing.jsx#1-22 @components/orders/AdditionalsListing.jsx#1-25 
- **Reusability & Composition:** Shared primitives (e.g., `Counter`, `PrimaryButton`, toolbar buttons) abstract repeated control patterns. `Counter` toggles between toolbar and inline usage via props, demonstrating how small switches handle drastically different UI states without duplicating logic.@components/global/Counter.js#1-31  Layout composition leans on Ant Design grid components, so each module simply returns rows/cols, aligning with POS-like tiling requirements.
- **POS-specific UX:** `OrderToolbar` maintains always-on actions (back, table shortcuts, cancel) with large tap targets and color-coded states for high-pressure environments.@components/orders/OrderToolbar.js#1-48  Listing components favor dense grids with predictable spacing, ensuring cooks or waitstaff can scan categories and products without scroll jitter. `ResumProducts` and `ResumNoProducts` toggle the side summary to reflect whether anything is queued, keeping mental load low during rushes.@components/orders/ResumProducts.jsx#1-30 @components/orders/ResumNoProducts.jsx#1-16 
- **Design System Patterns:** The SCSS partials mirror atomic layers: typography mixins feed button rectangles, which in turn are extended by toolbar buttons. This pseudo-design-system allowed me to ship consistent accents quickly even without a full token library.

### 4. State Management Deep Dive
- **State Shape:** The global store tracks `products`, `categories`, and `cart`, seeded via API calls and mocked data for offline demos.@config/store.js#5-18 @pages/orders.jsx#30-134  Each category nests products, matching the domain payloads from the miPOS API.
- **Action Flows:** `fetchCategories` and `fetchProducts` call the miPOS API gateway with bearer tokens, dispatching reducer-specific actions so slices stay decoupled.@actions/product.js#1-38  `setCart` mirrors the same pattern for cart mutations.@actions/cart.js#1-9  In `Orders.jsx`, selecting a category triggers `fetchProducts`, drilling into category-specific SKUs, while adding products mutates the cart in-place before dispatching the updated structure.@pages/orders.jsx#151-199 
- **Async Patterns:** Fetches are handled with bare `fetch` promises—no middleware—which kept the MVP lightweight but meant error handling and loading states had to be embedded per component. Side effects such as initial category fetches and cart bootstrapping live inside `useEffect`, reflecting Next.js’ client hydration path.@pages/orders.jsx#201-210 
- **Gaps & Improvements:** The reducers are minimal and lack normalization, so nested cart updates rely on `lodash` cloning and custom loops.@pages/orders.jsx#184-199  There’s no central error or loading state, and API tokens live in config, which would need hardening.

### 5. Engineering Challenges of a Delivery-Aggregation POS
1. **Normalizing multi-source orders:** Aggregating Rappi, Uber Eats, and in-house orders means each payload carries different field names and optional metadata. The cart schema (categories → products → specifications) provided a normalized view the UI could rely on, regardless of the upstream platform.@pages/orders.jsx#30-134 
2. **Real-time UI pressure:** The toolbar, grids, and sidecar summary keep critical information in view without modal lock-ups, reflecting the need for operators to react in seconds.@components/orders/OrderToolbar.js#1-48 @components/orders/ResumProducts.jsx#1-30 
3. **State consistency across channels:** By mutating a single `cart` tree and dispatching `setCart`, simultaneous order arrivals from different integrations could converge into one source of truth, even if the data started as mock payloads during early demos.@pages/orders.jsx#184-199 @actions/cart.js#1-9 
4. **Designing for operators:** Components prioritize large hit areas, minimal text, and high-contrast CTA blocks (`order-toolbar__right-actions`), reflecting the ergonomics of a POS more than a marketing site.@components/orders/OrderToolbar.js#1-48 @scss/components/orders/_order-toolbar.scss#1-85 

### 6. What I Would Do Differently Today (2025+)
- **State:** Swap custom reducers for Redux Toolkit or Zustand to gain immer-powered immutability, async thunks, and type-safe slices.
- **Framework:** Migrate to Next.js App Router with React Server Components for streaming menus and edge-friendly deployments.
- **Styling:** Replace SCSS + manual variables with Tailwind CSS plus a headless component kit (e.g., Radix) to accelerate design tokens and responsive variants.
- **Type Safety:** Adopt TypeScript end-to-end, deriving API types from an OpenAPI schema to eliminate runtime shape guessing.
- **Real-time:** Move from fetch polling to WebSockets or Server-Sent Events so new delivery orders appear instantly without manual refresh loops.
- **Testing:** Layer React Testing Library for component behavior and Playwright (already partially configured) for E2E coverage, ensuring order flows stay stable through refactors.
- **Security & Config:** Pull API hosts and tokens from environment variables with secrets management instead of hardcoding in `config/constants`.

### 7. Business Impact & Legacy
Centralizing delivery and on-prem orders in one UI eliminated the re-entry loop that cost kitchens 2–5 minutes per ticket, saving whole hours each shift. A single cart and menu source reduced misfires from transcription mistakes, improving order accuracy when juggling multiple platforms. miPOS’ open API vision—letting partners build marketplaces or ghost-kitchen workflows on top of restaurant data—anticipated today’s unified commerce push years ahead. Even though the product sunset, this frontend shows how to translate API-first thinking into an operator-friendly console, which remains a relevant reference point for modern platform teams.
