# Architecture — Little Friends Learning Loft

## System Overview

LFLL is a Next.js 14+ App Router application with four distinct route groups, a Supabase backend, and integrations with Stripe, Google Calendar, Resend, Twilio, and the Claude API. It serves as both a public marketing site (SSG) and a multi-portal authenticated application (SSR).

```
┌─────────────────────────────────────────────────────┐
│                     Vercel (Edge + Node)             │
│  ┌───────────────────────────────────────────────┐   │
│  │              Next.js App Router                │   │
│  │                                                │   │
│  │  /(public)  SSG/ISR    Marketing site          │   │
│  │  /(portal)  SSR        Parent portal           │   │
│  │  /(admin)   SSR        Admin dashboard         │   │
│  │  /(staff)   SSR        Staff portal            │   │
│  │  /api/      Edge/Node  Route handlers          │   │
│  └──────────────────┬────────────────────────────┘   │
│                     │                                 │
└─────────────────────┼─────────────────────────────────┘
                      │
        ┌─────────────┼──────────────┐
        │             │              │
        ▼             ▼              ▼
  ┌──────────┐  ┌──────────┐  ┌──────────────┐
  │ Supabase │  │  Stripe  │  │ External APIs│
  │ Postgres │  │ Payments │  │ Google Cal   │
  │ Auth     │  │ Webhooks │  │ Resend       │
  │ Storage  │  │          │  │ Twilio       │
  │ Realtime │  │          │  │ Claude API   │
  └──────────┘  └──────────┘  │ Google Maps  │
                               └──────────────┘
```

## Rendering Strategy

| Route Group | Strategy | Reasoning |
|---|---|---|
| `/(public)/*` | SSG with ISR | Maximum SEO/AEO performance. Rebuilt via ISR on content changes. |
| `/(public)/community/blog/*` | ISR | Static at build, revalidates when new posts are published. |
| `/(portal)/*` | SSR | Requires session validation, displays real-time family data. |
| `/(admin)/*` | SSR | Requires session validation, real-time operations dashboard. |
| `/(staff)/*` | SSR | Requires session validation, classroom real-time data. |
| `/api/*` | Edge/Node Runtime | Mutations, webhook handlers, cron jobs. |

### SSG/ISR Pages
All public pages render to static HTML at build time. Content changes trigger ISR revalidation via Supabase webhook → Vercel deploy hook or on-demand revalidation via `revalidatePath()`.

### SSR Pages
All authenticated routes validate the Supabase session server-side via `@supabase/ssr`. No authenticated data renders on the client until the server confirms a valid session and appropriate role.

## Project Structure (Detailed)

