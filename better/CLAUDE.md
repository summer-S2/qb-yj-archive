/# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Common Development Commands

### Package Management

- Package manager: **pnpm 8.3.1+** (required, not npm/yarn)
- Install dependencies: `pnpm install`

### Development Servers

- Default (local): `pnpm start` (uses `.env`)
- Local test: `pnpm start:local`
- Development: `pnpm start:dev`
- Production: `pnpm start:prod`
- Internal prod: `pnpm start:internal:prod`
- Other variants: `pnpm start:srv`, `pnpm start:pfms`, `pnpm start:v3`

Dev server runs on `http://0.0.0.0:8000`

### Testing

- Run all tests: `pnpm test`
- Run single test: `pnpm test src/path/to/__test__/file.test.ts`
- Watch mode: `pnpm test:watch`
- E2E tests: `pnpm e2e open` (opens Cypress UI)

### Building

- Development: `pnpm build:dev`
- Production: `pnpm build:prod` (includes TypeScript check)
- Internal production: `pnpm build:internal:prod`
- Preview build: `pnpm build:preview` (serves static build on port 4173)
- Analyze bundle: `pnpm build:vis` (generates visualizer report)

Build output: `webApp/www/`

### Code Quality

- Lint: `pnpm lint`
- Lint & fix: `pnpm lint:fix`
- Unused code detection: `pnpm knip`

### Design Tokens

- Build tokens: `pnpm tokens:build`
- Watch tokens: `pnpm tokens:watch`

### Storybook

- Dev: `pnpm storybook` (port 6006)
- Build: `pnpm storybook-build`

### SVG Generation

- Generate React components from SVG: `pnpm svgr`

## High-Level Architecture

### Technology Stack

- **Build Tool**: Vite 5 with SWC for fast compilation
- **React**: 18.3.1 (enforced via pnpm overrides)
- **State Management**: Redux Toolkit with redux-persist
- **Server State**: React Query 3.39 with persistence
- **Routing**: React Router v6
- **UI Framework**: Material-UI v5.14
- **Styling**: MUI + Emotion + SCSS + Tailwind CSS (newer components)
- **Forms**: React Hook Form with Yup validation
- **Testing**: Vitest + React Testing Library + Cypress (E2E)

### Project Structure

```
src/
├── api/              # API client layer with React Query hooks
├── components/       # Reusable UI components
│   ├── better-ui/   # Custom design system components
│   ├── tailwindcss/ # Newer Tailwind-based components
│   ├── dialogs/     # Modal/dialog components
│   └── drawers/     # Drawer components
├── contexts/         # React Context providers
├── features/         # Feature-specific logic & components
├── guards/           # Auth route guards (AuthGuard, GuestGuard)
├── hooks/            # Custom React hooks (30+ utilities)
├── pages/            # Page-level components tied to routes
├── redux/            # Redux slices & store configuration
│   ├── slices/      # Domain-specific state slices
│   └── selectors/   # Redux selectors
├── routes/           # React Router configuration
│   ├── paths.ts     # Centralized path constants
│   └── */           # Domain-based route modules
├── theme/            # MUI theme configuration
├── types/            # TypeScript type definitions
└── utils/            # Utility functions & helpers
    └── native/      # React Native WebView bridge
```

### State Management Philosophy

**Redux (Global UI State)**:

- Persistent: auth, app settings, mydata, fund, notifications, events
- Session-only: SMS verification, some deep link tracking
- Non-persistent: dialogs, drawers, loaders (ephemeral UI state)
- All slices in `redux/slices/`, accessed via typed hooks

**React Query (Server State)**:

- All API data fetching with automatic caching
- Persistent via localStorage (5-day max age)
- Query keys centralized in `constants/query-key.ts`
- Mutation patterns with `useMutation` hooks
- Special behavior: Detects device errors and updates Redux

**Separation Pattern**:

- Redux = Client state (UI, preferences, session)
- React Query = Server state (API responses, remote data)
- Context = Theme settings, authentication, secure keypad

### Routing Architecture

**Route Organization**: Domain-based modules in `routes/`

- `authRoutes.tsx`: Login, signup, PIN, FIDO flows
- `dashboardRoutes.tsx`: Protected dashboard routes
- `menuRoutes.tsx`: Settings & user information
- `mydataRoutes.tsx`: Asset connection & MyData flows
- `commonRoutes.tsx`: Public routes

