# EventFlow — Client

The Next.js 16 frontend for the EventFlow platform. Communicates exclusively with the **API Gateway** at port 3000.

---

## Prerequisites

- [Node.js](https://nodejs.org/) v20+
- [pnpm](https://pnpm.io/installation) v9+
- The API Gateway must be running at `http://localhost:3000` before you start the client.

---

## Step 1 — Install Dependencies

From the `client/` directory:

```bash
pnpm install
```

If you see `ERR_PNPM_IGNORED_BUILDS`, run:

```bash
pnpm approve-builds
```

Select both `sharp` and `unrs-resolver` when prompted (these are Next.js's image optimizer and module resolver — both safe), then re-run `pnpm install`.

---

## Step 2 — Configure Environment Variables

Create a `.env.local` file in the `client/` directory:

```bash
cat > .env.local << 'EOF'
NEXT_PUBLIC_API_BASE_URL=http://localhost:4000
EOF
```

If `NEXT_PUBLIC_API_BASE_URL` is not set, the client defaults to `http://localhost:3000` automatically.

---

## Step 3 — Start the Dev Server

```bash
pnpm dev
```

The app runs at [http://localhost:4000](http://localhost:4000).

---

## Available Scripts

```bash
pnpm dev      # Start dev server on port 4000 with hot reload
pnpm build    # Production build
pnpm start    # Start production server (run pnpm build first)
pnpm lint     # ESLint check
```

---

## App Structure

```
src/
├── app/                  # Next.js App Router pages
│   ├── (auth)/           # Auth route group (login, register)
│   └── page.tsx          # Home page
├── components/
│   ├── providers.tsx     # TanStack Query + theme providers
│   └── ui/               # shadcn/ui components
├── features/
│   └── auth/             # Auth feature (API calls, forms, hooks)
├── lib/
│   ├── api-client.ts     # Axios instance (attaches JWT, unwraps response)
│   └── query-client.ts   # TanStack Query client config
├── stores/
│   ├── auth-store.ts     # Zustand auth state (token, user)
│   └── ui-store.ts       # Zustand UI state
└── types/                # Shared TypeScript types
```

---

## How API Calls Work

All requests go through `src/lib/api-client.ts`:

- **Base URL:** `NEXT_PUBLIC_API_BASE_URL` (defaults to `http://localhost:3000`)
- **Auth:** JWT token is read from `localStorage` (`accessToken`) and attached as a `Bearer` header automatically on every request.
- **Response unwrapping:** The interceptor unwraps the API Gateway's `{ success, data, meta }` envelope — feature code receives `data` directly.
- **Error handling:** API error messages from the `{ error: { code, message } }` shape are extracted and attached to the thrown error object.

---

## Troubleshooting

**Blank page or API errors on load**
Make sure the API Gateway is running: `pnpm start:dev api-gateway` from the project root.

**Login/register returns network error**
Check that `NEXT_PUBLIC_API_BASE_URL` in `.env.local` matches where the API Gateway is listening. Restart the dev server after changing env vars.

**Port 4000 already in use**
```bash
lsof -i :4000
kill -9 <PID>
```
