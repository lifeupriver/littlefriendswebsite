# DEPLOYMENT.md — Infrastructure & Deployment

> Vercel hosting, Supabase configuration, environment variables, CI/CD pipeline, domain setup, and monitoring. Read alongside SECURITY.md for auth config and API_INTEGRATIONS.md for third-party service setup.

---

## Hosting Architecture

```
littlefriendslearningloft.com
        ↓
   Vercel Edge Network (CDN + SSL)
        ↓
   Next.js 14+ App Router
   ├── Static pages (SSG) → served from edge cache
   ├── Dynamic pages (SSR) → Vercel serverless functions
   └── API routes → Vercel serverless functions
        ↓
   Supabase (us-east-1)
   ├── PostgreSQL database
   ├── Auth (email/password, magic link, Google SSO)
   ├── Storage (photos, documents)
   └── Realtime (portal notifications)
```

---

## Vercel Configuration

### Project Setup

1. Create new project in Vercel dashboard
2. Connect to GitHub repository
3. Framework preset: Next.js (auto-detected)
4. Build command: `next build` (default)
5. Output directory: `.next` (default)
6. Install command: `npm ci`
7. Node.js version: 20.x

### Deployment Settings

| Setting | Value |
|---|---|
| Production branch | `main` |
| Preview branches | All branches get preview deployments |
| Auto-deploy | Enabled (push to `main` → production deploy) |
| Build timeout | 5 minutes (default) |
| Serverless function region | `iad1` (US East — Virginia, closest to Newburgh) |
| Edge function runtime | Edge (for middleware) |

### `vercel.json`

```json
{
  "framework": "nextjs",
  "regions": ["iad1"],
  "crons": [
    {
      "path": "/api/cron/daily-reports",
      "schedule": "0 18 * * 1-5"
    },
    {
      "path": "/api/cron/payment-reminders",
      "schedule": "0 9 1,15 * *"
    },
    {
      "path": "/api/cron/attendance-reminder",
      "schedule": "0 7 * * 1-5"
    },
    {
      "path": "/api/cron/immunization-expiry",
      "schedule": "0 9 1 * *"
    },
    {
      "path": "/api/cron/backup-check",
      "schedule": "0 3 * * *"
    }
  ]
}
```

### Headers Configuration

Via `next.config.ts`:

```typescript
async headers() {
  return [
    {
      source: '/(.*)',
      headers: [
        { key: 'X-Frame-Options', value: 'DENY' },
        { key: 'X-Content-Type-Options', value: 'nosniff' },
        { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
        { key: 'Permissions-Policy', value: 'camera=(), microphone=(), geolocation=()' },
        {
          key: 'Content-Security-Policy',
          value: [
            "default-src 'self'",
            "script-src 'self' 'unsafe-inline' 'unsafe-eval' https://js.stripe.com https://www.googletagmanager.com",
            "style-src 'self' 'unsafe-inline' https://fonts.googleapis.com",
            "font-src 'self' https://fonts.gstatic.com",
            "img-src 'self' data: blob: https://*.supabase.co https://maps.googleapis.com https://maps.gstatic.com",
            "connect-src 'self' https://*.supabase.co wss://*.supabase.co https://api.anthropic.com https://api.stripe.com https://www.google-analytics.com https://api.resend.com",
            "frame-src https://js.stripe.com https://www.google.com",
          ].join('; '),
        },
      ],
    },
    {
      // Cache static assets aggressively
      source: '/images/(.*)',
      headers: [
        { key: 'Cache-Control', value: 'public, max-age=31536000, immutable' },
      ],
    },
  ];
},
```

---

## Supabase Configuration

### Project Setup

1. Create project in Supabase dashboard (region: `us-east-1`)
2. Note project URL and keys (anon key, service role key)
3. Enable Row Level Security on ALL tables (non-negotiable)
4. Configure auth providers (see below)
5. Create storage buckets
6. Enable Realtime on relevant tables

### Auth Configuration

