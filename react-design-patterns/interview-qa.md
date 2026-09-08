# React Design Patterns — Interview Q&A

Interview questions and answers, organized by day, matching [README.md](README.md)'s 10-day checklist. Add each day's questions here as you complete it.

---

## Day 1 — Component Fundamentals Patterns

### Q1. What is the Component Composition Pattern, and why does React favor it over inheritance?

**A.** Composition means building complex UI by combining small, focused components (via JSX nesting), instead of one component trying to handle every variation through props, or extending a base component through class inheritance. React's own docs state "React has a powerful composition model" and recommend composition over inheritance because:

- Small components are individually easy to understand, test, and reuse.
- New UI variants are just new JSX arrangements — you don't touch the base component's code or its prop list.
- Inheritance couples a child class tightly to its parent's implementation; composition only depends on the parent's public interface (its props/children), which is far more flexible.

### Q2. What's the difference between the `children` prop and passing a component through a regular named prop?

**A.** Both are composition mechanisms, but:

- `children` is a single, implicit slot — whatever is nested between a component's opening and closing JSX tags. Best when a component wraps *one* variable region of content (`<Button>Save</Button>`).
- A named prop (e.g. `<Layout header={<Header />} />`) gives you an explicit, named slot, and you can have several of them for different regions (header, sidebar, footer). This becomes the **Slot Pattern**, and combining multiple such slots with shared internal state becomes the **Compound Components Pattern** — both covered later in this checklist.

`children` is the right tool when there's one region of injected content; named props/slots are the right tool when there are several distinct regions.

### Q3. What is "prop drilling" and how does composition help avoid it?

**A.** Prop drilling is passing a prop down through several component layers that don't use it themselves, just to reach a deeply nested child. Composition avoids it by letting the top-level owner render the deep child directly as `children` (or via a slot prop), instead of relaying data through every layer:

```tsx
// Prop drilling
<Page user={user}>
  <Sidebar user={user}>
    <UserPanel user={user} />
  </Sidebar>
</Page>

// Composition — Page and Sidebar never need to know about `user`
<Page>
  <Sidebar>
    <UserPanel user={user} />
  </Sidebar>
</Page>
```

(For state that genuinely needs to reach many unrelated components, the **Provider Pattern** — Day 2 — is the complementary solution.)

### Q4. What is the difference between a Controlled and an Uncontrolled component?

**A.**
- **Controlled**: React state is the single source of truth. The input's `value` comes from state, and `onChange` updates that state — the DOM element always reflects React's state.
- **Uncontrolled**: The DOM itself owns the current value. React sets only an initial value via `defaultValue`/`defaultChecked`, and reads the current value on demand via a `ref`, rather than on every change.

### Q5. Why does React recommend controlled components for most forms?

**A.** Because React state as the single source of truth makes several things trivial that are awkward otherwise: live validation per keystroke, conditionally enabling/disabling submit buttons, formatting input as the user types, resetting or pre-filling fields programmatically, and keeping multiple inputs in sync with each other. All of these require reading or writing the value from *outside* the input at arbitrary times — which is exactly what controlled state gives you for free.

### Q6. What happens if you switch an input from uncontrolled to controlled (or vice versa) during its lifetime? Why does React warn about this?

**A.** React logs a warning: *"A component is changing an uncontrolled input to be controlled."* This happens if an input's `value` prop starts as `undefined` (uncontrolled) and later becomes a defined string (controlled), e.g.:

```tsx
const [value, setValue] = useState(); // undefined initially — uncontrolled
<input value={value} onChange={(e) => setValue(e.target.value)} />
```

React warns because an input can't reliably switch who "owns" its value mid-life without risking lost keystrokes or inconsistent DOM state. **Fix:** always initialize controlled state with a defined value, e.g. `useState("")` instead of `useState()`.

### Q7. Give a real reason you'd be forced to use an uncontrolled component.

**A.** `<input type="file">`. For security reasons, browsers do not allow JavaScript (and therefore React) to set the `value` of a file input programmatically — only the user can pick a file. So a file input can never be a `value`-driven controlled component; you must read `inputRef.current.files` via a `ref` instead.

### Q8. Can a single reusable Input/Checkbox component support both controlled and uncontrolled usage for consumers of your component library?

**A.** Yes — this is a distinct, more advanced pattern (**Dual-Mode Controlled/Uncontrolled API Pattern**, covered on Day 4). The component checks whether a `value` prop was passed: if it was, it operates in controlled mode (uses the prop + calls `onChange`); if not, it falls back to internal state seeded from `defaultValue`. Libraries like Radix UI and Headless UI components are built this way so consumers can choose either mode.

### Q9. Coding question: Convert this uncontrolled input into a controlled one that disables the submit button until the value is non-empty.

```tsx
function NameForm() {
  const ref = useRef<HTMLInputElement>(null);
  return (
    <form>
      <input ref={ref} defaultValue="" />
      <button type="submit">Submit</button>
    </form>
  );
}
```

**A.**

```tsx
function NameForm() {
  const [name, setName] = useState("");
  return (
    <form>
      <input value={name} onChange={(e) => setName(e.target.value)} />
      <button type="submit" disabled={name.trim().length === 0}>
        Submit
      </button>
    </form>
  );
}
```

Key points an interviewer looks for: state initialized to `""` (not `undefined`, to avoid the controlled/uncontrolled warning), `value` bound to state, `onChange` updating state, and the `disabled` condition derived from that same state.

### Q10. Does using controlled components hurt performance in large forms? How would you address that?

**A.** Every keystroke in a controlled input triggers a state update and a re-render of that component (and its subtree, unless memoized). For a handful of fields this is invisible; for very large forms (50+ fields) it can add up. Mitigations: colocate each field's state as close to that field as possible (avoid one giant state object for the whole form so unrelated fields don't re-render together), memoize expensive child components with `React.memo`, or use a form library (React Hook Form) that keeps most fields uncontrolled internally via refs and only re-renders on submit/validation.

---

## Day 2 — State Management Patterns

### Q1. What problem does React Context solve, and when should you reach for it?

**A.** Context lets any descendant component read a value directly from an ancestor, without every intermediate component having to accept and forward it as a prop ("prop drilling"). Reach for it when data is genuinely cross-cutting — theme, locale, current user, feature flags — needed by components scattered around the tree, not just a parent and its immediate child (plain props are simpler there).

### Q2. What's the difference between just using `createContext`/`useContext` directly versus the "Provider Pattern"?

