# Microfrontend Architecture — UI Principal Engineer Interview Prep
### ConnectWise Platform Context

[← Back to Index](../README.md)

---

## 1. The Core Problem Microfrontends Solve

Know your *why* before anything else. Interviewers at principal level expect this framing first.

```
Monolith frontend problems:
  - Team A's deploy blocks Team B — one pipeline, one release
  - Tech debt is shared — one framework upgrade breaks everyone
  - Feature flag conflicts, merge conflicts, slow CI
  - One team's bug can crash the entire product

Microfrontend solution:
  - Each team owns their app end-to-end (repo → CI → deploy → runtime)
  - Shell app composes them into one unified UX
  - Teams move independently
  - Platform team owns the shell + shared contracts
```

**ConnectWise context:** Multiple products (Manage, Automate, Control, etc.) from separate teams need to feel like one platform. The platform team's job is to make that composition seamless while keeping team autonomy intact.

---

## 2. The 4 Integration Strategies

Know all four. Recommend Module Federation. Explain why you ruled out the others.

```
Strategy 1 — Build-time (NPM packages)
  Team A publishes @cw/billing-ui as a versioned package
  Shell imports it like any dependency
  ❌ Everyone must rebuild + redeploy to pick up billing changes
  ❌ No independent deployment — defeats the purpose
  ✅ Simple, type-safe, tree-shakeable
  Use: shared LIBRARIES (design system, utils) — not whole apps

Strategy 2 — iframes
  Each app lives in its own iframe
  ✅ Complete isolation — no JS/CSS conflicts
  ❌ Hard to share state, auth, routing
  ❌ Poor UX — no cross-frame popups, broken accessibility
  Use: third-party embeds only

Strategy 3 — Web Components (Custom Elements)
  Each team exposes their app as a custom element
  Shell: <billing-app orgId="123"></billing-app>
  ✅ Framework-agnostic — React + Vue + Angular teams all work
  ❌ No standard for cross-component state sharing
  ❌ Shadow DOM CSS isolation blocks shared design tokens
  Use: when teams use DIFFERENT frameworks

Strategy 4 — Module Federation (Webpack 5)  ← RECOMMEND
  Runtime code sharing — shell downloads remote chunks at runtime
  ✅ True independent deployment (each team ships their own bundle)
  ✅ Shared dependencies (React loaded once, not per-MFE)
  ✅ Same framework — full React ecosystem works
  ✅ Dynamic loading — only pay for what the user accesses
```

---

## 3. Module Federation — Deep Dive

### Config Setup

```javascript
// Host (Shell App) — webpack.config.js
plugins: [
  new ModuleFederationPlugin({
    name: 'shell',
    remotes: {
      billing: 'billing@https://billing.cw.com/remoteEntry.js',
      manage:  'manage@https://manage.cw.com/remoteEntry.js',
    },
    shared: {
      react:     { singleton: true, requiredVersion: '^18.0.0' },
      'react-dom': { singleton: true, requiredVersion: '^18.0.0' },
      '@cw/design-system': { singleton: true },
    }
  })
]

// Remote (Billing MFE) — webpack.config.js
plugins: [
  new ModuleFederationPlugin({
    name: 'billing',
    filename: 'remoteEntry.js',         // entry point shell downloads
    exposes: {
      './App': './src/App',             // what shell can import
    },
    shared: {                           // same shared config as shell
      react:     { singleton: true, requiredVersion: '^18.0.0' },
      'react-dom': { singleton: true, requiredVersion: '^18.0.0' },
      '@cw/design-system': { singleton: true },
    }
  })
]
```

### Shell consumes a remote

```jsx
// shell/src/App.jsx
const BillingApp = React.lazy(() => import('billing/BillingApp'));

function Shell() {
  return (
    <Suspense fallback={<PlatformSpinner />}>
      <ErrorBoundary fallback={<AppCrashFallback appName="Billing" />}>
        <BillingApp />
      </ErrorBoundary>
    </Suspense>
  );
}
```

### Why singleton matters

```
singleton: true → only ONE instance of that package loads across shell + all remotes
                  highest compatible version wins

Without singleton: shell loads React 18.2, billing loads React 18.3
  → two React instances in the page
  → hooks break across boundaries
  → context doesn't cross MFE boundaries
  → very confusing bugs
```

---

## 4. The 3 Hardest Problems (Principal-Level)

### Problem 1 — Cross-MFE Communication

```
Option A: URL / query params
  billing fires: router.push('/manage?tab=invoices&orgId=123')
  manage reads:  const { orgId } = useSearchParams()
  ✅ Shareable links, back-button works, survives refresh
  ❌ Only for navigation-level events

Option B: Custom DOM Events (event bus)
  // Billing fires
  window.dispatchEvent(new CustomEvent('cw:org:changed', {
    detail: { orgId: 'org123' }
  }));

  // Any MFE listens
  window.addEventListener('cw:org:changed', (e) => setOrgId(e.detail.orgId));

  ✅ Fully decoupled — no shared import needed
  ✅ Works across framework boundaries
  ❌ No TypeScript, no replay, no structure enforcement
  Fix: define @cw/platform-events with typed event constants

Option C: Shared Zustand store
  const useGlobalStore = create(...); // in @cw/platform-state
  // Both shell and billing import from the same singleton package
  ✅ Full TypeScript, selectors, persistence
  ❌ Couples teams to a shared dependency version
  ❌ More complex to evolve without breaking remotes

RECOMMENDATION for ConnectWise:
  URL params      → navigation and deep-linking
  Custom events   → cross-MFE side effects (org switch, notifications, logout)
  Shared store    → session/auth ONLY (read-only from remotes)
```