**Email/Password:** Enabled (primary auth method for parents and staff)

**Magic Link:** Enabled (for password-free portal login option)

**Google SSO:** Optional — enabled for convenience, not required

**Auth settings:**
- Site URL: `https://littlefriendslearningloft.com`
- Redirect URLs: 
  - `https://littlefriendslearningloft.com/auth/callback`
  - `https://littlefriendslearningloft.com/portal`
  - `https://littlefriendslearningloft.com/admin`
  - `http://localhost:3000/auth/callback` (dev only)
- JWT expiry: 3600 seconds (1 hour)
- Refresh token rotation: Enabled
- Email templates: Customized with LFLL branding (yellow header, Fredoka One heading, Nunito body)

### Storage Buckets

| Bucket | Access | Purpose |
|---|---|---|
| `photos` | Authenticated read/write, public read for approved photos | Classroom photos, daily reports, event photos |
| `documents` | Authenticated read/write | Enrollment docs, permission slips, certificates, incident reports |
| `public-assets` | Public read | Site images, blog photos, teacher headshots |
| `exports` | Admin read/write | CSV exports, report PDFs |

**Storage policies:**
- Max file size: 10MB (photos), 25MB (documents/PDFs)
- Allowed MIME types: `image/jpeg`, `image/png`, `image/webp`, `application/pdf`, `text/csv`
- Photos auto-processed through Sharp on upload (strip EXIF, resize, generate thumbnails)

### Realtime Configuration

Enable Realtime on these tables:
- `messages` — parent-teacher messaging
- `notifications` — push notifications
- `attendance_records` — live attendance dashboard
- `daily_reports` — real-time report publishing

### Database Backups

- Supabase automatic daily backups (included in Pro plan)
- Point-in-time recovery: 7 days
- Manual backup before any migration or major schema change
- Monthly export of critical tables to CSV (via cron job → Supabase storage)

---

## Environment Variables

### Vercel Environment Variables

Set in Vercel Dashboard → Project → Settings → Environment Variables:

```env
# ── Supabase ──
NEXT_PUBLIC_SUPABASE_URL=https://xxxxxxxxxxxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJhbGci...
SUPABASE_SERVICE_ROLE_KEY=eyJhbGci...

# ── Stripe ──
STRIPE_SECRET_KEY=sk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_live_...

# ── Resend (Email) ──
RESEND_API_KEY=re_...
RESEND_FROM_EMAIL=hello@littlefriendslearningloft.com

# ── Twilio (SMS) ──
TWILIO_ACCOUNT_SID=AC...
TWILIO_AUTH_TOKEN=...
TWILIO_PHONE_NUMBER=+1845XXXXXXX

# ── Google Calendar API ──
GOOGLE_CALENDAR_CLIENT_ID=...apps.googleusercontent.com
GOOGLE_CALENDAR_CLIENT_SECRET=...
GOOGLE_CALENDAR_REFRESH_TOKEN=...
GOOGLE_CALENDAR_ID=...@group.calendar.google.com

# ── Google Maps ──
NEXT_PUBLIC_GOOGLE_MAPS_API_KEY=AIza...

# ── Claude API (AI Chatbot + Content Assist) ──
ANTHROPIC_API_KEY=sk-ant-...

# ── Cron Secret ──
CRON_SECRET=... (random 32-char string)

# ── App ──
NEXT_PUBLIC_SITE_URL=https://littlefriendslearningloft.com
NEXT_PUBLIC_APP_NAME=Little Friends Learning Loft

# ── Analytics ──
NEXT_PUBLIC_GA_MEASUREMENT_ID=G-XXXXXXXXXX
GOOGLE_SITE_VERIFICATION=...

# ── Feature Flags ──
NEXT_PUBLIC_ENABLE_CHATBOT=true
NEXT_PUBLIC_ENABLE_SMS=true
```

### Variable Scoping