**A.** Raw Context only gives you a way to pass a value down the tree. The Provider Pattern wraps that in a dedicated `<XProvider>` component that owns the state/logic, plus a custom hook (`useAuth()`, `useTheme()`) that reads the context *and* validates it was actually provided — throwing a clear error like `"useAuth must be used within an AuthProvider"` instead of silently returning `undefined`. It also hides the raw context object from consumers entirely, so they never import it directly.

### Q3. Why should you always expose a custom hook (`useAuth`) instead of letting consumers call `useContext(AuthContext)` directly?

**A.** Three reasons: (1) **Safety** — the hook can check `if (!ctx) throw new Error(...)`, catching "used outside provider" bugs immediately at the call site instead of a silent `undefined` bug appearing later. (2) **Encapsulation** — consumers never need to know a Context object exists; you can refactor the internal implementation (even move away from Context entirely) without touching consumer code. (3) **Discoverability** — `useAuth()` documents intent far better than `useContext(AuthContext)`.

### Q4. What's a major performance pitfall of Context, and how do you fix it?

**A.** Every component that consumes a context re-renders whenever that context's value changes — even if the consumer only cares about part of it. Two common causes and fixes:
- Passing a new object literal as the `value` prop on every render (`<Ctx.Provider value={{ user, login }}>`) creates a new reference every time, forcing all consumers to re-render even if nothing meaningful changed. In React 18 and earlier the fix is to wrap the value in `useMemo` so the reference is stable unless a dependency actually changes; in React 19 with the React Compiler enabled, this is memoized automatically and manual `useMemo` is no longer needed for this case.
- One large context holding many unrelated fields means a change to any field re-renders every consumer of the whole context. Fix: split into multiple smaller, focused contexts (e.g. separate `UserContext` and `ThemeContext`) so consumers only re-render for the slice they actually use.

### Q5. When would you reach for `useReducer` instead of several `useState` calls?

**A.** When state has multiple related fields that must update together and staying in sync by hand across several `useState` setters is error-prone (e.g. cart `items` and `total`), when there are many distinct kinds of updates (more than 3–4 "events"), or when the next state depends on the previous state in a non-trivial way. If it's a couple of independent, simple fields, plain `useState` is simpler and has less boilerplate — don't reach for `useReducer` just because it "looks more professional."

### Q6. How does `useReducer` compare to Redux?

**A.** They share the exact same core idea: a pure reducer function `(state, action) => newState`, and you trigger transitions by dispatching action objects instead of calling setters directly. The difference is scope — `useReducer` manages state local to one component subtree (usually paired with Context to share it), while Redux manages a single global store outside the component tree, with middleware, devtools, and time-travel debugging built around that same reducer concept. If you're comfortable with `useReducer`, you already understand Redux's core mental model.

### Q7. Coding question: write a reducer for a shopping cart supporting `ADD_ITEM` and `REMOVE_ITEM`, keeping a running `total` in sync.

```tsx
type CartItem = { id: string; price: number };
type CartState = { items: CartItem[]; total: number };
type CartAction =
  | { type: "ADD_ITEM"; item: CartItem }
  | { type: "REMOVE_ITEM"; id: string };
```

**A.**

```tsx
function cartReducer(state: CartState, action: CartAction): CartState {
  switch (action.type) {
    case "ADD_ITEM": {
      const items = [...state.items, action.item];
      return { items, total: items.reduce((sum, i) => sum + i.price, 0) };
    }
    case "REMOVE_ITEM": {
      const items = state.items.filter((i) => i.id !== action.id);
      return { items, total: items.reduce((sum, i) => sum + i.price, 0) };
    }
    default:
      return state;
  }
}
```

Key points an interviewer looks for: the reducer is a pure function (no side effects, returns a new state object rather than mutating), every action recomputes `total` from `items` so the two fields can never drift out of sync, and a `default` case returning unchanged state for safety.

### Q8. Can you combine Context and `useReducer`? Why would you?

**A.** Yes — this is one of the most common patterns for app-wide state without an external library. A Provider internally holds `const [state, dispatch] = useReducer(reducer, initialState)`, then shares `{ state, dispatch }` through Context. Any descendant can then call a custom hook (e.g. `useCart()`) to read state or dispatch actions, without prop drilling and without needing Redux for moderate-complexity global state.

### Q9. What's a common bug when creating a context's `value` inline in JSX, and how do you fix it?

**A.** Writing `<AuthContext.Provider value={{ user, login, logout }}>` creates a brand-new object every single render, even if `user`, `login`, and `logout` haven't actually changed — because object literals are never `===` to a previous render's object. Every consumer re-renders as a result, regardless of whether anything meaningful changed. **Fix (pre-React 19 / no compiler):** wrap it in `useMemo(() => ({ user, login, logout }), [user])` so the reference only changes when a real dependency changes. **In React 19 with the React Compiler enabled**, the compiler detects this at build time and memoizes it for you automatically — you can write the plain object literal and get the same stable-reference behavior without touching `useMemo` yourself. It's still worth knowing the manual fix, since plenty of production codebases haven't adopted the compiler yet.

### Q10. Name two popular libraries whose public API is essentially the Provider Pattern.

**A.** React Query (`<QueryClientProvider>` + `useQueryClient()`/`useQuery()`), Redux (`<Provider store={store}>` + `useSelector()`/`useDispatch()`), NextAuth (`<SessionProvider>` + `useSession()`), and Radix/shadcn UI's `<ThemeProvider>` + `useTheme()` all follow the exact same shape: a provider component owning state, paired with a custom hook for safe, encapsulated access.

---

## Day 3 — Logic Reuse Patterns

### Q1. What is a custom hook, and what problem does it solve?

**A.** A custom hook is just a function whose name starts with `use` that calls other hooks (`useState`, `useEffect`, etc.) internally, letting you extract stateful logic out of a component so it can be reused. It solves duplication: without it, logic like "fetch this URL, track loading/error state, clean up on unmount" gets copy-pasted into every component that needs it, and a bug fix in one copy never reaches the others. With a shared `useFetch(url)` hook, every consumer gets the fix automatically.

### Q2. Does a custom hook share state between the components that use it?

**A.** No — this is a common misconception. Each call to a custom hook gets its own independent state. If `ProductList` and `OrderList` both call `useFetch(url)`, they each get their own `data`/`loading`/`error` state; nothing is shared between them. What's shared is the *logic* (the implementation), not the state itself.

### Q3. Coding question: extract the duplicated fetch logic below into a reusable `useFetch` hook.

```tsx
function ProductList() {
  const [products, setProducts] = useState<Product[]>([]);
  const [loading, setLoading] = useState(true);
  useEffect(() => {
    fetch("/api/products").then((r) => r.json()).then((data) => {
      setProducts(data);
      setLoading(false);
    });
  }, []);
  // ...
}
```