```
/
├── CLAUDE.md                    # Single source of truth
├── CLAUDE_CODE_PROMPT.md        # Kickoff prompt
├── docs/                        # All spec documents
├── app/
│   ├── layout.tsx               # Root layout: fonts, metadata, analytics, providers
│   ├── not-found.tsx            # Custom 404 with playful illustration
│   ├── error.tsx                # Root error boundary
│   ├── globals.css              # Tailwind directives + custom properties
│   │
│   ├── (public)/                # Public website (SSG/ISR)
│   │   ├── layout.tsx           # Public layout: navbar, footer, mobile CTA bar, chatbot
│   │   ├── page.tsx             # Homepage (scroll sections)
│   │   ├── about/
│   │   │   ├── rebecca/page.tsx
│   │   │   ├── montessori/page.tsx
│   │   │   ├── jcc/page.tsx
│   │   │   └── mission/page.tsx
│   │   ├── programs/
│   │   │   ├── primary/page.tsx
│   │   │   ├── kindergarten/page.tsx
│   │   │   ├── enrichment/page.tsx
│   │   │   ├── before-after-care/page.tsx
│   │   │   └── clubs/page.tsx
│   │   ├── admissions/
│   │   │   ├── process/page.tsx
│   │   │   ├── tour/page.tsx
│   │   │   ├── apply/page.tsx
│   │   │   ├── faq/page.tsx
│   │   │   └── readiness/page.tsx
│   │   ├── summer-camp/
│   │   │   ├── overview/page.tsx
│   │   │   ├── weekly-themes/page.tsx
│   │   │   ├── register/page.tsx
│   │   │   └── staff/page.tsx
│   │   ├── community/
│   │   │   ├── events/page.tsx
│   │   │   ├── workshops/page.tsx
│   │   │   ├── blog/
│   │   │   │   ├── page.tsx         # Blog index
│   │   │   │   └── [slug]/page.tsx  # Individual blog post
│   │   │   ├── press/page.tsx
│   │   │   └── donate/page.tsx
│   │   ├── careers/
│   │   │   ├── openings/page.tsx
│   │   │   └── apply/page.tsx
│   │   ├── contact/page.tsx
│   │   ├── privacy/page.tsx
│   │   ├── terms/page.tsx
│   │   └── newsletter-signup/page.tsx
│   │
│   ├── (portal)/                # Parent portal (SSR, auth)
│   │   ├── layout.tsx           # Portal layout: sidebar, session check
│   │   └── portal/
│   │       ├── page.tsx         # Redirect to dashboard
│   │       ├── dashboard/page.tsx
│   │       ├── calendar/page.tsx
│   │       ├── children/
│   │       │   ├── page.tsx
│   │       │   └── [id]/page.tsx
│   │       ├── forms/page.tsx
│   │       ├── payments/page.tsx
│   │       ├── communication/page.tsx
│   │       ├── signup/page.tsx
│   │       ├── resources/page.tsx
│   │       └── settings/page.tsx
│   │
│   ├── (admin)/                 # Admin dashboard (SSR, auth)
│   │   ├── layout.tsx           # Admin layout: sidebar, session + role check
│   │   └── admin/
│   │       ├── page.tsx         # Redirect to dashboard
│   │       ├── dashboard/page.tsx
│   │       ├── enrollment/page.tsx
│   │       ├── families/
│   │       │   ├── page.tsx
│   │       │   └── [id]/page.tsx
│   │       ├── children/
│   │       │   ├── page.tsx
│   │       │   └── [id]/page.tsx
│   │       ├── staff/page.tsx
│   │       ├── billing/page.tsx
│   │       ├── scheduling/page.tsx
│   │       ├── communications/page.tsx
│   │       ├── content/
│   │       │   ├── blog/page.tsx
│   │       │   ├── photos/page.tsx
│   │       │   ├── faqs/page.tsx
│   │       │   ├── pages/page.tsx
│   │       │   └── newsletter/page.tsx
│   │       ├── events/page.tsx
│   │       ├── reports/page.tsx
│   │       ├── compliance/page.tsx
│   │       ├── grants/page.tsx
│   │       ├── work-trade/page.tsx
│   │       ├── inventory/page.tsx
│   │       ├── press/page.tsx
│   │       └── settings/page.tsx
│   │
│   ├── (staff)/                 # Staff portal (SSR, auth)
│   │   ├── layout.tsx
│   │   └── staff/
│   │       ├── dashboard/page.tsx
│   │       ├── classroom/page.tsx
│   │       ├── photos/page.tsx
│   │       ├── incidents/page.tsx
│   │       ├── training/page.tsx
│   │       ├── messages/page.tsx
│   │       └── resources/page.tsx
│   │
│   ├── (auth)/                  # Auth pages (login, signup, reset)
│   │   ├── login/page.tsx
│   │   ├── signup/page.tsx
│   │   ├── reset-password/page.tsx
│   │   └── callback/route.ts    # OAuth/magic link callback
│   │
│   └── api/
│       ├── webhooks/
│       │   └── stripe/route.ts  # Stripe webhook handler
│       ├── cron/
│       │   ├── daily-digest/route.ts
│       │   ├── immunization-reminders/route.ts
│       │   ├── payment-reminders/route.ts
│       │   └── lead-sequence/route.ts
│       ├── contact/route.ts     # Contact form submission
│       ├── inquiry/route.ts     # Enrollment inquiry submission
│       ├── tour/route.ts        # Tour booking
│       ├── upload/route.ts      # Photo/document upload with Sharp
│       ├── chat/route.ts        # AI chatbot endpoint
│       └── revalidate/route.ts  # On-demand ISR revalidation
│
├── components/
│   ├── ui/                      # shadcn/ui base (Button, Card, Dialog, etc.)
│   ├── public/                  # Public site components
│   │   ├── Hero.tsx
│   │   ├── FeatureCard.tsx
│   │   ├── TestimonialCard.tsx
│   │   ├── AdmissionsStepper.tsx
│   │   ├── PhotoGallery.tsx
│   │   ├── CTABanner.tsx
│   │   ├── FAQAccordion.tsx
│   │   ├── MobileStickyBar.tsx
│   │   ├── Chatbot.tsx
│   │   └── ExitIntentModal.tsx
│   ├── portal/                  # Parent portal components
│   │   ├── DashboardSnapshot.tsx
│   │   ├── ActionItems.tsx
│   │   ├── CalendarWidget.tsx
│   │   ├── BillingCard.tsx
│   │   ├── MessageThread.tsx
│   │   └── SignupCard.tsx
│   ├── admin/                   # Admin dashboard components
│   │   ├── MorningBriefing.tsx
│   │   ├── EnrollmentKanban.tsx
│   │   ├── FamilyCard.tsx
│   │   ├── ComplianceMatrix.tsx
│   │   ├── FinancialChart.tsx
│   │   ├── BlogEditor.tsx
│   │   ├── PhotoCurationQueue.tsx
│   │   └── BoardReportGenerator.tsx
│   ├── staff/                   # Staff portal components
│   │   ├── AttendanceTracker.tsx
│   │   ├── IncidentReportForm.tsx
│   │   ├── PhotoUploader.tsx
│   │   └── TrainingProgress.tsx
│   ├── shared/                  # Cross-portal shared components
│   │   ├── Calendar.tsx
│   │   ├── FormStepper.tsx
│   │   ├── StatusBadge.tsx
│   │   ├── NotificationToast.tsx
│   │   ├── DataTable.tsx
│   │   ├── FileUpload.tsx
│   │   └── ConfirmDialog.tsx
│   └── motion/                  # Framer Motion wrappers
│       ├── FadeIn.tsx
│       ├── StaggerChildren.tsx
│       ├── ScrollReveal.tsx
│       ├── PageTransition.tsx
│       └── ConfettiBurst.tsx
│
├── lib/
│   ├── supabase/
│   │   ├── client.ts            # Browser client
│   │   ├── server.ts            # Server client (cookies-based)
│   │   ├── admin.ts             # Service role client (server-only)
│   │   ├── middleware.ts        # Session refresh middleware
│   │   └── queries/             # Typed query functions per table
│   ├── stripe/
│   │   ├── client.ts            # Stripe instance
│   │   ├── webhooks.ts          # Webhook event handlers
│   │   └── invoices.ts          # Invoice creation/management
│   ├── google/
│   │   ├── calendar.ts          # 2-way calendar sync
│   │   └── maps.ts              # Map embed helper
│   ├── email/
│   │   ├── resend.ts            # Resend client
│   │   ├── templates/           # Email HTML templates
│   │   └── sequences.ts         # Automated sequence runner
│   ├── sms/
│   │   └── twilio.ts            # SMS sending
│   ├── ai/
│   │   ├── chatbot.ts           # Claude chatbot logic + knowledge boundaries
│   │   └── content-assist.ts    # Admin content generation helpers
│   ├── images/
│   │   └── pipeline.ts          # Sharp: compress, EXIF strip, resize, WebP/AVIF
│   └── utils/
│       ├── constants.ts         # App-wide constants
│       ├── formatters.ts        # Date, currency, phone formatters
│       ├── validators.ts        # Zod schemas
│       └── roles.ts             # Role check helpers
│
├── types/
│   ├── database.ts              # Auto-generated from Supabase (npx supabase gen types)
│   ├── forms.ts                 # Form data types
│   ├── api.ts                   # API request/response types
│   └── index.ts                 # Shared types
│
├── hooks/
│   ├── useUser.ts               # Current user + role
│   ├── useFamily.ts             # Current family data
│   ├── useRealtime.ts           # Supabase realtime subscription
│   └── useMediaQuery.ts         # Responsive breakpoint detection
│
├── stores/
│   └── app-store.ts             # Zustand store (UI state, notifications)
│
├── styles/
│   ├── design-tokens.css        # CSS custom properties for colors, motion, spacing
│   └── print.css                # Print stylesheet for reports/invoices
│
├── public/
│   ├── favicon.ico
│   ├── apple-touch-icon.png     # 180x180
│   ├── icon-192.png             # PWA
│   ├── icon-512.png             # PWA
│   ├── manifest.json            # PWA manifest with LFLL brand
│   ├── og-image.png             # 1200x630 branded
│   └── illustrations/           # Hand-drawn SVG accents
│
├── middleware.ts                 # Auth middleware: session refresh, route protection
├── next.config.ts
├── tailwind.config.ts           # LFLL design tokens
├── tsconfig.json
└── package.json
```