- `NEXT_PUBLIC_*` — Available in browser AND server (safe for non-secrets)
- All others — Server-only (NEVER exposed to client bundle)
- **NEVER** put secret keys in `NEXT_PUBLIC_` variables

### Local Development

Create `.env.local` (git-ignored):

```env
# Point to Supabase local or development project
NEXT_PUBLIC_SUPABASE_URL=http://127.0.0.1:54321
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJ...local-anon-key
SUPABASE_SERVICE_ROLE_KEY=eyJ...local-service-role

# Stripe test mode
STRIPE_SECRET_KEY=sk_test_...
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_test_...
STRIPE_WEBHOOK_SECRET=whsec_test_...

# Other services (dev/test keys)
RESEND_API_KEY=re_test_...
ANTHROPIC_API_KEY=sk-ant-...
CRON_SECRET=dev-cron-secret

NEXT_PUBLIC_SITE_URL=http://localhost:3000
```

---

## Domain Setup

### DNS Configuration

Domain `littlefriendslearningloft.com` is currently pointing to Squarespace. At Phase 1 cutover:

1. Add custom domain in Vercel Dashboard → Project → Settings → Domains
2. Vercel provides required DNS records:
   - `A` record: `76.76.21.21` (Vercel)
   - `CNAME` for `www`: `cname.vercel-dns.com`
