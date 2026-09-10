# Day 10: Classic Software Design Patterns

Patterns covered today:

1. [Strategy Pattern](#1-strategy-pattern)
2. [Factory Pattern](#2-factory-pattern)
3. [Observer Pattern](#3-observer-pattern)
4. [Pub/Sub Pattern](#4-pubsub-pattern)
5. [Singleton Pattern](#5-singleton-pattern)
6. [Proxy Pattern](#6-proxy-pattern)

Same running scenario as every day so far: an e-commerce admin dashboard. These last six aren't React-specific — they're the classic "Gang of Four"-era software design patterns, predating React by decades. The point of today isn't to learn something new about React; it's to recognize that most of what this checklist has covered (Provider Pattern, Server State, Dependency Injection) are really these same, much older ideas wearing React's clothes.

Study method for each pattern: understand the problem, see the bad approach, see the pattern applied, real-world example, practice exercise, know when (not) to use it.

---

## 1. Strategy Pattern

### The problem

The dashboard needs to calculate shipping cost, and the rule keeps changing depending on the order: standard shipping is a flat fee, express shipping is a higher flat fee plus a per-kilogram surcharge, and orders over a threshold ship free. Writing this as one function with a growing `if/else if/else` chain means every new shipping rule (a future "same-day" option) means editing that same function again, risking breaking the rules that already worked.

### Bad approach (one function, growing conditionals)

```tsx
function calculateShipping(order: Order): number {
  if (order.shippingMethod === 'standard') {
    return 5.99;
  } else if (order.shippingMethod === 'express') {
    return 12.99 + order.weightKg * 0.5;
  } else if (order.total > 100) {
    return 0;
  }
  return 5.99;
}
```

### Pattern applied

Extract each calculation rule into its own interchangeable function (a "strategy"), and pick which one to use at runtime based on the order.

```tsx
type ShippingStrategy = (order: Order) => number;

const standardShipping: ShippingStrategy = () => 5.99;
const expressShipping: ShippingStrategy = (order) => 12.99 + order.weightKg * 0.5;
const freeShipping: ShippingStrategy = () => 0;

function getShippingStrategy(order: Order): ShippingStrategy {
  if (order.total > 100) return freeShipping;
  if (order.shippingMethod === 'express') return expressShipping;
  return standardShipping;
}

function calculateShipping(order: Order): number {
  return getShippingStrategy(order)(order);
}
```

Adding a "same-day" strategy later means writing one new function and adding one line to `getShippingStrategy` — every existing strategy stays untouched and untested-for-regressions.

### Real-world example

This is exactly how payment processing works in most checkout systems — a `PaymentStrategy` interface with `CreditCardStrategy`, `PayPalStrategy`, and `BankTransferStrategy` implementations, selected based on what the customer picks at checkout, is the standard architecture behind Stripe-integrated and Shopify-style checkout flows.

### Practice exercise

Add a `discountStrategy` the same way: a `percentageDiscount(order, rate)`, a `flatDiscount(order, amount)`, and a `noDiscount(order)`, selected by which discount type (if any) is attached to the order.

### When to use

- A behavior that has several interchangeable variants, selected by some condition, where the variants are likely to grow over time or need independent testing.

### When NOT to use

- Only one or two variants that will realistically never grow — a plain `if/else` is simpler and doesn't need the extra indirection of a strategy interface.

---

## 2. Factory Pattern

### The problem

Creating a notification object (`EmailNotification`, `SMSNotification`, `PushNotification`) directly with `new EmailNotification(...)` scattered across the codebase means every call site needs to know the exact class name and constructor shape for every notification type. Adding a new notification type, or changing how one is constructed (adding a required `retryCount` field), means finding and updating every single call site.

### Pattern applied

Centralize object creation behind one function that decides which concrete type to build, so callers never construct objects directly.

```tsx
interface Notification {
  send(message: string): Promise<void>;
}

class EmailNotification implements Notification {
  constructor(private email: string) {}
  async send(message: string) { /* send via email API */ }
}

class SMSNotification implements Notification {
  constructor(private phone: string) {}
  async send(message: string) { /* send via SMS gateway */ }
}

function createNotification(user: User): Notification {
  switch (user.preferredChannel) {
    case 'email': return new EmailNotification(user.email);
    case 'sms': return new SMSNotification(user.phone);
    default: throw new Error(`Unknown channel: ${user.preferredChannel}`);
  }
}

// Callers never see the concrete classes
const notification = createNotification(user);
await notification.send('Your order has shipped!');
```

Every call site depends only on `createNotification` and the `Notification` interface — never on `EmailNotification` or `SMSNotification` directly. Adding a `PushNotification` type means touching one function, not every call site that sends a notification.

### Real-world example

`document.createElement(tagName)` in the browser's own DOM API is a factory — it returns different concrete element types (`HTMLButtonElement`, `HTMLInputElement`) based on the string passed in, without the caller ever writing `new HTMLButtonElement()` directly. Redux Toolkit's `configureStore()` and React Query's `QueryClient` constructor options both apply the same idea to configuring complex objects from simpler input.

### Practice exercise

Build a `createRepository(type: 'rest' | 'in-memory')` factory (tying back to Day 5's Repository Pattern) that returns either a `restOrderRepository` or a `createInMemoryOrderRepository()`, so test setup code never has to know which concrete implementation exists behind the scenes.

### When to use

- Object creation that involves choosing between several concrete implementations based on input, or that requires enough setup logic that scattering `new X(...)` calls across the codebase would duplicate that logic.

### When NOT to use

- A single, simple class with one obvious way to construct it. Wrapping `new Order(...)` in a `createOrder()` factory that does nothing but call `new Order(...)` adds a layer with no decision-making inside it.

---

## 3. Observer Pattern

### The problem

The shopping cart's item count needs to show up in three unrelated places: a badge in the header, a summary in the cart drawer, and an analytics event fired on every change. If `addToCart()` directly calls `updateHeaderBadge()`, `updateCartDrawer()`, and `sendAnalyticsEvent()` by name, the cart logic is now tightly coupled to three specific UI functions — adding a fourth consumer (a "free shipping progress bar") means editing the cart's core logic again.

### Pattern applied

The cart doesn't know or care who's listening — it just notifies a list of subscribers whenever it changes, and anything can subscribe or unsubscribe independently.

```tsx
type Listener = (items: CartItem[]) => void;

class Cart {
  private items: CartItem[] = [];
  private listeners: Listener[] = [];

  subscribe(listener: Listener): () => void {
    this.listeners.push(listener);
    return () => {
      this.listeners = this.listeners.filter((l) => l !== listener);
    };
  }

  addItem(item: CartItem) {
    this.items = [...this.items, item];
    this.listeners.forEach((listener) => listener(this.items));
  }
}

const cart = new Cart();
const unsubscribeBadge = cart.subscribe((items) => updateHeaderBadge(items.length));
const unsubscribeDrawer = cart.subscribe((items) => updateCartDrawer(items));
cart.subscribe((items) => sendAnalyticsEvent('cart_updated', items));
```

Adding the "free shipping progress bar" is one new `cart.subscribe(...)` call, added wherever that feature lives — the `Cart` class itself never changes.

### Real-world example

This is the exact mechanism underneath React's own `useState`/`useSyncExternalStore`, Redux's `store.subscribe()`, and RxJS Observables — a subject (the cart, the store) maintains a list of subscribers and notifies them all on change, without the subject needing to know anything about who's listening or why.

### Practice exercise

Add an `unsubscribe` call for `unsubscribeDrawer` when the cart drawer component unmounts (mirroring how a React `useEffect`'s cleanup function calls the function returned by `subscribe`), and verify that further `addItem` calls no longer trigger `updateCartDrawer`.

### When to use

- One piece of state that multiple, independent, potentially-changing-in-number consumers need to react to, without the state owner needing a hardcoded list of who those consumers are.

### When NOT to use

- A single, fixed consumer relationship (one parent, one child) — passing a prop or calling a callback directly is simpler than setting up a subscription mechanism for a relationship that will only ever have one listener.

---

## 4. Pub/Sub Pattern

### The problem

Observer Pattern's `cart.subscribe(...)` still requires holding a direct reference to the `cart` instance to subscribe to it. But an `order:cancelled` event might need to be reacted to by code in completely unrelated parts of the app — a notification system, an analytics module, an inventory-restocking service — that have no reason to import or hold a reference to whatever object originally detected the cancellation.

### Pattern applied

Introduce a shared, decoupled event bus that publishers publish named events to, and subscribers listen for named events on — publisher and subscriber never reference each other directly, only the shared bus.

```tsx
type EventMap = {
  'order:cancelled': { orderId: string };
  'order:placed': { orderId: string; total: number };
};

class EventBus {
  private handlers: { [K in keyof EventMap]?: ((payload: EventMap[K]) => void)[] } = {};

  on<K extends keyof EventMap>(event: K, handler: (payload: EventMap[K]) => void) {
    (this.handlers[event] ??= []).push(handler);
  }

  emit<K extends keyof EventMap>(event: K, payload: EventMap[K]) {
    this.handlers[event]?.forEach((handler) => handler(payload));
  }
}

export const eventBus = new EventBus();

// In the order-cancellation feature, with no knowledge of who's listening
eventBus.emit('order:cancelled', { orderId: order.id });

// In a completely unrelated inventory module
eventBus.on('order:cancelled', ({ orderId }) => restockInventoryFor(orderId));

// In an unrelated analytics module
eventBus.on('order:cancelled', ({ orderId }) => trackEvent('order_cancelled', { orderId }));
```

The order-cancellation code has zero imports from the inventory or analytics modules, and vice versa — they're connected only through the shared `eventBus` and an agreed-upon event name.

### Real-world example

Node.js's built-in `EventEmitter` is exactly this pattern at the platform level, and browser-side libraries like `mitt` (used in many production Vue and React codebases) provide the same tiny event-bus API for exactly this kind of cross-feature decoupling. Message queues like RabbitMQ or Kafka implement the same publish/subscribe idea at a distributed-systems scale, between entirely separate services instead of modules in one app.

### Practice exercise

Add a `'user:logged_out'` event, published from the authentication module when a user logs out, and subscribed to by the cart module (to clear the cart) and a "recently viewed products" module (to clear that list) — neither of which should import anything from the authentication module.

### When to use

- Cross-cutting concerns that need to react to something happening in a completely unrelated part of the app, where introducing a direct import/dependency between the two features would be architecturally wrong.

### When NOT to use

- Two closely related pieces of code that already know about each other (a parent and its direct child, or a class and its own direct caller) — routing that through a global event bus adds indirection that makes the code harder to trace (you can no longer "find usages" of a function call; you have to search for a string event name instead).

---

## 5. Singleton Pattern

### The problem

Every part of the app that needs to make an API call constructs its own `new ApiClient()` — one for the Orders feature, another for Products, another for Users. Each one might end up with slightly different configuration (a different base URL typo, a missing auth header), and there's no single shared place to add something like request logging or a shared in-memory cache across every API call the app makes.

### Pattern applied

Ensure exactly one instance of a class exists for the entire app's lifetime, and provide a single, well-known way to access it.

```tsx
class ApiClient {
  private static instance: ApiClient;
  private constructor(private baseUrl: string) {} // private: can't be called with `new` from outside

  static getInstance(): ApiClient {
    if (!ApiClient.instance) {
      ApiClient.instance = new ApiClient('https://api.example.com');
    }
    return ApiClient.instance;
  }

  async get<T>(path: string): Promise<T> {
    const res = await fetch(`${this.baseUrl}${path}`);
    return res.json();
  }
}

// Every call site gets the exact same instance
const client1 = ApiClient.getInstance();
const client2 = ApiClient.getInstance();
console.log(client1 === client2); // true
```

There's exactly one `ApiClient` for the whole app's lifetime — adding a shared request-logging feature means editing this one class, and it's guaranteed to affect every caller, because there's no second instance to have missed.

### Real-world example

This is precisely what Day 5's Dependency Injection Pattern's `ServiceProvider` example was actually doing under the hood: `restOrderRepository` and `FetchApiClient` were module-level `const` exports, meaning JavaScript's own module system enforces the singleton for you — every file that imports them gets the exact same object reference, without needing the class-based `getInstance()` ceremony above. Redux's single store, and a single shared React Query `QueryClient` instance passed to `QueryClientProvider`, are both singletons for the same reason: exactly one shared source of truth for the whole app.

### Practice exercise

Refactor the class-based `ApiClient` singleton above into the JavaScript-module-based version instead — a single `const apiClient = { baseUrl: '...', get: async (path) => {...} }` object exported from `apiClient.ts` — and explain why this achieves the same guarantee without any `getInstance()` method at all.

### When to use

- Exactly one shared instance is genuinely required app-wide — a single API client configuration, a single global store, a single logging service — and multiple independent instances would cause real bugs (inconsistent config, duplicated in-memory state).

### When NOT to use

- As a substitute for proper Dependency Injection (Day 5). A class-based Singleton with a hardcoded `getInstance()` is hard to swap out in tests (you can't easily inject a fake), unlike the module-export or `ServiceProvider`-based approach — most modern JavaScript codebases achieve the same "one shared instance" guarantee through a plain module export instead of the classic Singleton class ceremony.

---

## 6. Proxy Pattern

### The problem

Every call to `productService.getById(id)` should ideally be logged (for debugging) and cached (to avoid re-fetching the same product twice in a row) — but adding `console.log` and cache-checking code directly inside `productService` clutters its actual responsibility (talking to the API) with cross-cutting concerns that have nothing to do with fetching data per se.

### Pattern applied

Wrap the real object in a proxy that intercepts calls to it, adding behavior (logging, caching, access control) around the real implementation without modifying it.

```tsx
function createCachingProxy(realService: typeof productService) {
  const cache = new Map<string, Product>();

  return new Proxy(realService, {
    get(target, prop) {
      if (prop === 'getById') {
        return async (id: string) => {
          if (cache.has(id)) {
            console.log(`Cache hit for product ${id}`);
            return cache.get(id);
          }
          console.log(`Fetching product ${id} from network`);
          const product = await target.getById(id);
          cache.set(id, product);
          return product;
        };
      }
      return target[prop as keyof typeof target];
    },
  });
}

const cachedProductService = createCachingProxy(productService);
await cachedProductService.getById('123'); // logs "Fetching...", hits network
await cachedProductService.getById('123'); // logs "Cache hit...", no network call
```

Every other method on `productService` (like `list()`) passes through untouched via the `get` trap's fallback — only `getById` gets the extra caching/logging behavior, and `productService` itself was never modified.

### Real-world example

Vue 3's entire reactivity system is built on JavaScript's native `Proxy` — every reactive object is actually a `Proxy` that intercepts property reads (to track dependencies) and writes (to trigger re-renders), which is precisely the mechanism shown above, applied to plain data objects instead of a service. Many API client libraries also use a `Proxy` to auto-generate typed methods for every REST endpoint without hand-writing a function per endpoint.

### Practice exercise

Extend `createCachingProxy` with a cache expiration: store a timestamp alongside each cached product, and treat entries older than 60 seconds as a cache miss, re-fetching from the network and updating both the cached value and its timestamp.

### When to use

- Adding cross-cutting behavior (caching, logging, access control, validation) around an existing object's calls, without modifying that object's own source code — especially useful when the underlying object is from a third-party library.

### When NOT to use

- When you own the code and the added behavior is a core part of what the function should always do — at that point, just put the logic directly inside the real function instead of wrapping it in a separate proxy layer for no structural reason.

---

## How the six patterns fit together — and how Day 10 connects back to Days 1–9

Strategy and Factory are both about *creation and selection* — Strategy picks which *behavior* runs, Factory picks which *object* gets built — and both directly generalize Day 4's Dual-Mode API and Day 5's Repository Pattern (a repository factory choosing between `restOrderRepository` and an in-memory fake is a Factory returning different Strategy-like implementations of the same interface). Observer and Pub/Sub are both about *decoupled notification* — Observer when the subject and its listeners share a direct reference, Pub/Sub when they shouldn't — and Observer is quite literally the mechanism inside React's own state updates and Redux's `store.subscribe()`. Singleton and Proxy are both about *controlling access* to an object — Singleton controls *how many* instances exist, Proxy controls *what happens* on each access — and Singleton is exactly what Day 5's `ServiceProvider` was already relying on via plain module exports, while Proxy is the mechanism quietly running underneath Vue's reactivity and many typed API clients.

That closes the loop on all 10 days: nearly everything from Day 1's Composition through Day 9's Reliability patterns turns out to be a React-flavored application of one or more of these six much older ideas — component composition and hooks are how React expresses Strategy, Factory, Observer, and Dependency Injection; Context and Providers are React's take on Singleton-scoped shared state; and Portals, Proxies, and event buses all solve the same underlying problem classic software design identified decades before React existed: how to keep independent pieces of a growing system communicating without becoming tangled together.