## Data Flow Patterns

### Public Form Submission (Contact/Inquiry)
```
User fills form → Client-side Zod validation → POST /api/inquiry
→ Server validates → Insert into leads table → Trigger Resend email
→ Return success → Redirect to confirmation page
→ (Async) Start automated email sequence
```

### Authenticated Action (Parent Signs Form)
```
Parent clicks form → Server Component loads form data (RLS-scoped)
→ Parent fills + e-signs → POST /api/forms/submit
→ Server validates session + role + ownership → Update documents table
→ Log to audit_log → Send confirmation notification
→ Update dashboard action items via Supabase Realtime
```

### Photo Upload Pipeline
```
Staff uploads photo → POST /api/upload (multipart)
→ Sharp: strip EXIF/GPS → compress to max 2000px
→ Generate sizes: 400w, 800w, 1200w, 1600w
→ Generate formats: WebP, AVIF, JPEG fallback
→ Generate thumbnail (400px)
→ Store all variants in Supabase Storage
→ Insert into photos table (is_approved: false)
→ Photo appears in admin curation queue
→ Admin approves → Check child photo_release flags → Available for use
```

### Stripe Payment Flow
```
Admin generates invoice → Insert invoices table → Stripe invoice created
→ Parent views in portal → Clicks "Pay" → Stripe Checkout/Elements
→ Stripe processes → Webhook POST /api/webhooks/stripe
→ Verify signature → Update invoices + payments tables
→ Log to audit_log → Send receipt via Resend
→ Parent sees confirmation with celebration animation
```