**Route Features**:

- Lazy loading: All components wrapped with `React.lazy()` + `Loadable` wrapper
- Auth guards: `AuthGuard` redirects unauthenticated users
- Route animations: Framer Motion via `AnimatePresence`
- Path constants: Centralized in `routes/paths.ts` (never hardcode paths)

**Guard Logic**:

- Check authentication state from JWT context
- Validate onboarding completion
- Check FIDO status for biometric flows
- Redirect based on user state

### API Integration Pattern

**Structure**: Domain-organized API modules in `api/`

```
api/
├── auth.ts           # Auth endpoints + hooks
├── mydata.ts         # MyData/asset connection
├── better/           # Main app domain APIs
│   ├── main-assets.ts
│   ├── collection.ts
│   └── account-detail.ts
├── fa/               # Financial advisor APIs
└── fund.ts           # Fund operations
```

**API Pattern**:

```typescript
// Raw axios functions
export async function signIn(input: SignInInput): Promise<Login>;

// React Query hooks for queries
export function useGetMe<TData = User>(options?: UseQueryOptions);

// React Query hooks for mutations
export function useSignInMutation();
```

**API Configuration**:

- Multiple API gateways configured via env vars:
  - `VITE_API_GATEWAY_HOST` - Main API v1
  - `VITE_ASSET_API_GATEWAY_URL` - Asset service
  - `VITE_FA_API_GATEWAY_HOST` - Financial advisor API
- Proxy configuration in `vite.config.ts` for keypad endpoints

### Authentication Flow

**Auth Context** (`contexts/JWTContext.tsx`):

- Provides: `isAuthenticated`, `isInitialized`, `user`, `login()`, `logout()`
- JWT tokens stored in Redux + localStorage via `setSession()`
- Token validation with `jwt-decode` library

**Auth Methods**:

1. **PIN-based**: Traditional 6-digit PIN with server-side encryption
2. **FIDO2**: Biometric (fingerprint/face ID) via native bridge
3. **SMS**: Phone number verification for signup/recovery
4. **V-Keypad**: Virtual keyboard for secure input (toggleable)

**Device Management**:

- Single device policy enforced
- Device ID validation on login
- Unregistered device → force re-verification flow
- Device errors set `hasSingleDeviceError` Redux state

### Native Bridge Communication (WebView)

**Purpose**: Bidirectional communication between React web app and React Native container

**Location**: `utils/native/`

- `main.ts`: Production implementation
- `main.dev.ts`: Development mock (browser testing)
- `alliance.ts`: Alliance app variant
- Factory pattern exports correct implementation

**Communication Pattern**:

```typescript
// Web → Native (Promise-based)
sendMessage(command: COMMAND_TYPES, data: any, onSuccess?, onError?)

// Native → Web (via window message events)
window.addEventListener('message', (event) => {
  const msg: IReactNativeReceiveMessage = JSON.parse(event.data)
  // Execute callback or dispatch Redux action
})
```

**Key Native Commands**:

- User management: `askUserInfo()`, `askFidoUse()`
- App control: `askAppInfo()`, `sendQuit()`, `executeApp()`
- File operations: `askFileDownload()`, `askSavePdf()`
- Certificate handling: `askCertImportCode()`, `askSignMydata()`
- UI control: `sendBgColorStatusBar()`, `offerToggleKeyboard()`
- Push notifications: `askPushNotiData()`, `askOpenNotiSettings()`
- Biometrics: `askFido()`, `askFidoType()`
- Analytics: Airbridge tracking methods

**Message Queue**:

- Redux slice `app.messageQueue` stores async native command results
- Components subscribe to queue for native-initiated actions

### Environment Configuration

**Multiple Environments**: Separate `.env.*` files for each target

- `.env` - Default local
- `.env.development` - Dev server
- `.env.production` - Production
- `.env.internal.production` - Internal production
- `.env.srv`, `.env.pfms`, `.env.v3` - Other variants

**Key Environment Variables**:

- API endpoints: `VITE_API_GATEWAY_HOST`, `VITE_ASSET_API_GATEWAY_URL`
- Feature flags: `VITE_SHOW_SETTINGS_MENU`, `VITE_VKEYPAD`
- Debug tools: `VITE_ERUDA_CONSOLE`, `VITE_API_DEBUG_DIALOG`
- App variant: `VITE_ORIGIN_NAME` (better/v3/kinfa/finelab)
- Analytics: `VITE_GA_MEASUREMENT_ID`, `VITE_GOOGLE_TAG`
- Error tracking: `VITE_SENTRY_DSN`
- MyData: `VITE_MYDATA_TEST_BED`

**Runtime Detection** (`config.ts`):

- `IS_DEV` - Development mode
- `IS_WEBVIEW` - Running in React Native WebView vs browser
- `WHICH_WEBVIEW` - 'better' or 'alliance' app
- `ORIGIN_NAME` - App variant from env

### Form Handling

**React Hook Form Integration**:

- Centralized form components in `components/Forms/`
- Yup schema validation with `@hookform/resolvers`
- Controlled inputs via `Controller` component
- DevTools available via `@hookform/devtools`

**Secure Input**:

- `KeypadContext` for PIN/password entry
- V-Keypad option for additional security
- Native bridge integration for secure input capture

### Component Conventions

**Naming**:

- Components/Pages: `PascalCase.tsx`
- Other files: `camelCase.ts`
- Boolean variables: `isXXXing`, `hasXXXed`, `shouldXXXX`
- Avoid `snake_case.ts`

**Component Layers**:

1. **Pages** - Route-level components in `/pages/`
2. **Features** - Feature-specific in `/features/`
3. **Components** - Reusable in `/components/`

**Styling Approaches** (mixed):

- Material-UI components with theme overrides
- Emotion CSS-in-JS
- SCSS for complex styles
- Tailwind CSS for newer components (migration in progress)

**Lazy Loading**:

- All routed components use `React.lazy()`
- Wrapped with `Loadable` component for loading states
- Code splitting automatic via Vite

### Error Handling & Monitoring

**Sentry Integration**:

- Production error tracking
- User context attached
- Custom error capturing for critical flows
- Access token expiration monitoring

**Error Boundaries**:

- Top-level `ErrorBoundary` component
- React Query error boundaries for data fetching
- Custom fallback UI for errors

**Logging Strategy**:

- Factory pattern: `ProductionLogger` vs `DevelopmentLogger`
- Color-coded console output in dev
- Conditional logging based on `VITE_API_LOGGER_MODE`

### Performance Optimizations

**Code Splitting**:

- Route-based splitting via `React.lazy()`
- Manual vendor chunks in `vite.config.ts`
- Separate bundles for: react-core, charts, pdf libraries

**Caching**:

- React Query with 5-day localStorage persistence
- Redux persist for client state
- Asset image lazy loading with intersection observer

**Build Optimizations**:

- SWC for fast TypeScript compilation
- Terser minification (console/debugger not removed)
- Bundle analysis via `pnpm build:vis`

### Multi-Language Support (i18n)

**Setup**: i18next with react-i18next

- Language files in `src/locales/` (ko.json, en.json)
- Korean as primary locale
- Browser language detection via `i18next-browser-languagedetector`

**Usage**:

```tsx
import { useTranslation } from 'react-i18next';
const { t } = useTranslation();
<Text>{t('common.ok')}</Text>;
```

### Analytics & Tracking

**Google Analytics 4**:

- Setup via `react-ga4`
- Page view tracking on route change
- Custom event tracking

**Airbridge**:

- Mobile attribution tracking
- Native bridge integration
- Event tracking methods on `nativeUtil.airbridge`

**Sentry**:

- Error tracking
- Performance monitoring
- User session replay

### Git Workflow

**Branch Strategy**:

- Main branch: `develop` (not `main`)
- Feature branches: `feature/#issue-description`
- Release branches: `release/v1.0.YYYYMMDD`
- Hotfix branches: `hotfix/#issue-description`

**Commit Convention**:

```
[team-name] #issue-number message
```

Examples:

- `[better] #244 added new route`
- `[pfms] #123 fixed auth bug`

**Pre-commit Hooks** (Husky):

- Runs lint-staged
- Auto-formats with Prettier
- Runs ESLint --fix
- Configured in `.husky/` directory

**Pull Request Requirements**:

- Follow template in `.github/PULL_REQUEST_TEMPLATE.md`
- Link related issue
- Specify PR type (Feature/Bug Fix/Refactor/etc)
- Add QA instructions
- Note if Cypress tests added

### Deployment

**Jenkins CI/CD**:

- Development: `https://dev-ci.betterday.finset.io/`
- Production: `https://jenkins.betterday.co.kr/`

**Multiple Deployment Targets**:

- Better (main product)
  - Prod: `https://app.betterday.co.kr/`
  - Dev: `https://betterday.finset.io/`
- V3 variant: `https://v3.betterday.co.kr`
- Kinfa variant: `https://kinfa.betterday.co.kr`
- Finelab variant: `https://finelab.betterday.finset.io` (dev only)

### Special Development Features

**Debug Tools**:

- Eruda console: Enable with `VITE_ERUDA_CONSOLE=Y`
- API debug dialog: Enable with `VITE_API_DEBUG_DIALOG=true`
- React Query DevTools: Auto-enabled in dev mode

**Test User Credentials** (from README):

- Name: 홍길동
- ID Number: 000101-3
- Mobile: 010-1111-3333
- SMS code: Any 6 digits
- PIN: 112233

**Storybook**:

- Component development environment
- Run with `pnpm storybook`
- Stories in `.stories.tsx` files

### Critical Development Notes

1. **TypeScript Configuration**:

   - `noImplicitAny: false` - implicit any allowed
   - `strictNullChecks: false` - null checks relaxed
   - Path alias `*` → `src/*` (no `@/` imports)

2. **Vite Proxy**:

   - Keypad endpoints (`/servlets`, `/decrypt`) proxied to production
   - Allows local development with secure input

3. **Asset Connection Flow**:

   - Complex state machine for MyData integration
   - Progress tracked in `assetConnection` Redux slice
   - Lock status polling during asset sync
   - Context fetch triggers data refresh

4. **FA (Financial Advisor) Integration**:

   - Separate API gateway
   - Asset sync to FA system via `postCustomerAssetsToFa()`
   - Progress indicators during sync
   - Error handling for sync failures

5. **Auto-Logout**:

   - 10-minute background timer (`AUTO_LOGOUT_TIME_MIN`)
   - Tracked via `lastActiveTime` in Redux
   - Triggers on app returning to foreground
   - Implemented in `App.tsx`

6. **Single Device Policy**:

   - Device ID validation on every auth request
   - `hasSingleDeviceError` Redux state for violations
   - Forces re-verification flow
   - Error handled in React Query defaults

7. **Toast Notifications**:

   - Custom `showToast()` function in `utils/toast`
   - Uses `react-native-root-toast` for Better app
   - Uses `notistack` for other variants
   - Line breaks with `\n`

8. **Dynamic Links & Deep Links**:

   - Handled via `useDynamicLink` hook
   - Redux state: `dynamicLink`, `initialDynamicLink`
   - Push notification links via `usePushLinkHandler`
   - Support for various link types (investment info, FA linking, etc)

9. **Bundle Optimization Strategy**:

   - Critical libs (axios, lodash, date-fns) in main bundle
   - React, router, animations in separate vendor chunks
   - PDF libraries isolated to reduce main bundle size
   - Other node_modules auto-chunked individually

10. **Context Fetch Pattern**:
    - `shouldContextFetch` flag triggers data refresh
    - Waits for asset lock release
    - Invalidates React Query cache
    - Refetches `CACHED` queries
    - Resets progress indicators
    - Used after asset connection/sync

### Common Gotchas

- **Never use npm/yarn** - Only pnpm is supported
- **Path imports**: Use `import X from 'components/X'` not `@/components/X`
- **No hardcoded paths** - Always use `routes/paths.ts` constants
- **Query keys** - Use centralized keys from `constants/query-key.ts`
- **Redux vs React Query** - Redux for UI state, React Query for server state
- **Native methods** - Always check if running in WebView before using `nativeUtil`
- **Material-UI version** - Locked to 5.14.x, don't upgrade without testing
- **React version** - Enforced at 18.3.1 via pnpm overrides
- **Test files** - Must be in `__test__/*.test.ts` directories
- **Line breaks in translations** - Korean locale is primary, not English