**A.**

```tsx
function useFetch<T>(url: string) {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    let isActive = true;
    setLoading(true);
    fetch(url)
      .then((r) => r.json())
      .then((json) => isActive && setData(json))
      .finally(() => isActive && setLoading(false));
    return () => { isActive = false; };
  }, [url]);

  return { data, loading };
}

function ProductList() {
  const { data: products, loading } = useFetch<Product[]>("/api/products");
  // ...
}
```

Key points an interviewer looks for: generic `<T>` so it works for any resource type, an `isActive` flag (or `AbortController`) in the cleanup function to avoid setting state after unmount, and `url` in the dependency array so the hook refetches if the URL changes.

### Q4. What is Hook Composition, and how is it different from just calling multiple hooks inside a component?

**A.** Hook Composition means building a higher-level hook by calling several lower-level hooks *inside another custom hook*, then returning the combined/derived result — e.g. `useProductState(id)` internally calling `useAuth()`, `useWishlist()`, and `useCart()` to return `{ isWishlisted, cartQuantity, canPurchase }`. The difference from calling all three hooks directly inside a component is reuse and separation of concerns: if two different components (a product detail page and a product card) both need that same combination, composing it into one hook means the derivation logic exists in exactly one place instead of being copy-pasted into every consuming component.

### Q5. What is the Hook Factory Pattern, and when would you reach for it instead of writing hooks by hand?

**A.** A hook factory is a function that *generates* a hook, parameterized by whatever varies between use cases (a URL, a storage key, an event name) — e.g. `createResourceHook<T>(endpoint)` returns a ready-to-use `useResource()` hook. Reach for it when you need the same *shape* of hook for 3+ resources that differ only in a parameter (`useUsers`, `useProducts`, `useOrders` all fetching different endpoints with identical logic). For only one or two resources, or when each one's logic genuinely diverges, a factory just adds indirection — write them by hand instead.

### Q6. In a `createApiHook<T>(url)`-style factory, what does the factory actually return, and why is that useful?

**A.** It returns a *function* (the hook itself), not the data directly — calling `createApiHook<T>(url)` produces a new `useX()` hook that, when called inside a component, internally manages its own `data`/`loading`/`error` state via `useState`/`useEffect`. This is useful because you can generate an unlimited number of independent, fully-functional hooks (one per endpoint) from a single implementation, and each generated hook still follows React's rules of hooks correctly since it's a real hook, not a plain utility function.

### Q7. What's a real risk of overusing custom hooks or hook composition?

**A.** Over-abstracting a single, one-off piece of logic into a named hook before it has a second consumer — this adds an indirection layer (you now have to jump to another file to understand what a component does) with no actual reuse benefit yet. The rule of thumb from this checklist's Day 1–3 patterns: extract only once you have (or clearly anticipate) a second consumer, not preemptively.

### Q8. How do Custom Hooks, Hook Composition, and Hook Factory relate to each other in a real codebase?

**A.** They typically layer: a Hook Factory generates many low-level, `useFetch`-style Custom Hooks consistently (e.g. one per API resource); Hook Composition then sits a level above, combining several of those (possibly factory-generated) hooks into the exact shape a specific screen or component needs. You rarely pick just one — a mature dashboard codebase usually has all three working together.

### Q9. What's the relationship between the Custom Hooks Pattern here and libraries like React Query or SWR?

**A.** React Query and SWR are productionized, battle-tested versions of the same `useFetch`-style custom hook idea, extended with caching, request deduplication, background refetching, and stale-data handling that a hand-rolled `useFetch` doesn't have. Understanding why you'd extract `useFetch` in the first place is exactly what makes adopting React Query later make sense — it's solving the same duplication problem, just with far more production-grade behavior built in. (This connects directly to the **Server State Pattern**, Day 7.)

---

## Day 4: Advanced Component API Patterns

### Q1. What is the Compound Components Pattern, and what problem does it solve?

**A.** Compound Components is a set of components that work together as a family (like `Tabs`, `Tabs.List`, `Tabs.Tab`, `Tabs.Panel`), sharing internal state through React Context instead of the parent passing that state down as props. It solves the rigidity of a config-object API: a `Tabs` component that takes a `tabs: { label, content }[]` prop can only support what that shape anticipated. Compound components let the consumer write plain JSX, so adding an icon, a badge, or a tooltip to one tab is just JSX, not a new prop on the config object.

### Q2. How does a subcomponent like `Tabs.Tab` get access to shared state without the parent passing it a prop?

**A.** The top-level component (`Tabs`) wraps its children in a Context Provider holding the shared state (e.g. `{ active, setActive }`). Each subcomponent calls a custom hook (`useTabsContext()`) that reads from that context. This is the same mechanism as the Provider Pattern from Day 2, just applied so the "consumers" are a fixed family of subcomponents instead of arbitrary parts of the app.

### Q3. Why attach subcomponents as properties on the parent (`Tabs.List = TabList`) instead of exporting them separately?

**A.** It keeps the API self-documenting and scoped: importing `Tabs` gives you `Tabs.List`, `Tabs.Tab`, and `Tabs.Panel` together, signaling they're meant to be used only inside `<Tabs>`, rather than as standalone components that happen to share a name convention. It also avoids polluting the module's export list with several tightly coupled pieces.

### Q4. What's the difference between the Compound Components Pattern and the Provider + Compound Components Pattern?

**A.** They're the same underlying mechanism (subcomponents sharing state via Context) at different levels of rigor. Plain Compound Components (like `Tabs` sharing just an `active` index) can get away with a simple Context and a small hook. Provider + Compound Components is for cases where the shared state is complex enough (multiple fields, side effects, a callback notifying a parent form) that it needs the fuller treatment from Day 2's Provider Pattern: a dedicated provider component, and a custom hook that throws a clear error if a subcomponent is used outside its provider.

### Q5. Coding question: a `SelectItem` component reads `select` from `useSelectContext()`. What happens if someone renders `<Select.Item>` outside of a `<Select>`, and how do you make the failure clear instead of confusing?

**A.**

```tsx
function useSelectContext() {
  const ctx = useContext(SelectContext);
  if (!ctx) throw new Error("Select subcomponents must be used inside <Select>");
  return ctx;
}
```

Without the `if (!ctx) throw`, `useContext` would silently return the context's default value (often `null` or `undefined`), and `SelectItem` would fail later with a cryptic error like "Cannot read properties of null" when it tries to call `select(...)`. The explicit throw fails immediately, at the exact call site, with a message that tells the developer exactly what's wrong.

### Q6. What is the Dual-Mode (Controlled/Uncontrolled) API Pattern, and how is it different from the Controlled/Uncontrolled Component Pattern from Day 1?