### Problem 2 — Authentication

```javascript
// Shell handles auth ONCE — remotes trust it via shared store
const useAuthStore = create(() => ({ token: null, user: null }));

// shell/src/main.jsx — runs before any MFE loads
const token = await authService.getToken();  // OIDC / SSO
useAuthStore.setState({ token, user: jwtDecode(token) });
renderShell();

// Token refresh — shell handles silently, remotes never know
authService.onTokenRefresh((newToken) => {
  useAuthStore.setState({ token: newToken });
});

// Any MFE — reads token, never manages refresh
const { token } = useAuthStore();
axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
```

### Problem 3 — Version Skew During Deployment

```
Scenario: Shell on v1, Billing deploys v2 with breaking changes
User opens platform → shell loads Billing v2 → breaking change → crash

Solution 1: Pinned URL versioning
  billing@https://billing.cw.com/v1.2.3/remoteEntry.js  ← locked
  billing@https://billing.cw.com/latest/remoteEntry.js  ← always latest
  Platform team controls which version shell points to via a shell PR → controlled rollout

Solution 2: Manifest pattern ← BEST FOR SCALE
  // Shell fetches a manifest at startup
  GET https://platform.cw.com/manifest.json
  → { "billing": "https://billing.cw.com/v2.1.0/remoteEntry.js", "manage": "..." }

  // Update manifest → all users get new version on next page load
  // Rollback = revert the manifest → instant, no shell redeploy needed
  // Platform team promotes versions explicitly — no surprises

Solution 3: Contract testing (Pact)
  Each remote publishes its component contracts
  Shell CI runs Pact verification before accepting a new remote version
  Breaking contract = failed CI, not production crash
```

---

## 5. Shell App Architecture

```
Shell responsibilities:
  1. Global layout (nav, sidebar, header, notification tray)
  2. Authentication + silent token refresh
  3. Routing — which MFE loads for which URL
  4. MFE registry — manifest lookup, lazy loading
  5. Global error boundary per MFE
  6. Analytics bootstrap (inject tracking before any MFE loads)
  7. Feature flags (LaunchDarkly context setup)
  8. Accessibility (skip links, focus management across MFE transitions)

Routing:
  /billing/*   → BillingMFE   (lazy from manifest)
  /manage/*    → ManageMFE
  /automate/*  → AutomateMFE
  /            → Dashboard    (lives in shell — fastest initial load)
```

```jsx
// Shell router — loads from manifest at startup
const routes = await fetchManifest();

<Route path="/billing/*" element={
  <Suspense fallback={<AppSkeleton />}>
    <ErrorBoundary fallback={<MFECrashPage name="Billing" />}>
      <RemoteComponent
        url={routes.billing}
        scope="billing"
        module="./App"
      />
    </ErrorBoundary>
  </Suspense>
} />
```

### Cross-MFE navigation without full page reload

```javascript
// @cw/platform-events (shared package — typed)
export const navigate = (path: string) =>
  window.dispatchEvent(new CustomEvent('cw:navigate', { detail: { path } }));

// Shell listens and delegates to React Router
window.addEventListener('cw:navigate', (e) => router.push(e.detail.path));

// Any MFE — no React Router import, no coupling to shell
import { navigate } from '@cw/platform-events';
<button onClick={() => navigate('/manage/tickets/123')}>View Ticket</button>
// Shell intercepts → React Router pushes → Manage MFE loads → no full reload ✅
```

---

## 6. Dynamic Remote Loading (coding question)

Interviewers may ask you to implement this from scratch — loads a federated module from a URL at runtime:

```javascript
async function loadRemoteModule({ url, scope, module }) {
  // 1. Inject the remote's script tag if not already loaded
  if (!window[scope]) {
    await new Promise((resolve, reject) => {
      const script = document.createElement('script');
      script.src = url;
      script.onload = resolve;
      script.onerror = reject;
      document.head.appendChild(script);
    });
  }

  // 2. Initialize shared scope
  await __webpack_init_sharing__('default');
  const container = window[scope];
  await container.init(__webpack_share_scopes__.default);

  // 3. Get the module factory
  const factory = await container.get(module);
  return factory();
}

// Wrap for React.lazy
const BillingApp = React.lazy(() =>
  loadRemoteModule({
    url: manifest.billing,
    scope: 'billing',
    module: './App'
  }).then(m => ({ default: m.default }))
);
```

---

## 7. Platform-Specific React Patterns

