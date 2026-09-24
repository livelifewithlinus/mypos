# OpenPOS

OpenPOS is an open-source foundation for a modern point-of-sale system. It is intended to support dependable day-to-day commerce while remaining local-first, maintainable, and extensible.

## Current status

### Phase 1 - UI Shell

The main OpenPOS application shell is now in place. It includes responsive sidebar navigation, a top header, a dashboard with realistic Kenyan Shilling demo data, recent sales, low stock alerts, and clean placeholder views for the remaining sections.

This phase is UI-only. Demo data is centralized in `src/data/demoData.ts` so it can later be replaced by database or API data. No POS business logic, database, authentication, barcode, payment, or inventory logic has been implemented.

### Phase 2 - Database Layer

The local data foundation is now in place without changing the Phase 1 interface. OpenPOS uses SQLite through Drizzle ORM and a small Node API layer. The React frontend remains separate from the database and does not import database modules directly.

Architecture:

```text
React UI -> Node API -> Drizzle ORM -> SQLite
```

The schema currently contains `categories`, `products`, `sales`, `sale_items`, `inventory_movements`, and `users`. Relationships use foreign keys, and SKU, receipt number, category name, and optional barcode/email fields use unique constraints. Monetary values are stored as integer minor units (for example, `6500` represents KSh 65.00) to avoid floating-point rounding errors. Typed unions document allowed payment methods, sale statuses, inventory movement types, and user roles.

Development SQLite data is stored at `.data/openpos.db`. The `.data/` directory and SQLite sidecar files are ignored by Git because local development data should never be committed.

### Phase 3 - Product Management

Products and categories are now managed through the React interface and Node API. Products support names, unique SKUs, optional unique barcodes, descriptions, category assignment, integer minor-unit selling/cost prices, initial stock, low-stock thresholds, image URLs, and active/inactive status. Product deactivation is a safe soft-delete so historical sales can reference the product later.

The Products page includes server-side search by name/SKU/barcode, category/status/stock filters, pagination (20 per page), reusable add/edit forms, category management, validation feedback, confirmation prompts, empty/loading/error states, and success notifications. Stock is set only at creation; ongoing inventory adjustments remain a future module.

API endpoints:

- `GET`, `POST` `/api/products`
- `GET`, `PUT`, `DELETE` `/api/products/:id`
- `GET`, `POST` `/api/categories`
- `PUT`, `DELETE` `/api/categories/:id`
- `GET` `/api/dashboard` (product and low-stock totals)

Run the API and UI in separate terminals during development:

```bash
npm run api
npm run dev
```

There is not yet a project test runner. Type checking is included in `npm run build`, and server type checking can be run with `npm run typecheck:server`.

### Phase 4 - Barcode Generator

The Barcode Generator creates product labels using [JsBarcode](https://github.com/lindell/JsBarcode) in the browser. Code 128 is the default and is recommended for flexible internal product identifiers such as SKUs. EAN-13 and UPC-A are supported only when a numeric value has the required length and a valid check digit; OpenPOS does not generate commercial identifiers automatically.

Choose a product, format, barcode value, and quantity (up to 100 labels), then preview, save, download PNG/SVG, or print a label. Saved barcode values use the existing product `barcode` field and are validated server-side for uniqueness and format. The Products page includes a Generate Barcode/View Barcode action, and the Barcode Generator also supports print-ready batch labels for products with assigned barcodes. Barcode scanning is intentionally not included.

The barcode-specific API is `PUT /api/products/:id/barcode`, accepting `{ barcode, format }`. It validates product existence, supported format, EAN-13/UPC-A checksums, and duplicate barcode values before saving.

### Phase 5 - Barcode Scanner

OpenPOS supports hardware, camera, and manual barcode lookup through a shared scanner flow. `GET /api/products/barcode/:barcode` returns safe product data only for active products. The Barcode Scanner page can be opened directly from the sidebar or Products page.

Hardware scanners are supported as keyboard-emulating USB/Bluetooth HID devices: rapid barcode characters followed by Enter are collected by the reusable `useBarcodeScanner` hook. It ignores focused text fields, uses an 80 ms configurable inter-key threshold, clears stale buffers, and requires a short, rapid scan before calling the lookup API. Scanner models and configuration vary; configure a device for HID/keyboard output with an Enter suffix where available. No special OpenPOS driver is required for compatible devices.

Camera scanning uses `@zxing/browser` and decodes frames locally; no images or frames are uploaded or stored. Camera permission is requested only after choosing Camera mode and pressing Start Camera. Use a modern browser on HTTPS or localhost, allow camera access, and stop any other app already using the camera if startup fails. Camera streams are stopped when scanning succeeds, when modes change, and when leaving the scanner page.

### Phase 6 - POS Checkout

The POS page supports product search, keyboard-emulating scanner input, cart quantity controls, fixed or percentage cart discounts, a structured 0% default tax rate, cash/card/other payment recording, cash change, and a simple completion receipt. F2 focuses product search and F4 readies hardware scanning. Card and Other only record the payment method; no payment gateway, terminal, M-Pesa, refund, or return integration is included.

`POST /api/sales` is authoritative: it reloads active products, rechecks stock, snapshots current product prices, recalculates subtotal/discount/tax/total using integer minor units, validates cash received, creates a human-readable receipt number, sale and sale items, reduces stock, and writes `sale` inventory movements in one SQLite transaction. The browser does not send prices or totals as trusted values. SQLite provides transactional consistency for this local application; deployments needing high concurrent checkout throughput should use a database with stronger multi-writer characteristics.

## Technology stack

- React
- TypeScript
- Vite
- Tailwind CSS
- Lucide React icons
- SQLite
- Drizzle ORM and Drizzle Kit
- Node.js API server
- Git

## Development setup

Install a current Node.js LTS release, then install dependencies from the project root:

```bash
npm install
```

Start the development server:

```bash
npm run dev
```

Build the project for production:

```bash
npm run build
```

Preview a production build:

```bash
npm run preview
```

## Database commands

Generate a migration after changing the schema:

```bash
npm run db:generate
```

Apply migrations to the local database:

```bash
npm run db:migrate
```

Seed development categories and products. The seed is idempotent for the provided names and SKUs:

```bash
npm run db:seed
```

Start the API server:

```bash
npm run api
```

The health check is available at `http://localhost:3001/api/health`. To inspect the database, use a SQLite-compatible client against `.data/openpos.db`, or run the migration and seed commands above before querying tables such as `categories` and `products`.

## Planned future modules

- Product and SKU management
- Barcode generation and scanning
- Inventory management
- POS checkout and payments
- Receipts and sales history
- Reports
- Users and permissions
- Offline and local-first operation

## Next phase

Phase 4 will introduce barcode generation and label printing. Barcode scanning and POS checkout are intentionally not included in this phase.

## License

This project is open source. Licensing details will be added before the first public release.