**A.** Day 1's Controlled and Uncontrolled patterns describe two different ways *you*, as the app developer, can use a component: pass `value` and own the state yourself, or pass `defaultValue` and let the DOM/component own it. The Dual-Mode API Pattern is about *building* a component (typically for a shared library) that supports both of those usage styles at once, so its consumers can choose either one without the library author writing two separate components.

### Q7. Coding question: explain what `useControllableState` does and why `controlledValue !== undefined` is the check used to decide the mode.

```tsx
function useControllableState<T>(controlledValue: T | undefined, defaultValue: T) {
  const [internalValue, setInternalValue] = useState(defaultValue);
  const isControlled = controlledValue !== undefined;
  const value = isControlled ? controlledValue : internalValue;
  return [value, setInternalValue] as const;
}
```

**A.** It always keeps an internal `useState` around (for the uncontrolled case), but the actual `value` it returns is either the caller-supplied `controlledValue` (if one was passed) or that internal state. The `!== undefined` check is the signal for "was a value passed in at all": if the consumer never passes the `open` prop, it's `undefined`, so the hook falls back to managing its own state; the moment a consumer passes any defined value (even `false`), the component treats it as controlled and defers to that value entirely.

### Q8. If a component using `useControllableState` is controlled, does calling the returned setter function actually change what's rendered?

**A.** No, and this is a common source of bugs when implementing this pattern: in controlled mode, `setInternalValue` still updates the internal state, but the hook returns `controlledValue` (the prop), not `internalValue`, so the render is unaffected. The component must call the consumer's `onChange`/`onOpenChange` callback so the consumer updates their own state, which flows back down as a new `controlledValue` on the next render. This is exactly how the native `<input value={...} onChange={...}>` works too.

### Q9. When would you deliberately *not* build a dual-mode component, even in a shared UI library?

**A.** When every realistic consumer needs the same mode. Supporting both adds real complexity (the `useControllableState` indirection, and the mental overhead of "which mode am I in" for every future contributor touching the component). If a component is genuinely only ever used one way in practice, hardcode that mode and add the other later if a real need for it shows up, rather than building flexibility speculatively.

---

## Day 5 — Data & Service Layer Patterns

### Q1. What is the Service Layer Pattern, and what problem does it solve?

**A.** It's the practice of moving all network calls for a resource (e.g. orders) out of components and into a dedicated module (`orderService`), with methods like `list()`, `getById(id)`, `cancel(id)`. It solves duplication and drift: without it, every component that needs orders writes its own `fetch` call with its own copy of the URL, headers, and error handling, and a change to auth or the base URL has to be found and repeated everywhere those copies exist. With a service layer, there's exactly one place to change.

### Q2. How is the Repository Pattern different from the Service Layer Pattern if both end up as "a module with `list()`/`getById()` methods"?

**A.** A service layer is a concrete implementation, typically tied to one data source (a REST API). A repository is an *interface* describing which data operations exist, independent of how they're fulfilled — the service layer becomes one implementation of that interface (`restOrderRepository`), and you can write a second one (`createInMemoryOrderRepository`) for tests or Storybook. Consumers depend on the interface (`OrderRepository`), never on which concrete implementation is currently wired up.

### Q3. Why would a repository interface make a component easier to unit test?

**A.** Because the component (or the hook it uses) accepts `repository: OrderRepository` rather than importing a concrete `orderService` directly. A test can pass `createInMemoryOrderRepository([...seedData])` and assert on rendered output with zero network mocking, no intercepted `fetch` calls, and no flakiness from timing — the fake repository just resolves promises with in-memory data synchronously wrapped in `Promise.resolve`.

### Q4. Coding question: given this component that imports a concrete service directly, refactor it to depend on a repository interface instead so it can be tested without hitting the network.

```tsx
import { orderService } from '../services/orderService';

function OrderList() {
  const [orders, setOrders] = useState<Order[]>([]);
  useEffect(() => {
    orderService.list().then(setOrders);
  }, []);
  // ...
}
```

**A.**

```tsx
interface OrderRepository {
  list(): Promise<Order[]>;
}

function OrderList({ repository }: { repository: OrderRepository }) {
  const [orders, setOrders] = useState<Order[]>([]);
  useEffect(() => {
    repository.list().then(setOrders);
  }, [repository]);
  // ...
}

// production
<OrderList repository={restOrderRepository} />

// test
<OrderList repository={createInMemoryOrderRepository(seed)} />
```

Key points an interviewer looks for: the component's prop type is the interface, not a concrete class/object; `repository` is in the effect's dependency array; and the production vs. test wiring differs only at the call site, never inside `OrderList`.

### Q5. What is the Facade Pattern, and what problem does it solve in a data layer?

**A.** A facade wraps a multi-step operation that coordinates several services (e.g. "place an order" = check stock via `productService`, create the order via `orderService`, then notify the warehouse via `notificationService`) behind one simple function, like `checkoutFacade.placeOrder(cart)`. It solves the problem of that coordination sequence — which order to call things in, how to handle a partial failure — being re-derived (and drifting out of sync) in every component that can trigger the operation, instead of living in exactly one place.

### Q6. How is the Facade Pattern different from the Service Layer Pattern?

**A.** A service wraps calls to *one* resource/endpoint (`orderService` only knows about orders). A facade coordinates *multiple* services/repositories to fulfill one business operation that spans several of them. A facade's implementation typically calls into several services; a service's implementation typically just calls `fetch`.

### Q7. When would wrapping a single service call in a facade be the wrong move?

**A.** When there's no actual coordination logic — a facade method that does nothing but call `orderService.getById(id)` and return the result adds an indirection layer with zero behavior of its own. Facades earn their place only once a real operation spans multiple services with real sequencing or error-handling rules; for a single call, call the service directly.

### Q8. What is the Dependency Injection Pattern in a React context, and how does it build on the Provider Pattern from Day 2?

**A.** Instead of a module (a facade, a component) importing concrete services/repositories directly at the top of the file, those dependencies are supplied from outside via a `ServiceProvider` (React Context) at the composition root, and consumed through a `useServices()` hook — the exact same shape as Day 2's Provider Pattern (a provider owning a value, a custom hook that throws if used outside it), just applied to service instances instead of UI state like auth or theme.

### Q9. Coding question: this facade hard-imports its dependencies, making it impossible to test without hitting the real API. Refactor it to receive its dependencies instead.

```tsx
import { restOrderRepository } from '../repositories/orderRepository';
import { restProductRepository } from '../repositories/productRepository';

export const checkoutFacade = {
  async placeOrder(cart: CartItem[]) {
    const product = await restProductRepository.getById(cart[0].productId);
    // ...
    return restOrderRepository.create({ items: cart });
  },
};
```

