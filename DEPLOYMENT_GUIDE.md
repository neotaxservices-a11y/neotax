# PracticeDesk Pro — Online Setup and Operations

**Deployment model:** Netlify (public HTTPS application and server functions) + Supabase (login, PostgreSQL, private documents) + optional Razorpay (payment links) and Resend (outbound billing email).

**Status:** Source-code delivery. This is **not** a hosted or production-audited application. You must supply accounts, configure the services, test access control and backups, and deploy it. Do **not** upload real clients' PANs, portal passwords or other confidential records until that is done.

## 1 — Prerequisites

- A computer with Node.js 20 or later and a GitHub account.
- A Supabase account at https://supabase.com and a Netlify account at https://www.netlify.com.
- Optional: a Razorpay merchant account and a Resend account with a verified sending domain.
- Use a **new, empty Supabase project** to prevent conflicts with unrelated databases.

## 2 — Configure Supabase

1. Supabase Dashboard → **New project**. Store the project database password securely.
2. In **SQL Editor**, open `supabase/schema.sql` and execute the complete file once, in the newly created project. It creates tables, indexes, Row Level Security (RLS) policies, accounting procedures, and a private storage bucket named `practice-documents`.
3. In **Project Settings → API / Data API**, copy the **Project URL** and **publishable** / **anon** key; these are the only Supabase configuration values used in the browser.
4. In **Authentication → Providers → Email**, enable email/password and email confirmation. Set a strong minimum password policy and turn on rate limits. Configure SMTP for reliable invitation / recovery / confirmation emails, particularly before adding staff.
5. In **Authentication → URL Configuration**, after Netlify gives you a URL, set **Site URL** to the Netlify HTTPS URL and add it under **Redirect URLs** (for example `https://your-site.netlify.app/**`).
6. For server functions, copy the **service_role** / secret key from Supabase dashboard, but **never** put it into a `VITE_` variable, repository, browser, or front-end code. It bypasses RLS.
7. Strongly recommended before live use: enable Supabase MFA for administrators (or use a provider that enforces it), monitor project logs and database backups, and test restore procedures. Confirm plan backup availability and retention in your project settings.

## 3 — Prepare the application

1. Extract this ZIP on your computer. Open the `PracticeDesk_Pro` folder in VS Code or terminal.
2. Run `npm install` then `npm run dev`. Vite shows the local development address (normally `http://localhost:5173`).
3. Copy `.env.example` to `.env` (which is ignored by Git) and set:

   ```text
   VITE_SUPABASE_URL=https://YOUR_PROJECT.supabase.co
   VITE_SUPABASE_ANON_KEY=YOUR_PUBLISHABLE_OR_ANON_KEY
   ```

4. Stop/restart the development server after changing `.env`.
5. Local **browser pages** may run at this stage. **Netlify server functions** (invitations, document-request links, Razorpay, automated email) require a Netlify development environment or a deployed Netlify site. Do not expect them to work under bare `vite dev`.
6. `npm run build` produces the static `dist` directory. `npm test` runs the source checks.

## 4 — Deploy to Netlify (easiest reproducible route)

1. Create a private GitHub repository and upload the **source folder**, excluding `.env` and any credentials. `netlify.toml` defines the build command, functions directory and SPA routing.
2. Netlify Dashboard → **Add new project → Import an existing project → GitHub → repository**.
3. Check **Build command:** `npm run build`, **Publish directory:** `dist`, **Functions directory:** `netlify/functions`.
4. Netlify **Site configuration → Environment variables**, add:

   | Variable | Where used | Required |
   |---|---|---|
   | `VITE_SUPABASE_URL` | Browser Vite build | Yes |
   | `VITE_SUPABASE_ANON_KEY` | Browser Vite build | Yes |
   | `SUPABASE_URL` | Server functions | Yes |
   | `SUPABASE_SERVICE_ROLE_KEY` | Server functions, secret | Yes |
   | `RAZORPAY_KEY_ID` | Server functions, secret | Optional |
   | `RAZORPAY_KEY_SECRET` | Server functions, secret | Optional |
   | `RAZORPAY_WEBHOOK_SECRET` | Webhook verification, secret | Optional |
   | `RESEND_API_KEY` | Email server function, secret | Optional |
   | `EMAIL_FROM` | Verified email sender | Optional |

5. Redeploy after setting variables. Open your assigned `https://something.netlify.app` link, not the local HTML file.
6. Sign up as the firm owner and confirm the email. If email confirmation opens in another browser tab and the firm details were lost, the **Create owner workspace** screen appears after sign-in: enter the firm name there.
7. Once signed in as Admin, open **Firm Settings** to set company address, GSTIN, bank details, invoice prefix and logo. This is where you update your Neotax branding.
8. Open **Staff → Invite staff** to email a login invitation. Each staff member uses their own email and password. Managers can edit clients, billing and accounts; staff can update progress only on their assigned tasks.
9. After the link works, optionally attach `app.neotax.in` under Netlify **Domain management** and set DNS according to the instructions provided there. HTTPS is issued by Netlify.

## 5 — Razorpay setup (optional)

1. Obtain your Razorpay API **Key ID** and **Key Secret** from the merchant dashboard. Start with test mode.
2. Add the keys as **Netlify server environment variables** (not `VITE_`), then redeploy functions.
3. In Razorpay Dashboard → Webhooks, configure the endpoint:

   `https://YOUR_NETLIFY_SITE/.netlify/functions/razorpay-webhook`

