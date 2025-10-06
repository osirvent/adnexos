# Astro + Svelte + PocketBase Frontend Notes

Use this guide when adapting the frontend stack to another PocketBase deployment.

## Build system
- `astro.config.ts` wires Astro with the Svelte integration and the Vite PWA plugin. TailwindCSS is injected as a Vite
  plugin, and a compile-time `__APP_VERSION__` constant is derived from `process.env.VERSION`. 【F:frontend/web/astro.config.ts†L1-L31】
- Svelte components use `vitePreprocess` so they can share the same tooling pipeline as Astro. 【F:frontend/web/svelte.config.js†L1-L5】

## PocketBase client integration
- `src/lib/pb.ts` exports a single PocketBase instance (auto-cancellation disabled) and wraps it in Svelte stores for
  reactive auth state. Import the `pb` writable when you need direct SDK access, and subscribe to the `auth` readable to
  react to login changes across routes and components. 【F:frontend/web/src/lib/pb.ts†L1-L14】
- Shared state such as the current settings, group, or expense lives in `src/lib/stores.ts`, which derives values from the
  PocketBase auth expansion data. Keep new shared stores colocated here to avoid duplicated fetch logic. 【F:frontend/web/src/lib/stores.ts†L1-L13】

## Routing and rendering flow
- Astro `.astro` pages define the routing shell and compose Svelte components for interactive areas. For example,
  `src/pages/index.astro` renders the marketing layout while the authenticated sections under `src/pages/groups` and
  `src/pages/expenses` delegate to Svelte components for data-bound UI. 【F:frontend/web/src/pages/index.astro†L1-L70】
- Cross-page layouts live in `src/layouts`, while reusable widgets reside in `src/components`. These components expect
  PocketBase record models (types imported from the SDK) and use the stores listed above for reactive data.

## Progressive Web App features
- `src/manifest.ts` defines install metadata (name, theme, icons) that is consumed by the Vite PWA plugin. Adjust the
  manifest and the static assets in `public/` when branding changes. 【F:frontend/web/src/manifest.ts†L1-L33】
- The PWA plugin is configured to register on demand (`registerType: 'prompt'`) and to ignore URL parameters during
  precaching, while excluding API and special routes from navigation fallback. Update `astro.config.ts` if the backend API
  paths change. 【F:frontend/web/astro.config.ts†L14-L31】

## Data fetching conventions
- Most components use PocketBase collection methods (e.g., `pb.collection('groups')...`) either directly or via loaders in
  Astro endpoints. Keep `fields=` query parameters consistent with the backend hooks because derived data such as
  `balance`, `costs`, or `membersBalance` is only populated when requested. 【F:backend/internal/plugin/groups.go†L150-L221】

## Local development
- During development the PocketBase client automatically targets `http://127.0.0.1:8090`; production builds fall back to
  relative requests. Ensure your deployment proxies the PocketBase API at the root path or update the base URL. 【F:frontend/web/src/lib/pb.ts†L4-L6】
