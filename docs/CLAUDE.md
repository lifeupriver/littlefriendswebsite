# CLAUDE.md — Little Friends Learning Loft

> Single source of truth for Claude Code. Read this file first before any task.

## Project Identity

- **Name:** Little Friends Learning Loft (LFLL)
- **Type:** Full-stack web application replacing Squarespace + Brightwheel + 6 other tools
- **Client:** Rebecca Harrison, Founder/Lead Teacher — Montessori preschool for ages 3–6
- **Location:** 290 North St, Newburgh, NY 12550 (inside Newburgh Jewish Community Center)
- **Developer:** Joshua Brown
- **Domain:** littlefriendslearningloft.com

## What This App Replaces

| Current Tool | Replacement |
|---|---|
| Squarespace | Modern public website with SEO/AEO, blog, lead capture |
| Brightwheel | Parent communication, attendance, billing, daily reports |
| Calendly | Tour scheduling, consultations, enrichment coordination |
| Google Forms | Before/after care, field trips, permission slips, volunteers |
| Google Sheets | Waitlist, tuition add-ons, aftercare billing, immunization tracking |
| Paper processes | Incident reports, conference forms, enrollment paperwork |
| Slack (photo channel) | Photo management and curation system |
| Canva newsletter | Newsletter builder/distribution |
| Sign-Up Genius | Conference scheduling, volunteer slots |

## Tech Stack

```
Frontend:        Next.js 14+ (App Router), TypeScript
Styling:         Tailwind CSS + custom design tokens
Components:      shadcn/ui base + custom components
Animation:       Framer Motion
State:           Zustand (client), Supabase Realtime (server)
Database:        Supabase (PostgreSQL)
Auth:            Supabase Auth (email/password, magic link, optional Google SSO)
Storage:         Supabase Storage (photos, documents, certificates)
Image Pipeline:  Sharp (resize, format conversion, EXIF stripping)
Payments:        Stripe (subscriptions, one-time, webhooks)
Email:           Resend (transactional + marketing)
SMS:             Twilio (emergency broadcast, reminders, business number)
AI:              Claude API (public chatbot, admin content assist)
Calendar:        Google Calendar API (2-way sync)
Hosting:         Vercel (SSG for public, SSR for portals)
Analytics:       Vercel Analytics + Google Analytics 4
```

## Accounts & Infrastructure

- **GitHub:** Ready ✅
- **Vercel:** Ready ✅
- **Supabase:** Ready ✅
- **Stripe:** Needs setup
- **Resend:** Needs setup
- **Twilio:** Needs setup
- **Google Calendar API:** Needs setup
- **Claude API:** Needs setup

## Architecture Overview

```
/(public)     → SSG/ISR — Homepage, About, Programs, Admissions, Community, Careers, Contact
/(portal)     → SSR + Auth — Parent portal (families, children, billing, calendar, messaging)
/(admin)      → SSR + Auth — Rebecca's command center (enrollment, billing, CMS, compliance)
/(staff)      → SSR + Auth — Teacher tools (attendance, photos, incidents, training)
/api/         → Route Handlers — mutations, webhooks, cron jobs
```

## User Roles & Permissions

| Role | Access |
|---|---|
| `admin` | Full access to everything (Rebecca) |
| `board` | Read-only: financial reports, enrollment data, compliance |
| `staff` | Classroom management, attendance, photos, incidents, own training — NO billing |
| `enrichment` | Own schedule, session attendance, hour logging |
| `parent` | Own family data, children, payments, communication, signups |
| `public` | Unauthenticated website access only |

## Critical Non-Negotiables (Rebecca's Requirements)