4. Create a random webhook signing secret, enter the **same secret** in Razorpay and Netlify as `RAZORPAY_WEBHOOK_SECRET`.
5. Enable the `payment_link.paid` and `payment_link.partially_paid` events (if available). Check Razorpay's currently supported event settings.
6. Within PracticeDesk create a GST invoice → **Razorpay** to create a link for its outstanding amount. A correctly signed webhook posts the collection into the receipts table.
7. Test payment, repeated webhook delivery, partially paid invoices and failed/cancelled payments before switching to live keys. Match settlement amounts to your actual Razorpay records and bank credits. **A link displayed as paid should not replace bank reconciliation.**

**Security:** API secrets are on Netlify only. The webhook verifies HMAC signatures against the exact raw request body. Browser users cannot mark an invoice paid by calling the payment endpoint.

## 6 — Email / WhatsApp communications

- **Email reminders via app:** Set `RESEND_API_KEY` and `EMAIL_FROM` (e.g. `Neotax <billing@neotax.in>`); verify ownership of the sender domain in Resend, then redeploy. Open an unpaid invoice and choose **Email reminder**.
- **WhatsApp:** **Remind** opens a prefilled WhatsApp message in your own WhatsApp account. You must click **Send**; this is not unattended WhatsApp API automation. Unattended template messages require a WhatsApp Business Platform provider, customer opt-ins, approved templates and a separate server integration.
- **A4 documents:** Choose **A4 PDF** or **Share PDF** from Billing. Browser device-share capability and messaging installation may affect share options.
- **Document requests:** Manager creates a checklist → app makes an expiring 14-day link → send it through email/WhatsApp → client uploads PDF/image/spreadsheet/document up to 15 MB → private storage and request checklist update. Treat the link as a bearer token and share only with the correct client. Regenerate it if forwarded unexpectedly.

## 7 — Import your old offline PracticeDesk clients

1. From your old **offline v4** PracticeDesk: Settings → **Export encrypted backup**. Store the backup safely.
2. In the online app: Firm Settings → **Import offline v4 backup**.
3. Select your encrypted `.json` backup and enter its original backup password. The decryption is done in the **browser**.
4. Review the prompt. The importer moves **client master records only** (excluding 5 known fictional demo client IDs), not passwords, notices, tasks, invoices, receipts or document binaries. This conservative limitation prevents accidental disclosure or corrupt reconciliations.
5. Validate GSTINs, contacts, fee terms and assignments. Keep the old backup until all information is reconciled. **Do not import the old credential vault into this application**; this online edition deliberately omits portal-password storage.

## 8 — Important operating controls

- Use individual logins; never share a principal's password with staff.
- Archive clients instead of deleting them. **Permanent deletion** may fail if invoices or other financial history depend on the client; preserve statutory records appropriately.
- Restrict invitations to trusted email addresses. Deactivate a departing employee under Staff immediately.
- Do not send portal credentials, Aadhaar or banking passwords through public document links, WhatsApp or regular email.
- Verify every statutory deadline against government notifications; automatic monthly tasks are **basic templates** (GST monthly, ESI and EPF) and are not an exhaustive compliance-law engine.
- Accounting is a **manual double-entry journal and trial-balance module**. It does not yet auto-post invoice/receipt journals, reconcile banks, generate statutory financial statements or replace audited accounting software.
- Invoicing covers line items, GST rate, tax mode, quotations and receipts, but advanced invoice numbering, place-of-supply checks, e-invoicing/IRN, credit notes and statutory invoice validation need additional development before fully automated compliance use.
- Keep independent backups. Supabase sync is not itself a backup; test data export and recovery before real deployment. Storage documents need separate backup/retention controls.
- This is an application *starter*, not a completed penetration test, compliance certification or production SLA. Consider a professional security review and privacy/retention policy before production.

## 9 — Test checklist before real client data

- [ ] Owner signs up, confirms email, creates firm and logs in from another laptop.
- [ ] Invite a test manager and a test staff account; each uses separate credentials.
- [ ] Staff sees shared clients but cannot view billing or change a different employee's assignment.
- [ ] Managers can create work, set statuses and send document requests.
- [ ] Client document link uploads a test PDF and shows received status; expired/revoked link fails.
- [ ] Cross-workspace user cannot access client rows or documents from another workspace.
- [ ] A4 invoice displays correct firm, GST totals, customer and bank details.
- [ ] Razorpay test payment records once, even if webhook is repeated.
- [ ] Reminders open to the correct client and optional email is sent only after setup.
- [ ] Exports/backups are independently restored and tested.

## Troubleshooting

- **Supabase not configured:** Add both `VITE_` variables and redeploy; Vite embeds these *public* values at build time.
- **SQL “already exists”:** Run schema on a **new, empty project**. This installer is not an idempotent migration.
- **Signup / invite email missing:** Check Supabase SMTP config and spam; invitation workflows rely on email delivery.
- **Workspace not linked:** Owner can create workspace on the screen after email verification; invited staff should not create a new workspace.
- **Netlify 404 for functions:** Deploy the **whole source repo through Netlify build** with its functions directory; don't only drag the prebuilt HTML to a static host.
- **CORS / Content-Security-Policy errors:** Verify Supabase project URL. Allow-listing only your real backend domains is preferred; `netlify.toml` already contains standard Supabase endpoints.
- **Functions work locally?** Use `netlify dev` with Netlify CLI (optional) or test on a Netlify deployment; plain Vite only serves the frontend.

**Use `npm run build` to verify the build, and complete the real integration checklist on your configured environment.**
