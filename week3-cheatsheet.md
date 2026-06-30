# Week 3 — Frontend Engineering & UX

## React Advanced

**TypeScript with React**
- Type props with an interface, type state with generics: `useState<User | null>(null)`
- No `any` ever — if I don't know the shape yet, use `unknown` and narrow it
- Generic components: `function List<T>({ items }: { items: T[] })`

**Hooks — useCallback vs useMemo**
- `useMemo` caches a **value** (expensive calculation result)
- `useCallback` caches a **function reference** (so child components don't re-render unnecessarily)
- Both take a dependency array — only re-run when deps change
- Don't overuse these — premature memoization adds complexity for no gain on cheap renders

**Why components re-render**
- Parent re-renders → all children re-render by default, even if their props didn't change
- Fix: `React.memo()` on the child + `useCallback`/`useMemo` on the props passed down
- New object/array/function literals created inline (`onClick={() => ...}`) break memoization every time — pull them out

**Error Boundaries**
- Only class components can be error boundaries (no hook equivalent yet)
- Catches errors in render, lifecycle methods, constructors — NOT in event handlers or async code
- Wrap risky parts of the tree, not the whole app, so one broken widget doesn't kill everything

**State management + Context API**
- Local state (`useState`) → component-only data
- Context → avoid prop drilling for global-ish data (theme, auth, language)
- Context isn't a performance tool — every consumer re-renders when the value changes, so don't dump everything in one giant context
- Redux/Zustand when state logic gets complex or needs to live outside React

**Redux vs Zustand**
- Redux: more boilerplate, but battle-tested, great devtools, good for big teams/apps
- Zustand: way less boilerplate, just a hook (`create()`), no providers needed — I prefer this for solo/small projects
- Both keep state outside the component tree so it survives unmounts

**Theme switching (dark/light mode)**
- CSS variables (`--color-bg`, `--color-text`) + toggle a class/attribute on `<html>` or `<body>`
- Store user choice in Context (or localStorage) so it persists
- New CSS `light-dark()` function exists now but check browser support before relying on it

**Chrome DevTools / React DevTools / Lighthouse**
- React DevTools → inspect component tree, props, state, and "highlight updates when components render" to spot unnecessary re-renders
- Lighthouse → audits performance, accessibility, SEO, best practices — run it on the built (not dev) version for real numbers
- Aim for 90+ on accessibility, it's usually quick wins (alt text, contrast, labels)

**Dynamic routing (React Router)**
- `<Route path="/users/:id" element={<UserPage />} />` then `useParams()` to read `:id`
- Protected routes = wrapper component that checks auth state and either renders children or `<Navigate to="/login" />`
- Lazy load route components with `React.lazy()` + `<Suspense>` for big apps

---

## API Consumption & Data Handling

**Env variables (Vite)**
- Must be prefixed `VITE_` or Vite won't expose them to client code
- Access via `import.meta.env.VITE_API_BASE_URL`
- `.env` files are gitignored, ship a `.env.example` instead so others know what's needed

**TanStack Query**
- Replaces manual `useEffect` + `fetch` + loading/error state juggling
- `useQuery({ queryKey: ['posts'], queryFn: fetchPosts })` → gives back `data`, `isLoading`, `isError`, `error`
- Auto caching, auto refetch on window focus, retries on failure — all free
- `useMutation` for POST/PUT/DELETE, then `queryClient.invalidateQueries()` to refresh related data after

**Async/await**
- Always wrap in try/catch when calling APIs — unhandled promise rejections are silent and annoying to debug
- `async function fetchData() { try { const res = await fetch(url); ... } catch (err) { ... } }`

**Error handling**
- Network errors vs HTTP errors are different — `fetch` doesn't throw on 404/500, gotta check `res.ok` manually
- Show the user something useful, not just "Error" — what failed and what they can do

---

## Accessibility & Responsiveness

**Responsive design**
- Mobile-first: write base styles for small screens, then add `min-width` media queries for bigger ones
- Use relative units (`rem`, `%`, `vw`) over fixed `px` where it makes sense
- Test with actual DevTools device toolbar, not just resizing the browser

**Accessibility (a11y)**
- Every image needs `alt` text (empty `alt=""` if purely decorative)
- Every form input needs a `<label>`, not just a placeholder
- Use semantic HTML (`<button>`, `<nav>`, `<main>`) before reaching for ARIA — native elements come with accessibility built in
- ARIA is for filling gaps, not a replacement for semantic HTML
- Keyboard nav matters — can I tab through the whole page and use it without a mouse?

**Web Storage API**
- `localStorage` — persists forever until cleared, readable by any JS on the page (XSS risk)
- `sessionStorage` — cleared when tab closes, same XSS exposure
- Cookies — can be `httpOnly` (JS can't read them at all), sent automatically with every request, good for auth tokens

---

## Charts & Forms

**Chart.js + react-chartjs-2**
- Register only the chart types/components I actually use (`Chart.register(BarElement, LineElement, ...)`) — keeps bundle smaller
- Wrap chart data in `useMemo` if it's derived from other state, so it doesn't recompute every render
- Charts need a sized parent container or they'll misbehave — wrap in a div with fixed height

**React Hook Form + Zod**
- RHF handles form state/validation triggering without re-rendering on every keystroke (uncontrolled by default — way faster than controlled forms for big forms)
- Zod defines the schema once, RHF + `@hookform/resolvers/zod` wires it in: `useForm({ resolver: zodResolver(schema) })`
- Inline errors come from `formState.errors.fieldName.message`
- Schema = single source of truth for both TypeScript types (`z.infer<typeof schema>`) and runtime validation

---

## Security & Auth

**JWT storage — localStorage vs cookies**
- localStorage: easy to use, but any XSS vulnerability = token theft
- httpOnly cookie: JS literally cannot read it, much safer against XSS, but vulnerable to CSRF instead (needs CSRF protection)
- My take: httpOnly cookie + SameSite=Strict is the safer default for anything real; localStorage is fine for quick POCs only

**CSRF**
- Happens when a malicious site tricks the browser into sending an authenticated request to your app (cookies auto-attach)
- Defense: CSRF tokens (server generates, client must send back) + `SameSite=Strict/Lax` cookies
- SameSite cookies alone block most CSRF in modern browsers, but defense-in-depth is safer

**Deploy (Vercel/Netlify)**
- Both auto-deploy on git push, give preview URLs per branch/PR
- Set env vars in their dashboard, not in the repo
- Vercel CLI: `vercel` for preview, `vercel --prod` for production

---

## Folder Structure & Testing

**Feature-based structure** (bulletproof-react style)
- Group by feature, not by file type — `features/auth/`, `features/dashboard/` instead of one giant `components/` folder
- Keeps related logic, components, hooks, and types together — easier to find and delete cleanly later

**Vitest + React Testing Library**
- Test behavior, not implementation — query by role/label text, not by CSS class or test ID where possible
- `render()` the component, `screen.getByRole(...)`, `userEvent` to simulate clicks/typing
- Three useful test types: pure utility function (easy, no rendering), form validation error display (render + trigger + assert error text), protected route redirect (mock auth state, assert `<Navigate>` fired)

---

## Gotchas / things that tripped me up
- New inline functions/objects as props break `React.memo` — pull them out or wrap in `useCallback`/`useMemo`
- `fetch` doesn't throw on HTTP errors, only network failures — always check `res.ok`
- Forgetting the `VITE_` prefix means the env var silently doesn't show up in `import.meta.env`
- localStorage tokens are XSS-readable — moved to cookie-based storage for the auth POC