1. **No tuition/fees on public website** — pricing shared only after tour/inquiry
2. **No teacher bios on public website** — only Rebecca's bio (staff privacy/safety)
3. **School calendar is NEVER public** — portal only (safety)
4. **Children's faces OK** but no full names or identifying info
5. **Yellow brand color (#F5C518) must be preserved**
6. **Fonts must feel playful** — Fredoka One (headings) + Nunito (body)
7. **Testimonials sprinkled throughout** — not a dedicated page
8. **Step-by-step clarity** for every multi-step process
9. **Contact form ≠ Enrollment inquiry form** — visually and functionally different
10. **Scroll-based navigation** — Rebecca dislikes tabs at the top

## Brand Colors

```
Primary Yellow:    #F5C518
Secondary Yellow:  #FFF3C4
Background:        #FFFDF7 (warm off-white)
Text Primary:      #2D2926 (warm charcoal)
Text Secondary:    #6B6560
Accent Green:      #4A7C59
Accent Blue:       #5B8DB8
Accent Coral:      #E8775D
Border/Divider:    #E8E4DF
Card Background:   #FFFFFF
```

## Typography

```
Headings:  Fredoka One 400 (playful, rounded)
Body:      Nunito 400/500/600/700 (warm, readable)
Admin:     JetBrains Mono (data tables only)
```

## Key Metrics (Success Criteria)

- Tour booking rate: 15%+ of inquiry form submissions
- Inquiry → first automated follow-up: < 2 minutes
- Parent portal adoption: 90%+ active within 30 days
- Rebecca's weekly admin hours: from ~37 → ~12
- Google review growth: 2+ per month
- Blog cadence: 2+ posts per month
- AEO: mentioned when AI engines answer "best preschool in Newburgh NY"

## 7-Phase Build Plan

| Phase | Weeks | Focus |
|---|---|---|
| 1 | 1–4 | Foundation: public website, design system, lead capture, basic CMS |
| 2 | 5–8 | Admissions: tour scheduling, pipeline, automated follow-up |
| 3 | 9–14 | Parent Portal: auth, dashboard, calendar, forms, messaging |
| 4 | 15–18 | Billing: Stripe integration, invoicing, autopay |
| 5 | 19–22 | Operations: compliance, staff mgmt, work-trade, grants, reports |
| 6 | 23–26 | Advanced: AI chatbot, newsletter, photos, events, donations, camp |
| 7 | Ongoing | Iteration: bilingual, Instagram API, GBP API, yearbook, analytics |

## Documentation Index

| Doc | Purpose |
|---|---|
| [docs/PRD.md](docs/PRD.md) | Full product requirements — master spec |
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | System architecture, rendering, project structure |
| [docs/FRONTEND.md](docs/FRONTEND.md) | Components, pages, motion system, responsive |
| [docs/BACKEND.md](docs/BACKEND.md) | API routes, server actions, cron jobs, webhooks |
| [docs/DESIGN_SYSTEM.md](docs/DESIGN_SYSTEM.md) | Colors, typography, spacing, components, motion tokens |
| [docs/DATA_MODEL.md](docs/DATA_MODEL.md) | Database schema, RLS, indexes, migrations |
| [docs/SECURITY.md](docs/SECURITY.md) | Auth, RBAC, RLS, COPPA, photo privacy, audit trail |
| [docs/API_INTEGRATIONS.md](docs/API_INTEGRATIONS.md) | Stripe, Google Calendar, Resend, Twilio |
| [docs/AI_INTEGRATION.md](docs/AI_INTEGRATION.md) | Chatbot spec, content assist, RAG pipeline |
| [docs/SEO_AEO.md](docs/SEO_AEO.md) | Schema markup, AEO content, local SEO |
| [docs/ADMIN_DASHBOARD.md](docs/ADMIN_DASHBOARD.md) | Rebecca's command center — all admin features |
| [docs/CONTENT_STRATEGY.md](docs/CONTENT_STRATEGY.md) | Voice/tone, blog plan, email sequences |
| [docs/MIGRATION.md](docs/MIGRATION.md) | Squarespace/Brightwheel data migration |
| [docs/DEPLOYMENT.md](docs/DEPLOYMENT.md) | Vercel, Supabase, env config, CI/CD |
| [docs/ROADMAP.md](docs/ROADMAP.md) | Phased implementation with dependencies |

## Development Commands

```bash
npm run dev                  # Start dev server
npx supabase start           # Local Supabase
npx supabase db push         # Push migrations
npx supabase gen types       # Generate TypeScript types
vercel                       # Deploy preview
vercel --prod                # Deploy production
npm run lint                 # Lint
npm run type-check           # TypeScript check
npm run test                 # Tests
```

## Environment Variables

```env
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=
RESEND_API_KEY=
TWILIO_ACCOUNT_SID=
TWILIO_AUTH_TOKEN=
TWILIO_PHONE_NUMBER=
GOOGLE_CALENDAR_CLIENT_ID=
GOOGLE_CALENDAR_CLIENT_SECRET=
GOOGLE_CALENDAR_REFRESH_TOKEN=
GOOGLE_MAPS_API_KEY=
ANTHROPIC_API_KEY=
NEXT_PUBLIC_SITE_URL=https://littlefriendslearningloft.com
NEXT_PUBLIC_APP_NAME="Little Friends Learning Loft"
```

## Coding Conventions

- **TypeScript strict mode** — no `any` types
- **Server Components by default** — `'use client'` only when needed
- **Supabase queries in `/lib/supabase/`** — never inline DB calls in components
- **Zod validation** on all form inputs and API routes
- **Error boundaries** around every portal/admin section
- **Loading states** — skeleton loaders matching content shape, never spinners
- **Empty states** — warm encouraging copy with hand-drawn illustrations
- **Optimistic UI** for common actions (attendance, messaging)
- **All images through Sharp pipeline** — strip EXIF, generate responsive sizes
- **RLS on every table** — test for cross-family data leakage
- **Audit log** every admin action and sensitive parent action
- **Print stylesheets** for incident reports, invoices, conference reports

## File Naming

```
Components:  PascalCase.tsx        (HeroSection.tsx, EnrollmentPipeline.tsx)
Utilities:   camelCase.ts          (formatCurrency.ts, validatePhone.ts)
API routes:  route.ts              (/api/leads/route.ts)
Types:       camelCase.types.ts    (enrollment.types.ts)
Hooks:       use-*.ts              (use-family.ts, use-attendance.ts)
```

## Important Context

- **Rebecca** is the sole admin user with moderate tech comfort — progressive disclosure UI
- **~50 children** across 3 classrooms, 2 teachers each (2:12 ratio)
- **Summer camp** is a separate seasonal program with its own registration/payment
- **Work-trade families** receive tuition adjustments for volunteer hours
- **OCFS** (NYS Office of Children and Family Services) regulates — compliance tracking is critical
- **The JCC** is the landlord — LFLL is a secular program using JCC spaces
- **Newburgh, NY** has significant Spanish-speaking population — i18n architecture from start, bilingual content in Phase 7