## Real-Time Features (Supabase Realtime)

| Feature | Channel | Trigger |
|---|---|---|
| Attendance updates | `attendance:{classroom_id}` | Staff checks in/out a child |
| New messages | `messages:{user_id}` | Message sent to user |
| Dashboard action items | `dashboard:{family_id}` | Form submitted, payment received |
| Admin activity feed | `admin:activity` | Any significant action |
| Enrollment pipeline | `leads:changes` | Lead status changes |

## Middleware Chain

```typescript
// middleware.ts
1. Supabase session refresh (every request)
2. Route protection:
   - /portal/* → require role: parent
   - /admin/*  → require role: admin
   - /staff/*  → require role: staff, enrichment
3. Session timeout enforcement:
   - Admin: 30 min inactivity → redirect to login
   - Portal: 60 min inactivity → redirect to login
4. Rate limiting on /api/* endpoints
```

## Environment Variables

```env
# Supabase
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=

# Stripe
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=

# Resend
RESEND_API_KEY=

# Twilio
TWILIO_ACCOUNT_SID=
TWILIO_AUTH_TOKEN=
TWILIO_PHONE_NUMBER=

# Google
GOOGLE_CALENDAR_API_KEY=
GOOGLE_CALENDAR_CLIENT_ID=
GOOGLE_CALENDAR_CLIENT_SECRET=
GOOGLE_MAPS_API_KEY=

# Anthropic
ANTHROPIC_API_KEY=

# App
NEXT_PUBLIC_SITE_URL=https://littlefriendslearningloft.com
CRON_SECRET=
```

## Error Handling Strategy

- **API routes**: Try/catch with typed error responses, never expose raw errors
- **Server Components**: Error boundaries per route segment with branded fallback UI
- **Client Components**: React error boundaries with retry capability
- **External API failures**: Graceful degradation (e.g., calendar unavailable → show "check back soon")
- **Supabase RLS violations**: Caught server-side, logged, user sees "Access denied" with friendly messaging
- **Stripe webhook failures**: Retry with exponential backoff, alert admin on repeated failures
