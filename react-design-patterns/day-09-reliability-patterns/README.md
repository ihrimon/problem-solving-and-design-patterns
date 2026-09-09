# Day 9: Reliability Patterns

Patterns covered today:

1. [Error Boundary Pattern](#1-error-boundary-pattern)
2. [Retry Pattern](#2-retry-pattern)
3. [Fallback UI Pattern](#3-fallback-ui-pattern)
4. [Portal Pattern](#4-portal-pattern)

Same running scenario as Days 3–8: an e-commerce admin dashboard. So far every pattern assumed things go right — data arrives, renders succeed, layouts nest cleanly. Today's focus is the opposite: what happens when a component throws, a network request fails, or a piece of UI needs to visually escape its parent's layout entirely.

Study method for each pattern: understand the problem, see the bad approach, see the pattern applied, real-world example, practice exercise, know when (not) to use it.

---

## 1. Error Boundary Pattern

### The problem

A bug in one widget — say, a malformed `order.total` causing `order.total.toFixed(2)` to throw inside a single `OrderSummaryCard` — crashes the *entire* React app. React unmounts the whole component tree on an uncaught render error, so the header, sidebar, and every unrelated widget on the dashboard disappear along with the one broken card, leaving a blank white screen for a bug that only affected a tiny part of the page.

### Bad approach (no error containment)

```tsx
function Dashboard() {
  return (
    <div>
      <Sidebar />
      <Header />
      <OrderSummaryCard order={order} /> {/* throws during render */}
      <RevenueChart />
      <RecentActivity />
    </div>
  );
}
```

One throw inside `OrderSummaryCard` takes down `Sidebar`, `Header`, `RevenueChart`, and `RecentActivity` too, even though none of them did anything wrong.

### Pattern applied

Wrap risky, independent sections of the UI in an Error Boundary — a component that catches render errors in its child tree and renders a fallback instead of letting the crash propagate upward.

```tsx
class ErrorBoundary extends React.Component<
  { fallback: React.ReactNode; children: React.ReactNode },
  { hasError: boolean }
> {
  state = { hasError: false };

  static getDerivedStateFromError() {
    return { hasError: true };
  }

  componentDidCatch(error: Error, info: React.ErrorInfo) {
    logErrorToService(error, info); // report it, don't just swallow it
  }

  render() {
    if (this.state.hasError) return this.props.fallback;
    return this.props.children;
  }
}

function Dashboard() {
  return (
    <div>
      <Sidebar />
      <Header />
      <ErrorBoundary fallback={<CardErrorFallback />}>
        <OrderSummaryCard order={order} />
      </ErrorBoundary>
      <RevenueChart />
      <RecentActivity />
    </div>
  );
}
```

Now a crash inside `OrderSummaryCard` is caught right there — the card shows an error message, and `Sidebar`, `Header`, `RevenueChart`, and `RecentActivity` keep working completely unaffected.

### Real-world example

Every production React app built with Next.js ships with this built in — an `error.tsx` file in the App Router is exactly an Error Boundary Vercel wires up for you per route segment. Sentry's React SDK also provides a ready-made `Sentry.ErrorBoundary` that combines this exact pattern with automatic error reporting, which is why `componentDidCatch` logging to a service (not just swallowing the error) is part of the pattern, not an afterthought.

### Practice exercise

Wrap the dashboard's `RevenueChart` (a component prone to throwing on malformed chart data) in its own `ErrorBoundary` with a fallback that shows "Chart unavailable" plus a "Retry" button that resets `hasError` back to `false`.

### When to use

- Around independent, self-contained sections of a page (a widget, a route, a third-party chart) where one section's failure shouldn't be allowed to take the rest of the page down with it.

### When NOT to use

- Wrapping *every single* component individually. Error Boundaries add a layer of ceremony and hide genuine bugs from visibility during development if overused — reach for one per meaningfully independent section (a widget, a route), not per `<Button>`.

---

## 2. Retry Pattern

### The problem

A `productService.list()` call fails because of a momentary network blip — a dropped Wi-Fi packet, a brief server hiccup — even though the exact same request would succeed if tried again half a second later. Showing the user a permanent error message (or an empty product list) for a failure that would have resolved itself on its own is a worse experience than quietly trying again.

### Pattern applied

Automatically re-attempt a failed request a bounded number of times, with increasing delay between attempts (exponential backoff), before finally giving up and showing an error.

```tsx
async function fetchWithRetry<T>(
  fn: () => Promise<T>,
  maxAttempts = 3,
): Promise<T> {
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (attempt === maxAttempts) throw error; // out of attempts, surface the real error
      const delay = 2 ** attempt * 100; // 200ms, 400ms, 800ms
      await new Promise((resolve) => setTimeout(resolve, delay));
    }
  }
  throw new Error('unreachable');
}

// Usage
const products = await fetchWithRetry(() => productService.list());
```

The delay doubles with each attempt (exponential backoff) instead of retrying instantly and repeatedly — retrying immediately and rapidly against a server that's already struggling (e.g., briefly overloaded) makes the problem worse, not better.

### Real-world example

React Query and SWR both build retry with exponential backoff in by default for every query — this is precisely why: momentary network blips are common enough in real-world mobile/Wi-Fi conditions that silently retrying before bothering the user is the right default behavior, not an edge case. AWS SDKs and most production HTTP clients implement the same exponential-backoff retry logic for the same reason at the infrastructure level.

### Practice exercise

Add a `shouldRetry` parameter to `fetchWithRetry` that skips retrying entirely for 4xx errors (a genuinely bad request, like invalid input, won't succeed no matter how many times it's retried) but does retry for 5xx errors and network failures.

### When to use

- Network requests where a failure is plausibly transient (server hiccup, dropped connection) rather than a certain, repeatable failure.

### When NOT to use

- Errors that are guaranteed to fail identically every time (invalid input causing a 400, an authentication failure from a bad token) — retrying those wastes time and delays showing the user the real, actionable error message. This is exactly why the practice exercise's `shouldRetry` check matters in production code.

---

## 3. Fallback UI Pattern

### The problem

Even with Error Boundaries (Pattern 1) catching crashes and Retry (Pattern 2) handling transient failures, some failures are final — the retries are exhausted, or the error is a real one (the product genuinely doesn't exist, a 404). Showing the user a blank space, a raw error message like "TypeError: Cannot read properties of undefined," or — worse — nothing at all, leaves them with no idea what happened or what to do next.

### Pattern applied

Design and render a deliberate, informative UI for every failure state a component can reach — not just a generic "Something went wrong," but one tailored to what actually happened and what the user can do about it.

```tsx
function ProductDetail({ productId }: { productId: string }) {
  const { data: product, error, isLoading, refetch } = useQuery({
    queryKey: ['product', productId],
    queryFn: () => productService.getById(productId),
  });

  if (isLoading) return <ProductDetailSkeleton />;

  if (error) {
    if (error.status === 404) {
      return <EmptyState message="This product no longer exists." action={<Link to="/products">Back to Products</Link>} />;
    }
    return <ErrorState message="Couldn't load this product." action={<Button onClick={() => refetch()}>Try again</Button>} />;
  }

  return <ProductView product={product} />;
}
```

Three distinct states get three distinct, purposeful fallbacks: a skeleton while loading (not a spinner — Pattern 8's territory, but worth noting here), a specific "this doesn't exist, here's where to go instead" for a 404, and a generic "try again" with an actual retry button for anything else.

### Real-world example

Stripe's Dashboard, GitHub, and virtually every polished SaaS product distinguish a "not found" empty state from a "something broke, here's a retry button" error state — never a single generic error screen for both, because the correct next action for the user is completely different in each case (navigate away versus try again).

### Practice exercise

Add a third fallback case to `ProductDetail` for when `productService.getById` succeeds but returns a product that's out of stock — a distinct `<OutOfStockState product={product} />` rather than treating it as either a loading, error, or fully-available state.

### When to use

- Every component with more than one way to fail or be empty — which, once Server State (Day 7) is in the picture, is nearly every data-driven component in the app.

### When NOT to use

- A component with a single, simple state and no realistic failure mode (a static "About" page). Building out loading/error/empty variants for something that can't meaningfully fail is speculative work with no payoff.

---

## 4. Portal Pattern

### The problem

A `Modal` rendered from deep inside `OrderTable` → `OrderTableRow` → `OrderActionsMenu` inherits every CSS property from its ancestors — `overflow: hidden` on a scrollable table container clips the modal at the table's edge, and a `z-index` stacking context set up by a parent can bury the modal behind other elements, no matter how high its own `z-index` is set. The modal needs to visually escape the DOM position it's logically rendered from.

### Pattern applied

Render the modal's actual DOM nodes into a completely different part of the document — typically a dedicated element directly under `<body>` — while keeping it logically part of the same React component tree (same props, same state, same event bubbling through React's synthetic event system).

```tsx
import { createPortal } from 'react-dom';

function Modal({ children, onClose }: { children: React.ReactNode; onClose: () => void }) {
  return createPortal(
    <div className="modal-overlay" onClick={onClose}>
      <div className="modal-content" onClick={(e) => e.stopPropagation()}>
        {children}
      </div>
    </div>,
    document.getElementById('modal-root')!, // a sibling of #root, directly under <body>
  );
}

// Still used exactly like a normal component, from deep inside the table
function OrderActionsMenu({ order }: { order: Order }) {
  const [isOpen, setIsOpen] = useState(false);
  return (
    <>
      <Button onClick={() => setIsOpen(true)}>Actions</Button>
      {isOpen && (
        <Modal onClose={() => setIsOpen(false)}>
          <OrderActionsList order={order} />
        </Modal>
      )}
    </>
  );
}
```

`OrderActionsMenu` renders `<Modal>` exactly like any other child component — no special API, no prop drilling to some top-level modal manager. But the actual DOM nodes land in `#modal-root`, completely outside the table's `overflow`/`z-index` constraints, while React events (like the `onClick` that closes it) still bubble up through `OrderActionsMenu` normally, because React's event system follows the component tree, not the DOM tree.

### Real-world example

Radix UI, Headless UI, and shadcn/ui's `Dialog`, `Popover`, `Tooltip`, and `DropdownMenu` components all use a Portal internally for exactly this reason — a tooltip or dropdown triggered from deep inside a scrollable card must never be visually clipped by that card's `overflow: hidden`, and a Portal is the only mechanism that guarantees it.

### Practice exercise

Build a `Toast` notification component using `createPortal` that renders into a `#toast-root` element, so toast notifications triggered from anywhere in the app (including from inside a modal, itself already a portal) always render on top of everything else, unclipped.

### When to use

- Any UI that must visually escape a scrollable or `overflow: hidden` container, or must guarantee top-most stacking regardless of where it's triggered from — modals, tooltips, dropdowns, toasts, popovers.

### When NOT to use

- Regular in-flow content that has no layout/stacking conflict with its parent. Reaching for a Portal by default for every component "just in case" adds indirection (the DOM tree and component tree diverge, which can confuse debugging) with no benefit for content that was never going to be clipped anyway.

---

## How the four patterns fit together

Error Boundary (Pattern 1) and Retry (Pattern 2) both work to prevent a failure from becoming visible at all — one contains crashes after they happen, the other avoids surfacing transient ones in the first place. Fallback UI (Pattern 3) is what happens once a failure genuinely can't be avoided or retried away — turning "the app is broken" into a clear, actionable message, and it's typically what an Error Boundary's `fallback` prop or a failed query's `error` state actually renders. Portal (Pattern 4) is an unrelated, purely visual concern — but it shows up constantly alongside the other three, since the modals and toasts used to surface retry buttons and error states are themselves almost always built on a Portal to escape their triggering component's layout constraints.