3. Update DNS at domain registrar (remove Squarespace records, add Vercel records)
4. Vercel auto-provisions SSL certificate (Let's Encrypt)
5. Redirect `www` → apex domain (or vice versa — pick one canonical)
6. Verify SSL is active and all pages load over HTTPS

### Email DNS (for Resend)

To send email from `hello@littlefriendslearningloft.com`:

1. Add domain in Resend dashboard
2. Add DNS records Resend provides:
   - SPF: TXT record
   - DKIM: CNAME records (3 records)
   - DMARC: TXT record
3. Verify domain in Resend dashboard
4. Test delivery to Gmail, Outlook, Yahoo

---

## CI/CD Pipeline

### GitHub Actions Workflow

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npm run type-check
      - run: npm run test -- --ci

  preview:
    needs: quality
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      # Vercel auto-deploys preview for PRs via GitHub integration
      # This step just ensures CI passes first
```

### Branch Strategy

| Branch | Purpose | Deploys To |
|---|---|---|
| `main` | Production code | littlefriendslearningloft.com |
| `develop` | Integration branch | Preview URL |
| `feature/*` | Feature branches | Preview URL (per-PR) |
| `hotfix/*` | Urgent production fixes | Merged directly to `main` |

### Pre-Deploy Checklist (Manual, Before Merging to Main)

- [ ] All CI checks pass (lint, types, tests)
- [ ] Preview deployment tested visually
- [ ] No console errors in browser dev tools
- [ ] Mobile viewport checked (375px)
- [ ] New pages have meta tags and structured data
- [ ] New images have alt text
- [ ] Any new tables have RLS policies
- [ ] Environment variables added to Vercel if needed

---

## Third-Party Service Setup

### Stripe Setup

1. Create Stripe account at stripe.com
2. Complete business verification (EIN for S-Corp or Rebecca's SSN for sole prop)
3. Set up products:
   - "Monthly Tuition — [Classroom Name]" (recurring subscription)
   - "Before Care" (recurring add-on)
   - "After Care" (recurring add-on)
   - "Enrichment — [Class Name]" (one-time or recurring per session)
   - "Summer Camp — [Session]" (one-time)
   - "Application Fee" (one-time, $50–$100)
4. Configure webhook endpoint: `https://littlefriendslearningloft.com/api/webhooks/stripe`
5. Webhook events to listen for:
   - `invoice.payment_succeeded`
   - `invoice.payment_failed`
   - `customer.subscription.updated`
   - `customer.subscription.deleted`
   - `checkout.session.completed`
6. Copy keys to Vercel environment variables

### Resend Setup

1. Create account at resend.com
2. Add and verify domain (DNS records above)
3. Create API key → `RESEND_API_KEY`
4. Set sending address: `hello@littlefriendslearningloft.com`

### Twilio Setup

1. Create account at twilio.com
2. Purchase a local phone number (845 area code preferred for Newburgh)
3. Configure messaging service
4. Copy Account SID, Auth Token, Phone Number to env vars
5. Test SMS delivery

### Google Calendar API Setup

1. Create project in Google Cloud Console
2. Enable Google Calendar API
3. Create OAuth 2.0 credentials (web application)
4. Authorized redirect URI: `https://littlefriendslearningloft.com/api/auth/google-calendar/callback`
5. Rebecca authenticates once → store refresh token as env var
6. Create a dedicated "LFLL School Calendar" in Google Calendar
7. Share calendar ID to `GOOGLE_CALENDAR_ID` env var

### Claude API Setup

1. Create account at console.anthropic.com
2. Generate API key → `ANTHROPIC_API_KEY`
3. Set up billing (usage-based)
4. Expected cost: $5–15/month for chatbot + content assist (see AI_INTEGRATION.md)

### Google Analytics 4 Setup

1. Create GA4 property for littlefriendslearningloft.com
2. Copy Measurement ID → `NEXT_PUBLIC_GA_MEASUREMENT_ID`
3. Set up conversion events:
   - `tour_booking` (tour form submission)
   - `inquiry_submission` (inquiry form submission)
   - `chatbot_engaged` (chatbot opened + question asked)
4. Link to Google Search Console

---

## Monitoring

### Error Monitoring

**Vercel Logs:** Real-time function logs in Vercel dashboard (free)

**Error Boundaries:** React error boundaries on every portal/admin section — catch and display friendly error, log to console. Add Sentry in Phase 7 if needed.

**Cron Monitoring:** Each cron job logs success/failure to `cron_logs` table in Supabase. Admin dashboard shows cron status.

### Uptime Monitoring

**Vercel Status:** Vercel provides uptime monitoring for Pro plans.

**Simple approach (free):** Set up UptimeRobot or Better Uptime to ping `/api/health` every 5 minutes.

**Health endpoint:**

```typescript
// app/api/health/route.ts
export async function GET() {
  const checks = {
    database: false,
    storage: false,
    timestamp: new Date().toISOString(),
  };

  try {
    const { error } = await supabase.from('health_check').select('id').limit(1);
    checks.database = !error;
  } catch {}

  try {
    const { error } = await supabase.storage.from('public-assets').list('', { limit: 1 });
    checks.storage = !error;
  } catch {}

  const healthy = checks.database && checks.storage;
  return Response.json(checks, { status: healthy ? 200 : 503 });
}
```

### Performance Monitoring

- **Vercel Analytics:** Real User Monitoring (RUM) for Core Web Vitals — free, included
- **Google Search Console:** Crawl errors, indexing status, Core Web Vitals field data
- **Lighthouse CI:** Run in GitHub Actions on PRs to catch performance regressions

---

## Cost Estimate

### Monthly Operating Costs

| Service | Plan | Monthly Cost |
|---|---|---|
| Vercel | Pro | $20 |
| Supabase | Pro | $25 |
| Stripe | Pay-as-you-go | 2.9% + $0.30 per transaction |
| Resend | Free tier (3,000 emails/month) | $0 |
| Twilio | Pay-as-you-go | ~$5–10 (phone number + SMS) |
| Claude API | Usage-based | ~$5–15 |
| Google Calendar API | Free | $0 |
| Google Maps Embed | Free tier | $0 |
| Domain renewal | Annual | ~$15/year ($1.25/month) |
| **Total (excluding Stripe %)** | | **~$55–75/month** |

### Scaling Considerations

At 50 families, all services are well within free/base tiers. No scaling concerns until 200+ families, which would require:
- Supabase: check database size (8GB on Pro)
- Vercel: check serverless function invocations
- Resend: upgrade if > 3,000 emails/month
- Twilio: costs scale linearly with SMS volume