**A.**

```tsx
type Services = {
  orderRepository: OrderRepository;
  productRepository: ProductRepository;
};

function createCheckoutFacade(services: Services) {
  return {
    async placeOrder(cart: CartItem[]) {
      const product = await services.productRepository.getById(cart[0].productId);
      // ...
      return services.orderRepository.create({ items: cart });
    },
  };
}

// production, wired via ServiceProvider/useServices()
const checkoutFacade = createCheckoutFacade({
  orderRepository: restOrderRepository,
  productRepository: restProductRepository,
});

// test
const checkoutFacade = createCheckoutFacade({
  orderRepository: createInMemoryOrderRepository(),
  productRepository: createInMemoryProductRepository(seed),
});
```

Key points an interviewer looks for: `createCheckoutFacade` is a factory taking dependencies as a parameter rather than importing them, and nothing inside the facade's returned methods changes between production and test — only what's passed in at construction time changes.

### Q10. When would you deliberately skip Dependency Injection, even in an app that already has a service layer and repositories?

**A.** When a dependency is a leaf-level, single-consumer utility that will never realistically be swapped or faked in a test — threading it through a `ServiceProvider` "for consistency" adds a context lookup and a factory function for no actual benefit. Reach for DI once something (usually a facade or a component with real business logic) genuinely needs to be tested with fakes, or genuinely needs to support more than one implementation (real API vs. mock, REST vs. GraphQL) at runtime.

---

## Day 6 — Application Architecture Patterns

### Q1. What is the Feature-Based Architecture Pattern, and what problem does it solve?

**A.** It means organizing folders by business domain/feature (`features/orders/`, `features/products/`) instead of by technical file type (`components/`, `hooks/`, `services/`). It solves the problem of a single feature's files being scattered across every type-based folder — understanding or deleting "how orders work" in a type-based structure means hunting through `components/`, `hooks/`, `services/`, and `types/` for every file that happens to relate to orders, instead of finding them all in one place.

### Q2. Does Feature-Based Architecture mean *every* component lives inside a feature folder?

**A.** No — genuinely shared, cross-feature UI (a generic `Button`, `Modal`, or the Day 5 `apiClient.ts`) still belongs in a top-level `components/`/`services/` folder. The rule is: if a file only makes sense in the context of one feature, it lives inside that feature's folder; if it's reused by multiple unrelated features, it stays shared. The failure mode to avoid is treating the shared folder as a dumping ground for everything, which recreates the original type-based mess one level up.

### Q3. What is Atomic Design, and what problem does it solve inside a shared component library?

**A.** Atomic Design (from Brad Frost) organizes UI components into layers of increasing complexity: atoms (a single HTML element with no business meaning, like `Button`), molecules (a few atoms composed together, like a `SearchBar`), and organisms (molecules/atoms plus real domain data, like an `OrderTableRow`). It solves the problem of components of wildly different complexity sitting at the same folder level with no signal about which ones are safe, generic building blocks versus which already encode business-specific logic — so a "make buttons rounder" change doesn't accidentally touch something that isn't a generic button anymore.

### Q4. Coding question: is this component an atom, a molecule, or an organism? Why?

```tsx
function PriceTag({ amount }: { amount: number }) {
  return <Badge>{formatCurrency(amount)}</Badge>;
}
```

**A.** A molecule. It composes one atom (`Badge`) with a small piece of formatting logic (`formatCurrency`), but it still has no knowledge of a specific business entity like an order or a product — it just renders a formatted number. An organism would be something like `OrderTableRow`, which takes a real `Order` object and assembles several atoms/molecules (a `PriceTag`, a `Button`) around actual domain data.

### Q5. Why do design systems like Material UI or Shopify's Polaris document components using atoms/molecules/organisms language?

**A.** Because it gives designers and engineers a shared vocabulary for "how generic is this component," independent of any specific product feature. A designer can say "this needs to be a new molecule" and an engineer immediately knows it composes a couple of atoms but shouldn't contain business logic — without that shared layering, the same conversation requires explaining the component's exact scope from scratch every time.

### Q6. What is the Container/Presentational Pattern, and what problem does it solve?

**A.** It splits a component into a **container** (owns data-fetching/state, renders no markup itself) and a **presentational** component (pure rendering, driven entirely by props, no data-fetching). It solves two problems at once: a component that both fetches and renders can't be reused with a different data source (e.g., already-fetched search results) without editing it, and it can't be unit-tested or shown in Storybook without mocking the network every time, even though the test only cares about rendering.

### Q7. Coding question: split this component into a presentational `OrderTableView` and a container `OrderTableContainer`.

```tsx
function OrderTable() {
  const [orders, setOrders] = useState<Order[]>([]);
  const [loading, setLoading] = useState(true);
  useEffect(() => {
    orderService.list().then((data) => {
      setOrders(data);
      setLoading(false);
    });
  }, []);
  if (loading) return <Spinner />;
  return <table>{orders.map((o) => <OrderTableRow key={o.id} order={o} />)}</table>;
}
```

**A.**

```tsx
function OrderTableView({ orders, loading }: { orders: Order[]; loading: boolean }) {
  if (loading) return <Spinner />;
  return <table>{orders.map((o) => <OrderTableRow key={o.id} order={o} />)}</table>;
}

function OrderTableContainer() {
  const [orders, setOrders] = useState<Order[]>([]);
  const [loading, setLoading] = useState(true);
  useEffect(() => {
    orderService.list().then((data) => {
      setOrders(data);
      setLoading(false);
    });
  }, []);
  return <OrderTableView orders={orders} loading={loading} />;
}
```

Key points an interviewer looks for: `OrderTableView` takes `orders`/`loading` as plain props and contains zero data-fetching code, and `OrderTableContainer` contains zero JSX of its own beyond rendering the view — the split is total, not partial.

### Q8. Why do many modern React codebases consider the Container/Presentational Pattern partly outdated, and what replaced it?