### Lazy loading with preload on hover

```jsx
const ManageApp = React.lazy(() => import('./ManageApp'));
const preloadManage = () => import('./ManageApp'); // triggers download early

<NavItem
  onMouseEnter={preloadManage}       // preload on hover
  onClick={() => navigate('/manage')}
>
  Manage
</NavItem>
```

### MFELoader wrapper (reusable shell pattern)

```jsx
function MFELoader({ name, loader }) {
  const Component = React.lazy(loader);
  return (
    <ErrorBoundary
      fallback={({ error }) => (
        <div>
          <h2>{name} failed to load</h2>
          <p>{error.message}</p>
          <button onClick={() => window.location.reload()}>Retry</button>
        </div>
      )}
    >
      <Suspense fallback={<AppSkeleton />}>
        <Component />
      </Suspense>
    </ErrorBoundary>
  );
}

<MFELoader name="Billing" loader={() => import('billing/BillingApp')} />
```

### Context splitting at platform scale

```jsx
// ❌ One giant AppContext — any change hits every consumer across all MFEs
const AppContext = createContext();
// { user, org, theme, notifications, featureFlags, permissions }
// User clicks notification → EVERYTHING re-renders

// ✅ Split by update frequency
const AuthContext  = createContext(); // changes: login/logout only
const ThemeContext = createContext(); // changes: effectively never
const OrgContext   = createContext(); // changes: org switch
const NotifContext = createContext(); // changes: frequently

// Shell wraps all MFEs
function PlatformProviders({ children }) {
  return (
    <AuthContext.Provider value={auth}>
      <ThemeContext.Provider value={theme}>
        <OrgContext.Provider value={org}>
          <NotifContext.Provider value={notif}>
            {children}
          </NotifContext.Provider>
        </OrgContext.Provider>
      </ThemeContext.Provider>
    </AuthContext.Provider>
  );
}
```

### Shell-level notification portal

```jsx
// Notifications render in the shell's DOM, not inside any MFE
function NotificationHost() {
  const [notifs, setNotifs] = useState([]);

  useEffect(() => {
    const handler = (e) => {
      const id = Date.now();
      setNotifs(prev => [...prev, { id, ...e.detail }]);
      setTimeout(() => setNotifs(prev => prev.filter(n => n.id !== id)), 5000);
    };
    window.addEventListener('cw:notify', handler);
    return () => window.removeEventListener('cw:notify', handler);
  }, []);

  return ReactDOM.createPortal(
    <div className="notification-tray">
      {notifs.map(n => <Toast key={n.id} {...n} />)}
    </div>,
    document.getElementById('notification-root') // outside any MFE's subtree
  );
}

// Any MFE fires a notification — no import from shell needed
window.dispatchEvent(new CustomEvent('cw:notify', {
  detail: { type: 'success', message: 'Invoice saved' }
}));
```

### Feature flag hook

```javascript
const useFeatureFlag = (flagName: string, override?: boolean): boolean => {
  const flags = useFlagStore(state => state.flags);
  if (override !== undefined) return override; // test/storybook override
  return flags[flagName] ?? false;
};

// Usage
const showNewDashboard = useFeatureFlag('new-dashboard');
return showNewDashboard ? <NewDashboard /> : <OldDashboard />;
```

---

## 8. Interview Answers

### "How would you architect a microfrontend platform for ConnectWise?"

> "I'd use Webpack Module Federation. Each product team — Billing, Manage, Automate — owns a separate repo, CI pipeline, and deployment. They expose their app as a federated module at a versioned URL. The shell app, owned by the platform team, acts as the composition layer: it handles routing, authentication via a single OIDC flow, and loads each remote lazily inside an ErrorBoundary and Suspense. React and the design system are declared as singletons in the shared config so only one copy loads — critical because two React instances breaks hooks and context.
>
> For cross-app communication I use a typed event bus for side effects like org switches and notifications, and URL params for navigation. Auth token lives in a shared Zustand singleton the shell populates — remotes read it, never write it.
>
> For deployment safety I use a manifest pattern: shell fetches a JSON at startup listing each remote's pinned URL. Platform team controls promotions — billing's new version merges to the manifest and rolls out without touching the shell. Rollback is a manifest revert."

### "What's the hardest problem in microfrontend architecture?"

> "Version skew. Independent deployment is the whole point — but it creates a window where shell and a remote are on incompatible versions simultaneously. The manifest pattern mitigates this by giving the platform team explicit control over which version of each remote the shell loads. Contract testing with Pact adds a safety net: remotes publish their component contracts, and shell CI fails if a new remote version breaks the contract. Between the two, you get independent deployments without surprise production crashes."

### "How do you handle shared state across microfrontends?"

> "It depends on the use case. Navigation events go through URL params — deep-linkable and back-button compatible. Cross-app side effects like org switches and notifications use a typed custom event bus — fully decoupled, no shared imports. Auth is the only thing that goes in a shared Zustand singleton, and remotes treat it as read-only. I avoid a single shared app store across all MFEs because it couples teams to a shared version and makes independent evolution harder."
