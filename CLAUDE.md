# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Common Commands

### Development
```bash
bun start                  # Start both frontend and backend (runs dev:frontend and dev:backend in parallel)
bun dev                    # Same as bun start
bun dev:frontend           # Start Vite dev server only (port 5173)
bun dev:backend            # Start Convex backend only
```

### Build & Quality
```bash
bun run build              # Type check and build for production
bun typecheck              # Check types across all TypeScript configs (app, node, convex)
bun lint                   # Run ESLint (excludes convex/_generated)
bun preview                # Preview production build locally
```

### Convex Setup
```bash
bunx convex dev --configure=new --once   # Initialize new Convex project
bunx @convex-dev/auth                    # Set up Convex Auth
bunx convex env set KEY value            # Set environment variable
bunx convex env get KEY                  # Get environment variable
```

### Stripe Development
```bash
# Required for testing Stripe webhooks locally
stripe listen --forward-to $(bunx convex env get CONVEX_SITE_URL)/stripe/webhook
```

## Architecture Overview

### Project Structure

- **src/** - React frontend with TanStack Router
  - **routes/** - File-based routing (see routing patterns below)
  - **ui/** - Reusable UI components (shadcn/ui based)
  - **utils/** - Utility functions and helpers
- **convex/** - Convex serverless backend
  - **schema.ts** - Database schema definitions
  - **auth.ts** - Authentication configuration
  - **stripe.ts** - Stripe integration logic
  - **http.ts** - HTTP routes and webhooks
  - **init.ts** - Database seeding (runs on first deployment)
  - **otp/** - Custom OTP email provider
- **public/** - Static assets including translation files (public/locales/)

### Path Aliases
- `@/` → src/
- `@cvx/` → convex/
- `~/` → root/

### Frontend-Backend Integration

The application uses a three-layer data flow:

```
React Components ↔ TanStack Query + ConvexQueryClient ↔ Convex Backend
```

**Key Pattern:**
```typescript
// Queries (reactive, real-time)
const { data } = useQuery(convexQuery(api.app.getCurrentUser, {}));

// Mutations
const { mutateAsync } = useMutation({
  mutationFn: useConvexAction(api.stripe.createSubscriptionCheckout),
});
```

**Provider Hierarchy** (set up in src/app.tsx):
```
ConvexAuthProvider
  └── QueryClientProvider (TanStack Query with Convex integration)
      └── RouterProvider (TanStack Router)
```

### TanStack Router Patterns

**File-Based Routing:**
- `_layout.tsx` files create shared layouts with `<Outlet />`
- `_prefix` directories organize routes without affecting URLs
- Route files like `_layout.index.tsx` render at the layout's base path

**Example Structure:**
```
routes/
├── __root.tsx                           # Root layout
├── _app.tsx                             # App-wide data preloading
└── _app/_auth/                          # Protected routes
    └── dashboard/
        ├── _layout.tsx                  # Dashboard layout with nav
        ├── _layout.index.tsx            # /dashboard
        └── _layout.settings.billing.tsx # /dashboard/settings/billing
```

**Authentication Guard:**
The `_auth.tsx` route protects all nested routes by checking authentication state and redirecting to /login if needed.

**Data Preloading:**
Use `beforeLoad` in route definitions to preload data before rendering:
```typescript
export const Route = createFileRoute("/_app")({
  beforeLoad: async ({ context }) => {
    await context.queryClient.ensureQueryData(
      convexQuery(api.app.getCurrentUser, {})
    );
  },
});
```

### Authentication

**Backend (convex/auth.ts):**
- Uses `@convex-dev/auth` with custom ResendOTP provider
- Supports OAuth (GitHub) and email OTP (8-digit codes, 20-min expiration)
- Database tables: `authAccounts`, `authSessions`, `authVerificationCodes`, `users`

**Frontend:**
- Two-step login flow: email entry → code verification
- Uses `useConvexAuth()` for auth state, `useAuthActions()` for sign in/out
- Protected routes automatically redirect to /login when unauthenticated

**Custom Sign Out:**
Always use the `useSignOut()` hook from src/utils/misc.ts to ensure proper cleanup:
```typescript
const signOut = useSignOut();
await signOut(); // Signs out, invalidates queries, redirects to login
```

### Stripe Integration

**Naming Convention in convex/stripe.ts:**
- `PREAUTH_*` - Internal functions requiring pre-authorized userId
- `UNAUTH_*` - Internal functions not requiring authentication
- Regular exports - Public actions/queries

**Subscription Flow:**
1. User clicks subscribe → `createSubscriptionCheckout` action
2. Action creates Stripe checkout session, returns URL
3. User completes payment in Stripe
4. Webhook triggers `handleCheckoutSessionCompleted`
5. `PREAUTH_replaceSubscription` updates database

**Webhook Events** (convex/http.ts):
- `checkout.session.completed` - New subscription created
- `customer.subscription.updated` - Subscription changed
- `customer.subscription.deleted` - Subscription canceled

**Database Tables:**
- `plans` - Product definitions with multi-currency pricing
- `subscriptions` - User subscription records
- `users.customerId` - Links to Stripe customer ID

**Initialization:**
- `bun run predev` runs convex/init.ts before dev starts
- Creates Stripe products, prices, and configures Customer Portal
- Seeds database with plan definitions (Free, Pro)

### Type Safety

**Convex Generated Types:**
Convex auto-generates types from schema in `convex/_generated/`. Import from:
```typescript
import { api } from "@cvx/_generated/api";
import { Id, Doc } from "@cvx/_generated/dataModel";
```

**Extended User Type** (types.ts):
```typescript
export type User = Doc<"users"> & {
  avatarUrl?: string;
  subscription?: Doc<"subscriptions"> & { planKey: PlanKey };
};
```

### Form Handling

Uses TanStack Form with Zod validation:
```typescript
const form = useForm({
  validatorAdapter: zodValidator(),
  defaultValues: { email: "" },
  onSubmit: async ({ value }) => { /* ... */ },
});
```

### File Uploads

Pattern for file uploads with Convex storage:
```typescript
// 1. Generate upload URL
const uploadUrl = await generateUploadUrl();
// 2. Upload file
const result = await fetch(uploadUrl, { method: "POST", body: file });
const { storageId } = await result.json();
// 3. Update record with storage ID
await updateUserImage({ imageId: storageId });
```

### Internationalization

- Configured in src/i18n.ts using i18next
- Supports English and Spanish
- Translation files in public/locales/{en,es}/translation.json
- Language switcher component in UI

### UI Components

- Based on shadcn/ui (Radix UI primitives + Tailwind CSS)
- Use `cn()` utility from src/utils/misc.ts for conditional classes
- Class variance authority for component variants

### Error Handling

Centralized error constants in errors.ts:
```typescript
import { ERRORS } from "~/errors";
throw new Error(ERRORS.AUTH_EMAIL_NOT_SENT);
```

### Environment Variables

All environment variables are centralized in convex/env.ts. Required variables:
- `AUTH_RESEND_KEY` - Resend API key for emails
- `STRIPE_SECRET_KEY` - Stripe secret key
- `STRIPE_WEBHOOK_SECRET` - Stripe webhook signing secret

Set using: `npx convex env set KEY value`

## Important Patterns

1. **Scheduled Functions:** Use Convex scheduler for async operations:
   ```typescript
   await ctx.scheduler.runAfter(0, internal.stripe.PREAUTH_createStripeCustomer, {
     userId, currency
   });
   ```

2. **Navigation:** Use type-safe navigation with route imports:
   ```typescript
   import { Route as DashboardRoute } from "@/routes/_app/_auth/dashboard/_layout.index";
   navigate({ to: DashboardRoute.fullPath });
   ```

3. **Database Queries:** Always use Convex's reactive queries via TanStack Query for real-time updates.

4. **Convex Functions:** Export queries, mutations, actions, and http functions from convex/ directory. Internal functions use `internal.` prefix.
