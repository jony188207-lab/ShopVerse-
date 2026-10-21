# ShopVerse: final integration package

Three parts, one Firebase project (**`shopverse21-831fc`**):

| Folder | What | Hosting |
|---|---|---|
| `customer-app/` | Customer + seller marketplace (Next.js 14), Firestore rules/indexes, Cloud Functions source | Vercel (Root Directory `customer-app`) |
| `admin-panel/` | Admin Panel (Next.js 14): dashboard, sellers, users, products, banners, **Adsterra settings**, categories, orders, withdrawals, reports, settings, logs | Vercel (Root Directory `admin-panel`) |
| `cloudflare-worker/` | Free seller-approval API (replaces the Blaze-only Cloud Function) | Cloudflare Workers, free plan |

**Not deployed.** Nothing in this package has been deployed or run against your live Firebase, Vercel, Cloudflare or Adsterra accounts, so the site is **not live** until you complete the steps below.

## What was built for this request
1. **Top banners:** Admin Panel → Banners: image (https link), title, subtitle, button text, target product (the banner's link), priority, schedule, on/off, delete. The storefront home page (`TopBannerCarousel`) reads the same `productBanners` collection and shows only live banners whose product is approved and in stock. Link = product page: the Firestore rules require a banner to point at an approved product, and I did not loosen that.
2. **Bottom Adsterra banner:** Admin Panel → **Adsterra settings**: paste, change, save, switch on/off, device, size, schedule. Stored in `adNetworkSettings/adsterra-bottom`. Rules: only a **super admin** can create or change the code; any admin can switch it off or on. The storefront already renders the code **only inside a sandboxed iframe** (no `allow-same-origin`), never as page text; the admin page never displays stored code in lists, never writes it to the audit log, and previews it only on request, in the same sandbox. The global "ads enabled" switch is in Settings (super admin).
3. **Seller approval without Blaze:** `cloudflare-worker/` (see its README). The admin panel uses it when `NEXT_PUBLIC_SELLER_API_URL` is set. Only `admin` / `super_admin` (verified on the server) can approve; approval needs the applicant's verified email; Firestore rules, roles, stores and existing sellers are untouched. **D1 is not used**: Firestore remains the single source of truth.
4. **Customer app:** unchanged apart from the earlier Node 22 runtime fix (Node 20 is decommissioned for deploys on 2026-10-30) and a clearer Auth-lookup error in the Cloud Function. Firebase login and the marketplace are intact.

## Verified (actually run in the authoring environment)
| Check | Result |
|---|---|
| Worker: 29 tests (`cloudflare-worker`, `npm test`, Node's built-in runner) against an in-memory **simulation** of Google token endpoints, Identity Toolkit and Firestore REST, with real RSA signing | **29 pass.** Six deliberate bugs injected into the Worker (non-admin allowed, email check removed, audience check removed, signature check removed, pending check removed, CORS check removed) were each caught after one extra test was added |
| Admin panel: 23 pure-logic tests (permissions, formatting, errors, Adsterra validation, URL handling) | **23 pass** on a hand-made stand-in for vitest (not real vitest) |
| Admin panel: strict TypeScript compile of our own code against **stub** library types | no errors in our code (finds our mistakes, not mismatches with the real React/Next/Firebase types). It found and fixed one real error earlier (`Badge`) |
| Import/export scan of all pages and components (admin panel and customer app) | 0 problems |
| Customer app: `npm run validate:all` (config, rules structure audit, index coverage for 24 client queries) | all pass |
| Secrets scan (no private keys, service-account files or old project id) | clean |

## NOT run (this environment cannot reach the npm registry)
`npm install`, `tsc`/`npm run typecheck`, `npm run lint`, `npm run build` (production build) and real `vitest` for all three parts; the Firestore rules emulator tests; any test against live Firebase, Cloudflare or Adsterra. **I did not run a production build, so I cannot claim it passes.** Vercel's first build log is the real test: send me its last lines if it fails.
Adsterra code inside the sandbox is untested against Adsterra's real servers.

## What you must do to go live (nothing here can be done for you)
1. `docs/FIREBASE_CONFIGURATION.md`: enable Email/Password, create Firestore, publish `customer-app/firestore.rules`, deploy indexes (computer or Codespace; free), create `roles/<your UID>` with `role = super_admin`, add both domains to Authorized domains.
2. Two Vercel projects (Root Directory `customer-app` and `admin-panel`) with the environment variables in `.env.example`.
3. `cloudflare-worker/README.md`: service account (two roles), Worker with the secret, then `NEXT_PUBLIC_SELLER_API_URL` in the admin project and redeploy.
4. Smoke test (below). Phone-friendly Bangla steps: `docs/QUICKSTART_BN.md`. Phone uploads lose folders: use `docs/github-unzip-workflow.yml`.

## Smoke test after deployment (about 10 minutes)
1. Admin login → Dashboard shows counts (a "Cannot verify your role" message means rules/roles are not set up).
2. Banners → create one for an approved product → storefront home shows it; pause it → disappears.
3. Adsterra settings (as super admin) → paste code → save → turn on → storefront footer shows the sandboxed ad; turn off → gone.
4. Create a test user, verify the email, apply as seller → Admin Sellers → Reject (note ≥ 5 characters) → applicant sees the note. Apply again → Approve → store, `roles/<uid> = seller`, `adminLogs` entry exist.
5. Try approving with an **unverified** email: must be refused. Try the Worker URL with a non-admin account: must be 403.

## Limits you should know
* Free plans: Vercel Hobby is for non-commercial use; Cloudflare Workers Free allows 100,000 requests/day and 10 ms CPU per request (if the Worker reports CPU errors, that needs a paid plan); Firestore Spark quotas apply.
* Still Blaze-only: checkout, user suspension/roles, withdrawals, announcements, COD confirmation, uploads (see `docs/FIREBASE_CONFIGURATION.md`). They show clear messages and are not faked.
* Security note: the rules give admins broad write access to orders, stores, products and videos. The panel limits itself to safe fields; keep admin accounts few and protected.