**A.** A custom hook (Day 3's Custom Hooks Pattern) usually achieves the same separation with less boilerplate: `const { orders, loading } = useOrders();` inside the *same* component that renders the table gives you data/rendering separation without needing a whole extra Container component and an additional layer of props. The pattern hasn't disappeared, though — it's still exactly the model Storybook uses (every story renders a presentational component with hardcoded props, never through a real fetch), and it's still relevant when a container needs to wrap multiple presentational children or do server-side work a hook can't.

### Q9. How do Feature-Based Architecture, Atomic Design, and Container/Presentational relate to each other in one real codebase?

**A.** They answer different-scoped questions and typically apply together rather than as alternatives. Feature-Based Architecture answers "where do files live" at the whole-app level (one folder per business domain). Atomic Design answers a narrower question inside the shared, cross-feature UI layer — how components are organized by complexity. Container/Presentational (or its modern hook-based equivalent) answers a question at the level of one component — how its data-fetching is separated from its rendering. A real dashboard's `features/orders/` folder can contain an `OrderTable` organism that's itself split into a view plus a `useOrders()` hook.

### Q10. When would introducing all three of these patterns at once be overkill?

**A.** In a small app with a handful of screens and components — a `features/` split for three files, an atoms/molecules/organisms taxonomy for eight components, and a Container/Presentational split for every component adds structure with no problem yet to justify it. As with every pattern in this checklist, the rule of thumb is the same: reach for the structure once the app's actual size and team size create the pain it solves, not preemptively.

---

## Day 7 — Server State & Data Fetching Patterns

### Q1. What is "server state," and why does treating it like local UI state cause bugs?

**A.** Server state is data that lives on a backend and is only mirrored into the frontend — orders, products, users — as opposed to local state that only ever exists in the browser (`isModalOpen`, `selectedTab`). Treating it like local state breaks down because server state can change without any action in the current app instance (another admin cancels an order), it's stale the instant it's fetched, and the same data is often needed by multiple unrelated components with no built-in way to keep them in sync — an `OrderCountBadge` and an `OrderTable` each fetching `/orders` independently have no idea when the other one's data changes.

### Q2. What specific problems do React Query/SWR solve that a hand-rolled `useFetch` (Day 3) doesn't?

**A.** Caching and deduplication (two components requesting the same `queryKey` share one cached result instead of firing two network calls), invalidation (`invalidateQueries` tells every consumer of a key to refetch after a mutation, without manually passing callbacks between unrelated components), and race-condition handling for rapidly changing inputs. A hand-rolled `useFetch` gives you loading/error state and reuse of the *logic*, but not a shared cache across separate hook calls — Day 3's Q2 already noted each call to a custom hook gets independent state, which is precisely the gap a server-state library closes.

### Q3. What is the Effect Synchronization Pattern, and how is "synchronization" different from the "run on mount" mental model of `useEffect`?

**A.** The "run on mount" framing leads to writing `useEffect(() => {...}, [])` and forgetting that the effect's body reads a prop or state value that can change later. The synchronization framing says: an effect should keep some external system (fetched data, `document.title`, a subscription) in sync with the reactive values it reads, which means every one of those values belongs in the dependency array — if `productId` is used inside the effect, `[productId]` must be the dependency list, so the effect re-runs and re-synchronizes whenever it changes.

### Q4. Coding question: this component shows stale data when `productId` changes without unmounting. Fix the race condition.

```tsx
function ProductDetail({ productId }: { productId: string }) {
  const [product, setProduct] = useState<Product | null>(null);
  useEffect(() => {
    productService.getById(productId).then(setProduct);
  }, []);
  return product ? <ProductView product={product} /> : <Spinner />;
}
```

**A.**

```tsx
function ProductDetail({ productId }: { productId: string }) {
  const [product, setProduct] = useState<Product | null>(null);
  useEffect(() => {
    let isCurrent = true;
    setProduct(null);
    productService.getById(productId).then((data) => {
      if (isCurrent) setProduct(data);
    });
    return () => { isCurrent = false; };
  }, [productId]);
  return product ? <ProductView product={product} /> : <Spinner />;
}
```

Key points an interviewer looks for: `productId` added to the dependency array (the actual bug), and an `isCurrent` flag set to `false` in the cleanup function so that if the user switches products again before the first request resolves, that stale response is discarded instead of overwriting the newer product's data.

### Q5. What is the Optimistic UI Pattern, and what's the real tradeoff it makes?

**A.** It updates the UI immediately as if an action already succeeded — before the server has actually confirmed it — then rolls back only if the server rejects it. The tradeoff is: on the common case (the action succeeds, which is the overwhelming majority of the time for things like likes or cancellations), the app feels instantaneous; on the rare failure case, the user briefly sees a false success that then reverses, which is a worse experience than a normal loading state would have been for that one case. It's a bet that the speed gained on the common path outweighs the confusion cost on the rare path.

### Q6. In a React Query optimistic mutation, what are `onMutate`, `onError`, and `onSettled` each responsible for?

**A.** `onMutate` runs before the request is sent — it should cancel any in-flight queries for that key (to prevent a race with the optimistic update), snapshot the current cached data (for rollback), and then apply the optimistic change to the cache. `onError` runs if the mutation fails — it restores the snapshot taken in `onMutate`, undoing the optimistic change. `onSettled` runs regardless of success or failure — typically it invalidates the query so the cache is refetched and guaranteed to reflect the server's actual current state, correcting any subtle drift the optimistic update introduced.

### Q7. Why is "liking a post" a good real-world fit for Optimistic UI, but "submitting a payment" is not?

**A.** A like almost always succeeds, is low-stakes if it briefly shows the wrong state, and is trivial to visually undo (un-fill the heart icon) — ideal conditions for optimism. A payment can genuinely fail for many real reasons (declined card, insufficient funds), carries real financial stakes, and showing "Payment successful" optimistically before reversing it would be actively misleading and alarming — exactly the pattern's own "when not to use" case.

### Q8. What's the difference between the Pagination Pattern and the Infinite Scroll Pattern, and how do you decide which one a feature needs?

**A.** Both solve the same underlying problem — fetching a large dataset in bounded chunks instead of all at once — but they serve different interaction models. Pagination (numbered pages, `queryKey: ['orders', page]`) fits data users need to navigate precisely within ("go to page 12," "that record was on page 3") and need a sense of total scope for. Infinite Scroll (`useInfiniteQuery`, appending pages as the user scrolls) fits feed-like content users browse forward through continuously without caring about a specific position — activity logs, social feeds. The deciding question is: does the user need to jump to or return to a specific position? If yes, Pagination; if the content is meant to be endlessly browsed, Infinite Scroll.

### Q9. Coding question: why does including `page` inside `queryKey: ['orders', page]` matter for a paginated table's performance?

**A.** React Query caches results per unique `queryKey`, so `['orders', 1]` and `['orders', 2]` are cached as two separate entries. Navigating from page 2 back to page 1 hits the cache instantly (no network request, no loading spinner) instead of refetching data that was already fetched moments ago. If `page` were left out of the key, every page would collide on the same cache entry, and switching pages would either serve wrong cached data or force a refetch every time — losing per-page caching entirely.

### Q10. A dashboard has an Orders table (admin needs to jump to specific orders) and a "Recent Activity" feed (a live log nobody navigates precisely within). Which pattern fits each, and why would swapping them be a mistake?

**A.** The Orders table should use Pagination — admins reference specific orders and benefit from knowing total scope ("143 orders across 6 pages"), which numbered pages provide and infinite scroll doesn't. The Activity feed should use Infinite Scroll — it's browsed forward continuously with no need to jump to "page 4 of activity." Swapping them would make the Orders table frustrating (no way to jump to a specific page, and reaching page footer/summary content becomes awkward as noted in this pattern's "when not to use"), and would make the Activity feed feel clunky (an admin monitoring a live log doesn't think in terms of page numbers, they just want to keep scrolling).

---

## Day 8 — Performance Patterns

### Q1. What is the Lazy Loading Pattern, and why does a normal top-of-file `import` defeat it?

**A.** Lazy Loading defers downloading a component's JavaScript until the moment it's actually needed at runtime, using `React.lazy(() => import('./X'))` instead of a static `import X from './X'`. A static import is resolved at build time and bundled unconditionally — even a component that only renders behind an `if (showModal)` check still has its code downloaded on every page load if it's imported normally, because the bundler can't know at build time that the runtime condition will often be false.

### Q2. Why does `React.lazy` require a `Suspense` boundary around it — what breaks without one?

**A.** `React.lazy` returns a component whose actual code isn't available yet at the moment React first tries to render it — the import is a `Promise` still in flight. React needs a way to render *something* while waiting, and `Suspense`'s `fallback` prop is exactly that — without wrapping a lazy component in `<Suspense>`, React throws because it has no fallback UI to show during the gap between "render requested" and "code loaded."

### Q3. What is the Code-Splitting Pattern, and how is it different from Lazy Loading — or is it the same thing?

**A.** They use the same underlying mechanism (`React.lazy` + dynamic `import()`), but at different scope and intent. Lazy Loading typically refers to deferring one specific, conditionally-rendered piece (a modal, a rarely-used panel). Code-Splitting refers to systematically applying that same technique across an app's structure — most commonly per route — so that visiting `/orders` only downloads the Orders page's chunk, not the Products or Users pages' code too. Code-Splitting is really Lazy Loading applied as an architectural default rather than a one-off optimization.

### Q4. Coding question: convert this eagerly-imported router into one that code-splits by route.

```tsx
import OrdersPage from './pages/OrdersPage';
import ProductsPage from './pages/ProductsPage';

function App() {
  return (
    <Routes>
      <Route path="/orders" element={<OrdersPage />} />
      <Route path="/products" element={<ProductsPage />} />
    </Routes>
  );
}
```

**A.**

```tsx
import { lazy, Suspense } from 'react';

const OrdersPage = lazy(() => import('./pages/OrdersPage'));
const ProductsPage = lazy(() => import('./pages/ProductsPage'));

function App() {
  return (
    <Suspense fallback={<PageSkeleton />}>
      <Routes>
        <Route path="/orders" element={<OrdersPage />} />
        <Route path="/products" element={<ProductsPage />} />
      </Routes>
    </Suspense>
  );
}
```

Key points an interviewer looks for: both imports converted to `lazy(() => import(...))`, and the whole `<Routes>` block wrapped in a single `Suspense` boundary (required for the lazy components to render at all).

### Q5. What is the Suspense Pattern, and what real coordination problem does it solve beyond "showing a spinner"?

**A.** Without Suspense, rendering several independently-loading pieces (two lazy-loaded components, or several Suspense-integrated data fetches) on one screen would require manually tracking multiple separate `isLoading` booleans and combining them by hand to decide when to stop showing a loading state. A `<Suspense>` boundary declaratively shows its `fallback` until *everything* inside it — regardless of how many independent async dependencies exist — has finished, with zero manual `isLoading` bookkeeping.

### Q6. What's the effect of nesting two `Suspense` boundaries instead of using one shared boundary around both components?

```tsx
<Suspense fallback={<PageSkeleton />}>
  <Header />
  <Suspense fallback={<ChartSkeleton />}>
    <RevenueChart />
  </Suspense>
  <RecentActivity />
</Suspense>
```

**A.** With nested boundaries, `Header` and `RecentActivity` can appear as soon as they're individually ready, and only `RevenueChart` (wrapped in its own inner boundary) shows its own `ChartSkeleton` while it specifically loads — a slow chart doesn't block the rest of the page. If everything shared one single outer boundary instead, the entire page would stay behind `PageSkeleton` until the slowest single piece (likely the chart) finished, even though `Header` and `RecentActivity` were ready much earlier.

### Q7. How does React Query's `useSuspenseQuery` connect Day 7's Server State Pattern to today's Suspense Pattern?

**A.** `useSuspenseQuery` makes a data-fetching hook integrate with the same `<Suspense>` mechanism originally built for lazy-loaded code — instead of a component checking `isLoading` and conditionally rendering a spinner itself, it "suspends" (throws a promise React catches) while the query is in flight, and the nearest `<Suspense>` boundary's `fallback` handles showing a loading state automatically. This means the exact same `<Suspense>` boundary already in place for a lazy-loaded route can also coordinate that route's data fetching, with one unified fallback mechanism instead of two separate loading concepts.

### Q8. What is the Virtualization Pattern, and why doesn't Day 7's Pagination Pattern make it unnecessary?

**A.** Virtualization renders only the currently-visible rows of a large list (plus a small buffer), recycling a small, constant number of DOM nodes as the user scrolls, rather than creating one DOM node per data item. Pagination limits how much data is *fetched and held* at once (e.g., 25 orders per page) but doesn't, by itself, solve a scenario where a single continuous, scrollable list still needs to show hundreds or thousands of already-loaded rows at once (a bulk-edit spreadsheet view, a long chat history) — that's specifically what Virtualization addresses, at the DOM-node level rather than the network-fetch level.

### Q9. Coding question: in a `useVirtualizer`-based list, why does the inner content wrapper need `height: virtualizer.getTotalSize()` if only ~25 rows are ever actually rendered?

**A.** The scrollable container needs a scrollbar that accurately reflects the *full* logical size of all items (e.g., 10,000 rows × 48px each), even though only a handful of those rows have real DOM nodes at any moment. Setting the inner wrapper's height to `getTotalSize()` reserves that correct total scrollable space, while each visible row is absolutely positioned via `transform: translateY(...)` at its correct offset within that space — giving native, correctly-proportioned scrolling behavior despite the DOM only containing a small fraction of the total items.

### Q10. When would adding virtualization to an already-paginated table be a bad idea?

**A.** If a table already shows a small, bounded page size (20-50 rows), there's no DOM-node-count problem left for virtualization to solve — the list is already small. Adding it anyway introduces real downsides for no gain: virtualized lists break the browser's native find-in-page (Ctrl+F) since off-screen rows don't exist in the DOM to be found, and they complicate handling variable-height content. Virtualization and Pagination address overlapping performance concerns from different angles, and a well-paginated table usually doesn't need both.

---

## Day 9 — Reliability Patterns

### Q1. What is the Error Boundary Pattern, and why does a bug in one small widget crash an entire React app without it?

**A.** React unmounts the whole component tree on an uncaught error thrown during render, regardless of how small or isolated the component that threw actually is — a `TypeError` inside one `OrderSummaryCard` takes down the header, sidebar, and every unrelated widget along with it. An Error Boundary is a component that catches render errors in its child tree (via `getDerivedStateFromError`/`componentDidCatch`) and renders a fallback instead, containing the crash to just the section it wraps.

### Q2. Why must `componentDidCatch` be a class component method — can't you write an Error Boundary as a function component with hooks?

**A.** As of React's current stable APIs, `getDerivedStateFromError` and `componentDidCatch` only exist on class components — there is no hook equivalent for catching render errors in children. This is one of the few remaining cases in modern React where a class component is still required; teams typically write one small, reusable `ErrorBoundary` class component once and never touch class syntax anywhere else in the codebase.

### Q3. Coding question: why is `componentDidCatch` used to call `logErrorToService(error, info)` rather than just relying on `getDerivedStateFromError` to show the fallback?

```tsx
static getDerivedStateFromError() {
  return { hasError: true };
}
componentDidCatch(error: Error, info: React.ErrorInfo) {
  logErrorToService(error, info);
}
```

**A.** `getDerivedStateFromError` runs during the render phase and must be a pure function with no side effects (it only returns new state) — React may call it multiple times and can't guarantee side effects inside it run correctly. `componentDidCatch` runs during the commit phase specifically to hold side effects like logging, so this is where error reporting (e.g., to Sentry) belongs. Skipping this and only setting `hasError` would silently swallow every caught error with no record of it ever happening.

### Q4. What is the Retry Pattern, and why is exponential backoff used instead of retrying immediately?

**A.** It automatically re-attempts a failed request a bounded number of times before surfacing an error, on the reasoning that many failures (a dropped Wi-Fi packet, a momentary server hiccup) are transient and would succeed if tried again shortly after. Exponential backoff (doubling the delay between attempts — 200ms, 400ms, 800ms) is used instead of instant, rapid retries because hammering a server that's already struggling (briefly overloaded, rate-limiting) with immediate repeated requests makes the underlying problem worse, not better.

### Q5. Coding question: why should this retry function distinguish between error types instead of retrying every failure the same way?

```tsx
async function fetchWithRetry<T>(fn: () => Promise<T>, maxAttempts = 3): Promise<T> {
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (attempt === maxAttempts) throw error;
      await new Promise((r) => setTimeout(r, 2 ** attempt * 100));
    }
  }
  throw new Error('unreachable');
}
```

**A.** A 4xx error (invalid input, bad auth token) will fail identically no matter how many times it's retried — retrying it three times with backoff just delays showing the user the real, actionable error by over a second for no benefit. A 5xx error or network failure, by contrast, is plausibly transient and worth retrying. Production code should add a `shouldRetry(error)` check that skips straight to throwing for 4xx-class errors and only applies the retry loop to 5xx/network failures.

### Q6. Which popular data-fetching libraries build the Retry Pattern in by default, and why is that the right default?

**A.** React Query and SWR both retry failed queries with exponential backoff automatically, without any extra configuration. This is the right default because momentary network blips are extremely common in real-world conditions (mobile networks, Wi-Fi handoffs, brief server hiccups), so silently retrying before ever bothering the user with an error is what a well-behaved app should do in the common case — the developer has to explicitly opt out (or configure `shouldRetry`) rather than opt in.

### Q7. What is the Fallback UI Pattern, and how is it different from just showing a spinner or a generic "Something went wrong" message?

**A.** It means designing a distinct, purposeful UI for every failure/empty state a component can actually reach, rather than one generic catch-all. A product page that's out of results (404) should show "this product doesn't exist" with a link back to the product list; a page that failed to load due to a network error should show "couldn't load this" with an actual retry button. Collapsing both into one generic error message forces the user to guess what happened and what to do next, when the component itself already knows exactly which case occurred.

### Q8. Coding question: this component only handles loading and success — what's missing, and why does it matter?

```tsx
function ProductDetail({ productId }: { productId: string }) {
  const { data: product, isLoading } = useQuery({
    queryKey: ['product', productId],
    queryFn: () => productService.getById(productId),
  });
  if (isLoading) return <Spinner />;
  return <ProductView product={product} />;
}
```

**A.** It has no handling for the query's `error` state at all — if the fetch fails, `product` stays `undefined` and `<ProductView product={undefined} />` either crashes (caught only if wrapped in an Error Boundary) or silently renders broken UI. A complete version checks `error`, and ideally distinguishes a 404 (show an empty state with navigation back) from any other failure (show a retry button) — exactly the three-state pattern (loading/error/success, with error itself often split further) that Fallback UI describes.

### Q9. What is the Portal Pattern, and what specific CSS problem does it solve that a `z-index` change alone can't?

**A.** A `Modal` rendered from deep inside a scrollable table (`OrderTable` → `OrderTableRow` → `OrderActionsMenu`) inherits ancestor CSS constraints like `overflow: hidden`, which physically clips the modal at the table's boundary — no `z-index` value, however high, can make content escape being clipped by an ancestor's `overflow: hidden`, because clipping and stacking order are different CSS mechanisms. `createPortal` solves this by rendering the modal's actual DOM nodes into a completely different location in the document (typically directly under `<body>`), bypassing the ancestor's `overflow` and stacking context entirely.

### Q10. If a Portal renders its content into a totally different DOM location, how does a click inside the portal still trigger an `onClick` handler defined on a component higher up in the React tree?

**A.** React's synthetic event system dispatches events based on the **component tree**, not the actual DOM tree — so even though `createPortal`'s output physically lives under `#modal-root` next to `<body>`, React still treats it as a child of the component that rendered it for the purposes of event bubbling. This is exactly why the practice of building `Dialog`/`Popover`/`Tooltip` components on `createPortal` (as Radix UI and shadcn/ui do) doesn't break click-outside handlers or other bubbling-based logic — from React's perspective, nothing about the component hierarchy changed, only where the DOM nodes physically render.

---

<!-- Add Day 10 questions below as you complete Day 10 -->
