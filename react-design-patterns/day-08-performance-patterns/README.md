# Day 8: Performance Patterns

Patterns covered today:

1. [Lazy Loading Pattern](#1-lazy-loading-pattern)
2. [Code-Splitting Pattern](#2-code-splitting-pattern)
3. [Suspense Pattern](#3-suspense-pattern)
4. [Virtualization Pattern](#4-virtualization-pattern)

Same running scenario as Days 3–7: an e-commerce admin dashboard. By now the dashboard has grown — an Orders page, a Products page, a Users page, an analytics chart library, a bulk CSV export tool. Today's focus is what happens when all of that gets bundled and shipped to the browser at once: a slow initial load, and a table with thousands of rows that makes the browser choke.

Study method for each pattern: understand the problem, see the bad approach, see the pattern applied, real-world example, practice exercise, know when (not) to use it.

---

## 1. Lazy Loading Pattern

### The problem

A `BulkExportModal` component pulls in a heavy CSV-generation library and a complex column-configuration UI — code that only ever runs if an admin clicks "Export" on the Orders page, which most sessions never do. If it's imported normally at the top of the file, its entire JavaScript weight is downloaded and parsed on *every* visit to the dashboard, even for the admin who never touches the export feature that day.

### Bad approach (everything imported eagerly)

```tsx
import { BulkExportModal } from './BulkExportModal'; // pulls in a heavy CSV library, always

function OrderTable() {
  const [showExport, setShowExport] = useState(false);
  return (
    <>
      <Button onClick={() => setShowExport(true)}>Export</Button>
      {showExport && <BulkExportModal onClose={() => setShowExport(false)} />}
    </>
  );
}
```

Even though `BulkExportModal` only renders when `showExport` is `true`, its *code* is bundled and downloaded unconditionally, because a static `import` at the top of the file is resolved at build time regardless of runtime conditions.

### Pattern applied

Defer loading a component's code until the moment it's actually needed, using a dynamic `import()` wrapped in `React.lazy`.

```tsx
import { lazy, Suspense } from 'react';

const BulkExportModal = lazy(() => import('./BulkExportModal'));

function OrderTable() {
  const [showExport, setShowExport] = useState(false);
  return (
    <>
      <Button onClick={() => setShowExport(true)}>Export</Button>
      {showExport && (
        <Suspense fallback={<Spinner />}>
          <BulkExportModal onClose={() => setShowExport(false)} />
        </Suspense>
      )}
    </>
  );
}
```

Now `BulkExportModal`'s code is only fetched from the network the first time an admin actually clicks "Export" — every other session that never opens the modal never downloads its JavaScript at all.

### Real-world example

Google Docs lazy-loads its print-preview and diagram-editing modules only when a user actually opens those features, rather than shipping every possible editing tool on initial page load — this is exactly why the print dialog has a brief loading moment the first time you open it in a session. Any Next.js app using `next/dynamic` for a heavy chart library or rich-text editor is applying this same pattern.

### Practice exercise

Convert an eagerly-imported `ProductBulkEditPanel` (used only when an admin selects multiple products and clicks "Bulk Edit") into a `lazy()` import, wrapped in `Suspense` with a fallback that matches the panel's expected layout size (to avoid a layout jump when it finishes loading).

### When to use

- Any component that's conditionally rendered and not needed on initial page load — modals, rarely-used settings panels, heavy third-party widgets (a rich-text editor, a chart library) gated behind a click or a permission check.

### When NOT to use

- Content needed immediately on first render (the page's main layout, above-the-fold content). Lazy-loading something the user sees instantly just adds a network round-trip and a loading flicker for no benefit — it only pays off for code that's genuinely deferred in when it's needed.

---

## 2. Code-Splitting Pattern

### The problem

Without any splitting, a bundler produces one giant `bundle.js` containing the Orders page, the Products page, the Users page, and the analytics dashboard — all of it. An admin who only ever visits the Orders page still downloads and parses the code for every other page before seeing anything, because it's all one file.

### Pattern applied

Split the bundle along natural boundaries — most commonly by route — so a visitor only downloads the code the specific page they're on actually needs, fetching the rest on demand as they navigate.

```tsx
import { lazy, Suspense } from 'react';
import { Routes, Route } from 'react-router-dom';

const OrdersPage = lazy(() => import('./pages/OrdersPage'));
const ProductsPage = lazy(() => import('./pages/ProductsPage'));
const UsersPage = lazy(() => import('./pages/UsersPage'));

function App() {
  return (
    <Suspense fallback={<PageSkeleton />}>
      <Routes>
        <Route path="/orders" element={<OrdersPage />} />
        <Route path="/products" element={<ProductsPage />} />
        <Route path="/users" element={<UsersPage />} />
      </Routes>
    </Suspense>
  );
}
```

Visiting `/orders` now only fetches `OrdersPage`'s chunk (plus whatever it imports) — `ProductsPage` and `UsersPage`'s code is fetched later, only if and when that admin actually navigates there.

### Real-world example

Every modern bundler (Vite, Webpack, Next.js's built-in per-page bundling) implements route-based code-splitting essentially as a default expectation for production apps today — Next.js's Pages and App Router both automatically create a separate JavaScript chunk per route without any manual configuration, precisely because shipping one monolithic bundle for a multi-page app is considered a performance anti-pattern industry-wide.

### Practice exercise

Take a single-bundle dashboard with `OrdersPage`, `ProductsPage`, `UsersPage`, and `SettingsPage` all statically imported in one router file, and convert all four to route-based code-splitting using `lazy()`, wrapping the `<Routes>` block in one shared `Suspense` boundary.

### When to use

- Any multi-page/multi-route app of meaningful size — splitting by route is close to a default best practice for production React apps, not a specialized optimization reserved for edge cases.

### When NOT to use

- A genuinely small, single-page app where the entire bundle is already small (a simple landing page, a tiny internal tool with one screen). Splitting a 40KB bundle into several tiny chunks can add more network round-trip overhead than it saves.

---

## 3. Suspense Pattern

### The problem

Lazy Loading (Pattern 1) and Code-Splitting (Pattern 2) both introduce a real gap: the moment between "the user needs this component" and "its code has finished downloading." Without a coordinated way to show a loading state during that gap, every lazy component would need its own hand-rolled `isLoading` flag, and nesting several lazy-loaded pieces means manually coordinating several independent loading flags at once.

### Pattern applied

Wrap a subtree containing one or more async dependencies (lazy-loaded components, and in React 18+, data fetched via Suspense-compatible libraries) in a `<Suspense>` boundary, which declaratively shows a `fallback` until everything inside has finished loading — without manual `isLoading` state.

```tsx
import { lazy, Suspense } from 'react';

const RevenueChart = lazy(() => import('./RevenueChart'));
const RecentActivity = lazy(() => import('./RecentActivity'));

function DashboardHome() {
  return (
    <Suspense fallback={<DashboardSkeleton />}>
      <RevenueChart />
      <RecentActivity />
    </Suspense>
  );
}
```

One `<Suspense>` boundary coordinates both `RevenueChart` and `RecentActivity` — the fallback shows until *both* have loaded, with no manual tracking of two separate loading booleans. Nesting boundaries lets different parts of a page reveal independently:

```tsx
<Suspense fallback={<PageSkeleton />}>
  <Header />
  <Suspense fallback={<ChartSkeleton />}>
    <RevenueChart /> {/* slow to load */}
  </Suspense>
  <RecentActivity /> {/* fast, doesn't wait for the chart */}
</Suspense>
```

Here, `Header` and `RecentActivity` can appear as soon as they're ready, while only `RevenueChart`'s own nested boundary waits specifically for the chart, instead of one slow component blocking the whole page behind a single fallback.

### Real-world example

Next.js's App Router builds routing directly on Suspense — a route segment's `loading.tsx` file is, under the hood, exactly a `<Suspense fallback={...}>` boundary Next.js wires up automatically per route. React Query's `useSuspenseQuery` (and SWR's Suspense mode) extend the same mechanism to data fetching, letting a component suspend while its data loads instead of manually checking an `isLoading` flag, using the same `<Suspense>` boundary already in place for lazy-loaded code.

### Practice exercise

Take the nested `DashboardHome` example above and add a third lazy-loaded `OrderSummaryWidget` inside its own nested `Suspense` boundary, so a slow `RevenueChart` and a slow `OrderSummaryWidget` each show their own independent skeleton without blocking `Header` or `RecentActivity` from appearing immediately.

### When to use

- Anywhere Lazy Loading or Code-Splitting is already in use (a `Suspense` boundary is required for `React.lazy` to work at all), and increasingly for Suspense-compatible data fetching once a project has adopted React 18+ and a library that supports it.

### When NOT to use

- Wrapping synchronous, already-loaded content in a `Suspense` boundary accomplishes nothing — there's nothing to suspend on. It's only meaningful around genuinely async boundaries: lazy-loaded code or Suspense-integrated data fetching.

---

## 4. Virtualization Pattern

### The problem

An Orders table showing 10,000 rows — even with Pagination (Day 7) turned off, or in a scenario like a spreadsheet-style bulk-edit view where all rows must be scrollable in one continuous list — means React creates 10,000 real `<tr>` DOM nodes. The browser has to lay out, paint, and keep in memory all 10,000 of them, even though at any given moment the user's screen can only physically display 20–30 rows. Scrolling becomes janky, and memory usage balloons for rows that are, at any instant, invisible.

### Pattern applied

Render only the rows currently visible in the viewport (plus a small buffer), recycling the same small set of DOM nodes as the user scrolls, instead of creating a DOM node per data item.

```tsx
import { useVirtualizer } from '@tanstack/react-virtual';

function VirtualizedOrderTable({ orders }: { orders: Order[] }) {
  const parentRef = useRef<HTMLDivElement>(null);

  const virtualizer = useVirtualizer({
    count: orders.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 48, // estimated row height in px
    overscan: 5, // render a few extra rows above/below the viewport
  });

  return (
    <div ref={parentRef} style={{ height: '600px', overflow: 'auto' }}>
      <div style={{ height: virtualizer.getTotalSize(), position: 'relative' }}>
        {virtualizer.getVirtualItems().map((virtualRow) => (
          <div
            key={virtualRow.key}
            style={{
              position: 'absolute',
              top: 0,
              transform: `translateY(${virtualRow.start}px)`,
              height: virtualRow.size,
              width: '100%',
            }}
          >
            <OrderTableRow order={orders[virtualRow.index]} />
          </div>
        ))}
      </div>
    </div>
  );
}
```

Even with 10,000 `orders`, `virtualizer.getVirtualItems()` only returns the ~25-30 rows currently visible (plus the 5-row overscan buffer) — the outer `div` is sized to the *full* scrollable height (`getTotalSize()`) so the scrollbar behaves correctly, but the actual DOM only ever contains a small, constant number of `OrderTableRow` elements, repositioned with `transform: translateY(...)` as the user scrolls.

### Real-world example

VS Code's file explorer and search results list, Slack's message history, and Google Sheets (rendering a grid of potentially millions of cells) all rely on virtualization — none of them could stay responsive if every row/cell in a huge dataset were a real, permanent DOM node. `react-window` and `@tanstack/react-virtual` are the two most widely used React libraries implementing this technique.

### Practice exercise

Apply the same `useVirtualizer` setup to a `ProductGrid` showing a few thousand products in a scrollable grid rather than a list, adjusting `estimateSize` for a card's height and confirming (via browser dev tools' element inspector) that only a small, constant number of `ProductCard` DOM nodes exist at any scroll position.

### When to use

- Any single, continuously-scrollable list or table with hundreds to thousands (or more) of rows/items rendered at once — a bulk-edit table, a chat message history, a large data grid.

### When NOT to use

- A paginated list (Day 7) already showing a small, bounded number of items per page (20-50 rows) has no virtualization problem to solve — the DOM node count is already small. Virtualization and Pagination solve overlapping problems from different angles; a well-paginated table usually doesn't need virtualization on top of it, and adding it anyway is unnecessary complexity (virtualized lists lose native browser find-in-page, and complicate variable-height content).

---

## How the four patterns fit together

Lazy Loading (Pattern 1) and Code-Splitting (Pattern 2) solve the same underlying problem — shipping less JavaScript upfront — at two different scopes: Lazy Loading defers one specific component (a modal, a rarely-used panel), while Code-Splitting applies the same `lazy()` mechanism systematically across an app's routes. Suspense (Pattern 3) is the coordination layer both of them are built on — it's literally required for `React.lazy` to render a fallback at all, and extends naturally to Suspense-compatible data fetching once adopted. Virtualization (Pattern 4) is a mostly unrelated concern — it's not about how much *code* ships, but how many *DOM nodes* exist for a large dataset already loaded in memory — but it commonly appears in the same performance-conscious codebases, often paired with Day 7's Infinite Scroll (which keeps *fetching* more data) sitting on top of a virtualized list (which keeps *rendering* that data cheap regardless of how large it grows).
