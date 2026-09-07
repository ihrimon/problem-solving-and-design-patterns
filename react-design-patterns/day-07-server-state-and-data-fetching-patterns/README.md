# Day 7: Server State & Data Fetching Patterns

Patterns covered today:

1. [Server State Pattern](#1-server-state-pattern)
2. [Effect Synchronization Pattern](#2-effect-synchronization-pattern)
3. [Optimistic UI Pattern](#3-optimistic-ui-pattern)
4. [Pagination Pattern](#4-pagination-pattern)
5. [Infinite Scroll Pattern](#5-infinite-scroll-pattern)

Same running scenario as Days 3–6: an e-commerce admin dashboard. Days 5–6 organized *how* the app talks to the backend and *where* that code lives. Today's focus is what happens once data actually arrives from the server: keeping it fresh, handling slow networks gracefully, and rendering large lists without melting the browser.

Study method for each pattern: understand the problem, see the bad approach, see the pattern applied, real-world example, practice exercise, know when (not) to use it.

---

## 1. Server State Pattern

### The problem

Treating data that lives on a server (orders, products, users) the same way as local UI state (`isModalOpen`, `selectedTab`) causes real bugs. Server state can change without the app doing anything (another admin cancels an order elsewhere), it can be stale the moment it's fetched, and the same data is often needed in multiple unrelated components (the order count badge in the header, and the full order table on the Orders page) — each one naively re-fetching, with no coordination.

### Bad approach (server data treated as local state)

```tsx
function OrderCountBadge() {
  const [count, setCount] = useState(0);
  useEffect(() => {
    orderService.list().then((orders) => setCount(orders.length));
  }, []);
  return <Badge>{count}</Badge>;
}

function OrderTable() {
  const [orders, setOrders] = useState<Order[]>([]);
  useEffect(() => {
    orderService.list().then(setOrders);
  }, []);
  // ...
}
```

Both components fetch the exact same `/orders` endpoint independently, with no shared cache. Cancel an order in `OrderTable` and `OrderCountBadge` has no idea anything changed until its own next unrelated re-render triggers a refetch — if that ever happens at all.

### Pattern applied

Treat server data as its own category of state — with caching, deduplication, and invalidation — using a dedicated library instead of `useState`/`useEffect`.

```tsx
import { useQuery, useQueryClient, useMutation } from '@tanstack/react-query';

function OrderCountBadge() {
  const { data: orders } = useQuery({
    queryKey: ['orders'],
    queryFn: () => orderService.list(),
  });
  return <Badge>{orders?.length ?? 0}</Badge>;
}

function OrderTable() {
  const { data: orders, isLoading } = useQuery({
    queryKey: ['orders'],
    queryFn: () => orderService.list(),
  });
  const queryClient = useQueryClient();

  const cancelMutation = useMutation({
    mutationFn: (id: string) => orderService.cancel(id),
    onSuccess: () => queryClient.invalidateQueries({ queryKey: ['orders'] }),
  });

  if (isLoading) return <Spinner />;
  return (
    <table>
      {orders?.map((order) => (
        <OrderTableRow
          key={order.id}
          order={order}
          onCancel={() => cancelMutation.mutate(order.id)}
        />
      ))}
    </table>
  );
}
```

Both components share the exact same cache entry (`['orders']`) — React Query fetches it once, not twice. When `cancelMutation` succeeds, `invalidateQueries` tells every component using that key to refetch, so `OrderCountBadge` updates automatically without any manual coordination between the two components.

### Real-world example

React Query (TanStack Query) and SWR are the two dominant libraries built specifically to solve this — both are used in production by companies like Microsoft, Walmart, and countless startups specifically because hand-rolled `useState`/`useEffect` data fetching becomes unmanageable past a handful of screens sharing data. This is the production-grade evolution of Day 3's `useFetch` custom hook.

### Practice exercise

Convert a hand-rolled `useProducts()` hook (using `useState`/`useEffect`) into a `useQuery({ queryKey: ['products'], queryFn: productService.list })` call, and add a `useMutation` for `productService.updateStock` that invalidates the `['products']` query on success.

### When to use

- Any data that originates from a server and can be needed by more than one component, or can go stale — which in practice is most of a real app's data.

### When NOT to use

- Data that's genuinely local and never touches a server (a form's draft input, whether a dropdown is open). Server state libraries solve caching/invalidation problems that local UI state doesn't have — using `useQuery` for `isModalOpen` is solving a problem that doesn't exist.

---

## 2. Effect Synchronization Pattern

### The problem

`useEffect` is often taught as "run this when the component mounts," which leads to bugs the moment a dependency changes. A product detail page keyed by `productId` that fetches data in a mount-only effect keeps showing the *previous* product's data when the user navigates from one product to another without unmounting the component (e.g., clicking a "related product" link that reuses the same route).

### Bad approach (effect not synchronized with its inputs)

```tsx
function ProductDetail({ productId }: { productId: string }) {
  const [product, setProduct] = useState<Product | null>(null);

  useEffect(() => {
    productService.getById(productId).then(setProduct);
  }, []); // missing productId — stale data on navigation

  return product ? <ProductView product={product} /> : <Spinner />;
}
```

### Pattern applied

Treat `useEffect` not as a lifecycle hook but as a way to keep an external system (here, fetched data) *synchronized* with the props/state it depends on — meaning every value the effect reads belongs in its dependency array, and stale in-flight requests must be ignored.

```tsx
function ProductDetail({ productId }: { productId: string }) {
  const [product, setProduct] = useState<Product | null>(null);

  useEffect(() => {
    let isCurrent = true;
    setProduct(null); // clear stale data immediately
    productService.getById(productId).then((data) => {
      if (isCurrent) setProduct(data);
    });
    return () => {
      isCurrent = false; // ignore this request if productId changes before it resolves
    };
  }, [productId]);

  return product ? <ProductView product={product} /> : <Spinner />;
}
```

Now every navigation between products re-runs the effect with the new `productId`, and the `isCurrent` flag guarantees that if a user rapidly clicks between three products, only the response for the *last* requested `productId` is ever applied — an earlier, slower response arriving late is silently discarded instead of overwriting newer data.

### Real-world example

This exact race condition (fast user navigation causing an older response to overwrite a newer one) is a well-documented class of bug the React team specifically calls out in the official docs' "You Might Not Need an Effect" and data-fetching guidance — it's also precisely the problem React Query and SWR solve automatically via request deduplication and cancellation, which is why Pattern 1 (Server State) is often reached for instead of hand-writing this synchronization logic yourself.

### Practice exercise

A `UserProfile({ userId })` component fetches `userService.getById(userId)` in a `useEffect` with an empty dependency array. Fix it using the `isCurrent` flag technique above so that rapidly switching between two user profiles never shows the wrong user's data.

### When to use

- Any effect whose logic depends on props or state that can change while the component stays mounted — which describes nearly every data-fetching effect that isn't already handled by a library from Pattern 1.

### When NOT to use

- If you're already using React Query/SWR for the fetch itself (Pattern 1), you don't need to hand-write this synchronization — the library does exactly this internally. Reach for this pattern only in the hand-rolled `useEffect` cases that remain (e.g., syncing with a non-fetch browser API like `document.title` or a WebSocket subscription).

---

## 3. Optimistic UI Pattern

### The problem

Clicking "Cancel" on an order and waiting for the server round-trip before showing any change makes the UI feel sluggish, especially on slower connections — the user clicks, nothing visibly happens for 300–800ms, and they're left wondering if the click registered at all.

### Pattern applied

Update the UI immediately, as if the action already succeeded, then roll back only if the server actually rejects it.

```tsx
function useCancelOrder() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (orderId: string) => orderService.cancel(orderId),

    onMutate: async (orderId) => {
      await queryClient.cancelQueries({ queryKey: ['orders'] });
      const previousOrders = queryClient.getQueryData<Order[]>(['orders']);

      // Optimistically mark it cancelled before the server responds
      queryClient.setQueryData<Order[]>(['orders'], (old) =>
        old?.map((o) => (o.id === orderId ? { ...o, status: 'cancelled' } : o)),
      );

      return { previousOrders }; // saved for rollback
    },

    onError: (_err, _orderId, context) => {
      // Server rejected it — roll back to the pre-optimistic state
      queryClient.setQueryData(['orders'], context?.previousOrders);
    },

    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ['orders'] });
    },
  });
}
```

The order row shows "Cancelled" the instant the button is clicked. If the server later responds with an error (the order already shipped, say), the `onError` handler restores the previous list and — ideally paired with a toast notification — tells the user the cancellation didn't actually go through.

### Real-world example

This is exactly how liking a post works on Instagram, X (Twitter), or Facebook: the heart/like icon fills in and the count increments the instant you tap it, well before any server confirmation, because the underlying action succeeds the overwhelming majority of the time and the perceived speed matters far more than waiting for round-trip confirmation on the rare case it fails.

### Practice exercise

Apply the same `onMutate`/`onError`/`onSettled` structure to a `useUpdateStock` mutation, so that changing a product's stock count in the UI updates instantly, with rollback if the server rejects the update (e.g., a concurrent sale already dropped stock to zero).

### When to use

- An action that succeeds the vast majority of the time, where perceived responsiveness matters more than the rare failure case — likes, toggling a favorite, marking a todo done, cancelling an order.

### When NOT to use

- High-stakes or frequently-failing actions (submitting a payment, an action with real side effects that are expensive to visibly undo) where showing a false success and then reversing it would confuse or alarm the user more than a brief loading state would.

---

## 4. Pagination Pattern

### The problem

Fetching and rendering all 50,000 rows of an Orders table at once means a massive JSON payload, a slow initial render, and a scrollbar nobody will ever scroll to the bottom of. Nothing about "show me the orders" requires having every single one in memory simultaneously.

### Pattern applied

Fetch and render data in fixed-size pages, requesting only the current page from the server.

```tsx
function OrderTable() {
  const [page, setPage] = useState(1);
  const pageSize = 25;

  const { data, isLoading } = useQuery({
    queryKey: ['orders', page],
    queryFn: () => orderService.list({ page, pageSize }),
    // response: { orders: Order[], totalCount: number }
  });

  if (isLoading) return <Spinner />;
  const totalPages = Math.ceil((data?.totalCount ?? 0) / pageSize);

  return (
    <>
      <table>{data?.orders.map((o) => <OrderTableRow key={o.id} order={o} />)}</table>
      <Pagination
        currentPage={page}
        totalPages={totalPages}
        onPageChange={setPage}
      />
    </>
  );
}
```

Including `page` in the `queryKey` means React Query caches each page separately — going back to page 1 after viewing page 2 is instant, served from cache, not refetched.

### Real-world example

Every admin dashboard with a data table — Stripe's Dashboard (Payments, Customers lists), GitHub's issues list, Gmail's inbox (25/50/100 per page) — paginates for exactly this reason: server-side pagination keeps both the database query and the payload bounded regardless of how many total records exist.

### Practice exercise

Add pagination to a `UserTable` component the same way, including a page-size selector (10/25/50) that resets `page` back to 1 whenever the page size changes (to avoid landing on a now out-of-range page).

### When to use

- Any list backed by a dataset whose size isn't tightly bounded and known to be small — which in practice is almost every real table in an admin dashboard.

### When NOT to use

- A genuinely small, fixed list (a dropdown of 8 country options, a settings page's 5 toggle switches) — paginating something that comfortably fits on one screen adds UI and state for no benefit.

---

## 5. Infinite Scroll Pattern

### The problem

Numbered pagination is the right choice for an Orders table an admin needs to jump around in ("go to page 12"), but it's the wrong interaction for a feed-like list — a live "Recent Activity" log on the dashboard — where users expect to keep scrolling and have more simply appear, the way virtually every social feed works today.

### Pattern applied

Fetch the next page automatically as the user scrolls near the bottom, appending it to the existing list instead of replacing it.

```tsx
function ActivityFeed() {
  const { data, fetchNextPage, hasNextPage, isFetchingNextPage } = useInfiniteQuery({
    queryKey: ['activity'],
    queryFn: ({ pageParam = 1 }) => activityService.list({ page: pageParam }),
    getNextPageParam: (lastPage) => lastPage.nextPage ?? undefined,
  });

  const activities = data?.pages.flatMap((page) => page.items) ?? [];

  return (
    <div>
      {activities.map((activity) => (
        <ActivityRow key={activity.id} activity={activity} />
      ))}
      {hasNextPage && (
        <SentinelObserver onVisible={() => fetchNextPage()} loading={isFetchingNextPage} />
      )}
    </div>
  );
}
```

`SentinelObserver` here wraps an `IntersectionObserver` watching an invisible element at the bottom of the list — when it scrolls into view, `fetchNextPage()` fires and the new page is appended via `data.pages`, not a page replacement.

### Real-world example

Instagram, X (Twitter), LinkedIn's feed, and TikTok's video stream are the canonical examples — none of them show page numbers because the interaction model is "keep scrolling for more," not "navigate to a specific position," which is precisely the distinction that decides Pagination (Pattern 4) versus Infinite Scroll (Pattern 5).

### Practice exercise

Convert the `OrderTable`'s numbered pagination from Pattern 4 into an infinite-scrolling order list for a "Recent Orders" widget on the dashboard's home page, using `useInfiniteQuery`, and consider (as a design question, not just code) why numbered pagination should stay on the full Orders page even after this change.

### When to use

- Feed-like content where users browse forward continuously and rarely need to jump to a specific position — activity logs, notification lists, social feeds, search results a user skims rather than navigates precisely.

### When NOT to use

- Anywhere a user needs to reference or return to a specific position ("that order was on page 3") or needs a sense of total scope (how many pages exist). Infinite scroll also makes reaching a page's footer content impossible, which is why it's a poor fit for an admin dashboard's main data tables — Pagination (Pattern 4) is usually the right choice there instead.

---

## How the five patterns fit together

Server State (Pattern 1) is the foundation underneath the rest: a shared, cached representation of backend data that any component can read or invalidate. Effect Synchronization (Pattern 2) is what a hand-rolled fetch needs to get right if you're not using a Server State library for it — and is largely handled for you once you are. Optimistic UI (Pattern 3) is a refinement on top of Server State's mutations, trading a small rollback risk for a much snappier feel on actions that usually succeed. Pagination (Pattern 4) and Infinite Scroll (Pattern 5) are two different answers to the same underlying question — how to fetch a large dataset in bounded chunks — chosen based on whether users need to navigate to a specific position (Pagination) or just keep browsing forward (Infinite Scroll); both integrate directly with a Server State library's caching (`queryKey: ['orders', page]` or `useInfiniteQuery`) rather than being built as a separate concern.
