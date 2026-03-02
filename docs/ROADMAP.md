# ROADMAP.md — Phased Build Plan

> Week-by-week implementation plan across 7 phases. Each phase has prerequisites, deliverables, and a checklist. Claude Code should complete one phase at a time, verify everything works, then proceed to the next.

---

## Phase Overview

| Phase | Weeks | Focus | Deliverable |
|---|---|---|---|
| 1 | 1–4 | Foundation | Public website live, design system, lead capture, basic CMS |
| 2 | 5–8 | Admissions | Tour scheduling, enrollment pipeline, automated follow-up |
| 3 | 9–14 | Parent Portal | Auth, family dashboard, calendar, forms, messaging, photos |
| 4 | 15–18 | Billing | Stripe integration, invoicing, autopay, financial reports |
| 5 | 19–22 | Operations | Compliance tracking, staff tools, work-trade, grants, reporting |
| 6 | 23–26 | Advanced | AI chatbot, newsletter builder, event system, donations, summer camp |
| 7 | Ongoing | Iteration | Bilingual (i18n), Instagram API, GBP API, yearbook, analytics dashboard |

**Rule:** Complete each phase fully before starting the next. No half-built features.

---

## Phase 1: Foundation (Weeks 1–4)

**Goal:** Public website live at littlefriendslearningloft.com with design system, all public pages, lead capture, blog CMS, and SEO/AEO optimization. Replace Squarespace.

### Week 1: Project Setup + Design System

- [ ] Initialize Next.js 14+ project with App Router, TypeScript, Tailwind
- [ ] Configure Tailwind with custom design tokens from DESIGN_SYSTEM.md:
  - Colors: `#F5C518` primary, `#FFFDF7` background, `#2D2926` text, accents
  - Fonts: Fredoka One (headings), Nunito (body), JetBrains Mono (admin data)
  - Spacing scale, border radius, warm shadows
- [ ] Install and configure shadcn/ui as component base
- [ ] Build base UI component library (all from DESIGN_SYSTEM.md):
  - Button (primary, secondary, ghost, sizes)
  - Card (with warm shadow, optional yellow accent border)
  - Input, TextArea, Select
  - Badge, Avatar, Toast
  - Modal, BottomSheet (mobile)
  - Skeleton loaders (matching content shapes)
- [ ] Set up Framer Motion with motion tokens from DESIGN_SYSTEM.md:
  - `stagger-children`: 0.08s
  - `section-enter`: fade-up, 0.5s, ease-out
  - `card-hover`: translateY(-2px), shadow elevation
  - `page-transition`: fade, 0.3s
- [ ] Configure Supabase client (`lib/supabase/client.ts`, `lib/supabase/server.ts`)
- [ ] Set up environment variables (`.env.local` for dev)
- [ ] Deploy initial build to Vercel (preview URL)

### Week 2: Public Pages (Part 1)

- [ ] Build shared layout: Header (scroll-responsive nav), Footer (full navigation + newsletter signup)
- [ ] **Homepage** — 7 scrolling sections per PRD:
  1. Hero with CTA, gradient overlay, staggered entrance
  2. "What Makes Us Special" — 4 feature cards + testimonial
  3. "A Day at Little Friends" — photo gallery/video + narrative
  4. "Programs at a Glance" — program cards
  5. "Admissions — Your Path" — 3-step visual stepper
  6. "Community & Events" — event feed + blog previews + email signup
  7. "Our Home at the JCC" — photo grid + map embed
- [ ] **About Section:**
  - `/about/rebecca` — Rebecca's story (bio, credentials, photo, testimonial)
  - `/about/montessori` — AEO-structured Q&A (see SEO_AEO.md)
  - `/about/jcc` — Photo gallery of all JCC spaces
  - `/about/mission` — Mission, values, tikkun olam
- [ ] **Contact** — form, map embed, click-to-call, hours
- [ ] Implement SEO: title tags, meta descriptions, canonical URLs, OG tags per page

### Week 3: Public Pages (Part 2) + Blog CMS

- [ ] **Programs Section:**
  - `/programs/primary` — Multi-age 3–6, daily schedule, milestones, FAQ schema
  - `/programs/kindergarten` — K-prep program
  - `/programs/before-after-care` — Extended day details
  - `/programs/summer-camp` — Seasonal program
  - `/programs/enrichment` — Art, yoga/dance, music, skateboarding
- [ ] **Admissions** — Overview page with inquiry form (NOT enrollment — just lead capture)
- [ ] **FAQ** — 20+ questions, organized by category, FAQ schema on each
- [ ] **Careers** — Job listings, application link
- [ ] **Blog system:**
  - Blog index page (`/community/blog`)
  - Dynamic blog post pages (`/community/blog/[slug]`)
  - Blog posts stored in Supabase `blog_posts` table
  - Basic admin page for creating/editing posts (enhanced in Phase 6)
- [ ] Geographic SEO pages: `/newburgh-preschool`, `/orange-county-montessori`

### Week 4: SEO, Performance, DNS Cutover

- [ ] Implement all JSON-LD structured data (see SEO_AEO.md):
  - EducationalOrganization + LocalBusiness (site-wide)
  - FAQPage (FAQ page + contextual FAQ sections)
  - BreadcrumbList (all interior pages)
  - BlogPosting (each blog post)
- [ ] Create `llms.txt` in `/public` (see SEO_AEO.md)
- [ ] Configure `next-sitemap` — auto-generate sitemap.xml and robots.txt
- [ ] Implement 301 redirect map from Squarespace URLs (see MIGRATION.md)
- [ ] Run Lighthouse audit — target 90+ all categories
- [ ] Optimize images: all through `next/image`, blur placeholders, AVIF/WebP
- [ ] Add Google Analytics 4 + Google Search Console verification
- [ ] **DNS CUTOVER:** Switch domain from Squarespace to Vercel
- [ ] Verify production site: SSL, all pages load, redirects work, mobile OK
- [ ] Submit sitemap to Google Search Console
- [ ] Announce new site to current families (email via existing channels)

**Phase 1 Complete:** Public website is live, replacing Squarespace. Lead capture active. Blog publishable. All SEO/AEO in place.

---

## Phase 2: Admissions (Weeks 5–8)

**Goal:** Tour scheduling, enrollment pipeline with statuses, automated email follow-up sequences. Replace Calendly.

### Week 5: Inquiry System + Lead Pipeline

- [ ] Build inquiry form (public, distinct from contact form per Rebecca's requirement)
- [ ] Create `enrollment_pipeline` table with statuses: `inquiry` → `tour_scheduled` → `tour_completed` → `application_submitted` → `application_review` → `accepted` → `enrolled` → `waitlisted` → `withdrawn`
- [ ] Admin pipeline view: Kanban board showing all leads by status
- [ ] Inquiry submission triggers:
  - Insert into `enrollment_pipeline`
  - Send confirmation email to parent (Resend)
  - Send notification to Rebecca (email + optional SMS)
  - Start lead nurture sequence (see CONTENT_STRATEGY.md)
- [ ] Lead detail page: family info, notes, status history, communication log

### Week 6: Tour Scheduling

- [ ] Tour booking page (`/admissions/schedule-tour`)
- [ ] Google Calendar API integration:
  - Fetch Rebecca's availability
  - Display available time slots
  - Book → create Calendar event + update pipeline status
- [ ] Tour confirmation email (with calendar .ics attachment)
- [ ] Tour reminder email (24 hours before)
- [ ] Post-tour follow-up email sequence (see CONTENT_STRATEGY.md)
- [ ] Admin calendar view: upcoming tours, past tours

### Week 7: Application Form

- [ ] Multi-step enrollment application form:
  - Step 1: Parent/guardian information
  - Step 2: Child information (name, DOB, medical, allergies)
  - Step 3: Emergency contacts
  - Step 4: Program selection + schedule preference
  - Step 5: Additional info (how heard about us, special needs, dietary)
  - Step 6: Document uploads (immunization records, prior school records)
  - Step 7: Review + submit
- [ ] Save progress (draft applications persist)
- [ ] Application submission updates pipeline status
- [ ] Admin review workflow: view application, add notes, approve/waitlist/request more info

### Week 8: Automated Follow-Up + Pipeline Polish

- [ ] Email sequence automation engine (triggered by pipeline status changes)
- [ ] Lead nurture sequence: 5 emails over 30 days (see CONTENT_STRATEGY.md)
- [ ] Post-tour sequence: 3 emails over 7 days
- [ ] Acceptance notification email (manual trigger by Rebecca)
- [ ] Waitlist management: position tracking, notification when spot opens
- [ ] Admin dashboard: pipeline analytics (conversion rates by stage, time in stage)
- [ ] Retire Calendly: update all links, cancel subscription

**Phase 2 Complete:** Full admissions funnel from inquiry to enrollment. Calendly replaced.

---

## Phase 3: Parent Portal (Weeks 9–14)

**Goal:** Authenticated parent experience — dashboard, calendar, forms, messaging, photos, child profiles. Replace Google Forms and paper processes. Begin replacing Brightwheel communication.

### Week 9: Authentication + Family Profiles

- [ ] Supabase Auth setup: email/password + magic link + optional Google SSO
- [ ] Role-based access: `parent`, `staff`, `admin`, `enrichment`, `board`
- [ ] Middleware: route protection for `/portal/*`, `/admin/*`, `/staff/*`
- [ ] Parent invitation system: admin generates invite link → parent clicks → creates account → linked to family
- [ ] Family profile setup: parents, children, emergency contacts, medical info
- [ ] Child profile: photo, classroom assignment, enrollment date, allergies, notes

### Week 10: Parent Dashboard + Calendar

- [ ] Parent dashboard (`/portal`):
  - Welcome message with child's name and classroom
  - Today's schedule / upcoming events
  - Action items (forms to complete, payments due)
  - Recent messages
  - Quick links to common actions
- [ ] School calendar view (portal only — NEVER public, per Rebecca's requirement):
  - Monthly calendar with school events, closures, enrichment schedule
  - Google Calendar 2-way sync
  - Filter by classroom, event type
  - Subscribe to personal calendar (iCal export)

### Week 11: Forms System

- [ ] Admin form builder: create custom forms with field types (text, select, checkbox, date, file upload, signature)
- [ ] Form types: permission slips, field trips, before/after care signup, volunteer, conference request
- [ ] Parent form inbox: see assigned forms, complete, submit
- [ ] Form submission → notification to admin
- [ ] Form response tracking in admin dashboard
- [ ] Replace Google Forms: recreate all active forms in new system

### Week 12: Messaging + Notifications

- [ ] Parent-teacher messaging (threaded, per-child)
- [ ] Admin broadcast messages (all families, per-classroom, per-program)
- [ ] Push notifications (web push via service worker)
- [ ] Email notifications for new messages
- [ ] SMS for emergencies only (Twilio integration)
- [ ] Notification preferences: parents choose email, push, both, or none per category

### Week 13: Photo System + Daily Reports

- [ ] Staff photo upload (classroom photos, activity shots)
- [ ] Photo auto-processing: Sharp pipeline strips EXIF, resizes, generates thumbnails
- [ ] Photo gallery in parent portal (visible only to parents of children in that classroom)
- [ ] Daily report system: staff creates daily summary with photos, activities, notes
- [ ] Parents see daily reports in portal, receive email/push notification
- [ ] Replace Slack photo channel workflow

### Week 14: Portal Polish + Staff Tools (Basic)

- [ ] Attendance tracking: staff marks daily attendance (present, absent, late, early pickup)
- [ ] Incident report form: staff creates → admin reviews → parent notified
- [ ] Portal onboarding wizard: first-time parent login experience
- [ ] Empty states for all portal sections (warm, encouraging copy)
- [ ] Loading states: skeleton loaders matching content shape
- [ ] Error boundaries with friendly recovery UI
- [ ] Mobile optimization: all portal features work at 375px

**Phase 3 Complete:** Parent portal functional. Google Forms and paper processes replaced. Brightwheel communication partially replaced.

---

## Phase 4: Billing (Weeks 15–18)

**Goal:** Stripe-powered billing — tuition subscriptions, add-ons, one-time payments, invoicing, financial reporting. Replace Google Sheets billing.

### Week 15: Stripe Integration + Product Setup

- [ ] Stripe Connect or direct integration setup
- [ ] Create Stripe products/prices:
  - Monthly tuition per classroom (recurring subscription)
  - Before care add-on (recurring)
  - After care add-on (recurring)
  - Application fee (one-time)
- [ ] Webhook handler (`/api/webhooks/stripe`) for payment events
- [ ] Stripe Customer created for each family on enrollment

### Week 16: Parent Payment Experience

- [ ] Payment dashboard in portal: current balance, upcoming charges, payment history
- [ ] Autopay setup (Stripe subscription with saved payment method)
- [ ] One-time payment option (for families who prefer manual)
- [ ] Payment method management (add/update card)
- [ ] Payment receipt emails (via Stripe + custom Resend template)
- [ ] Failed payment handling: notification → grace period → Rebecca alerted

### Week 17: Admin Financial Dashboard

- [ ] Invoice management: generate, send, track, mark paid (manual overrides)
- [ ] Revenue dashboard: monthly revenue, outstanding balances, payment trends
- [ ] Per-family financial view: payment history, balance, subscription status
- [ ] Tuition adjustment system: discounts, scholarships, sibling rates, work-trade offsets
- [ ] Before/after care billing: track daily usage → generate monthly invoice

### Week 18: Financial Reports + Compliance

- [ ] Monthly financial report: revenue, expenses categorized, outstanding AR
- [ ] Tax document generation: year-end statement per family (tuition paid for tax deduction)
- [ ] Export financial data to CSV (for Rebecca's accountant)
- [ ] Stripe dashboard link from admin (quick access to Stripe's native reporting)
- [ ] Retire Google Sheets billing: archive old sheets

**Phase 4 Complete:** Full billing system. Google Sheets billing tracking replaced. Parents can autopay.

---

## Phase 5: Operations (Weeks 19–22)

**Goal:** Compliance tracking (OCFS), staff management, work-trade program, grant tracking, operational reporting. Complete Brightwheel replacement.

### Week 19: Compliance Tracking (OCFS)

- [ ] Immunization records: per-child tracking with expiry dates, document uploads
- [ ] Staff certification tracking: CPR/First Aid, background checks, training hours
- [ ] Classroom ratio monitoring: real-time staff:child ratio based on attendance
- [ ] Inspection readiness checklist: all required docs accessible from one page
- [ ] Compliance alerts: expiring immunizations, certifications, upcoming inspections

### Week 20: Staff Management

- [ ] Staff profiles: credentials, certifications, training hours, classroom assignments
- [ ] Staff schedule: who's in which classroom, coverage planning
- [ ] Training log: record professional development hours, upload certificates
- [ ] Staff → admin communication: internal messaging (not visible to parents)
- [ ] Enrichment teacher portal: schedule, session attendance, hour logging

### Week 21: Work-Trade + Grants

- [ ] Work-trade family management: agreement terms, tuition offset structure
- [ ] Hour tracking: family logs hours, admin verifies, running total vs. expected
- [ ] Grant tracking: applications submitted, awarded amounts, reporting deadlines, deliverables
- [ ] Budget view: income sources breakdown (tuition, grants, donations, events)

### Week 22: Reporting + Full Brightwheel Retirement

- [ ] Enrollment report: current enrollment by classroom, waitlist length, conversion funnel
- [ ] Attendance report: daily/weekly/monthly, per-child absence tracking
- [ ] Financial summary: board-ready report with enrollment + revenue + expenses
- [ ] Custom report builder: select date range, metrics, export to PDF/CSV
- [ ] **Brightwheel migration:** import historical data, transition all families (see MIGRATION.md)
- [ ] **Cancel Brightwheel subscription**

**Phase 5 Complete:** All operations digitized. Brightwheel fully replaced. Rebecca's admin hours: target ~12/week.

---

## Phase 6: Advanced (Weeks 23–26)

**Goal:** AI chatbot, newsletter builder, event management, donation system, summer camp module.

### Week 23: AI Chatbot

- [ ] Public AI chatbot (Claude Haiku) — see AI_INTEGRATION.md:
  - RAG pipeline with approved school information
  - Brand voice enforcement via system prompt
  - Guardrails: never reveal tuition, teacher names, calendar
  - Lead capture: "Would you like to schedule a tour?"
  - Escalation: "Let me connect you with Rebecca"
- [ ] Admin content assist:
  - Blog post draft generation from bullet points
  - Email copy suggestions
  - SEO title/description recommendations
  - Event description generator

### Week 24: Newsletter Builder + Event System

- [ ] Admin newsletter builder:
  - Drag/drop sections: header, text, photos, events, CTA
  - Audience segmentation: current families, prospects, community, alumni
  - Preview + test send
  - Schedule or send immediately
  - Sent newsletter archive
- [ ] Event management system:
  - Create events: open house, field trip, holiday celebration, community event
  - RSVP tracking (portal for families, public form for community events)
  - Google Calendar sync (events appear in school calendar)
  - Event page on public site (`/community/events/[slug]`)
  - Post-event photo gallery

### Week 25: Donations + Camp Registration

- [ ] Donation system:
  - Public donation page (`/donate` — linked from community section)
  - Stripe one-time and recurring donations
  - Donation tracking in admin dashboard
  - Tax receipt emails (auto-generated)
  - Donor recognition (opt-in, displayed on site)
- [ ] Summer camp registration module:
  - Camp-specific landing page (`/programs/summer-camp`)
  - Session selection + registration form
  - Separate payment flow (one-time, per-session)
  - Camp-specific parent portal view
  - Camp waitlist if sessions fill up

### Week 26: Polish + Launch Review

- [ ] End-to-end testing: complete user journey from discovery → enrollment → daily portal use
- [ ] Accessibility audit: WCAG 2.1 AA on all pages
- [ ] Performance audit: Lighthouse 90+ on all public pages
- [ ] Security review: RLS policies, auth flows, data exposure check
- [ ] Admin training: document all admin workflows with screenshots
- [ ] Print-ready: incident reports, invoices, tax statements, conference summaries
- [ ] Retire remaining tools: Canva newsletter, Sign-Up Genius

**Phase 6 Complete:** All 9 tools fully replaced. Advanced features live.

---

## Phase 7: Iteration (Ongoing)

No fixed timeline — prioritize based on Rebecca's needs and user feedback.

### Bilingual Support (i18n)

- [ ] Locale files for all UI strings (`en`, `es`)
- [ ] CMS supports bilingual blog posts and page content
- [ ] Chatbot detects Spanish → responds in Spanish
- [ ] Critical forms bilingual: enrollment, emergency contact, permission slips
- [ ] Email templates with Spanish variants
- [ ] Public site language toggle

### Instagram Integration

- [ ] Instagram API: pull recent posts for community section
- [ ] Admin approval before displaying on site
- [ ] Auto-generate blog post draft from Instagram caption + photos

### Google Business Profile API

- [ ] Auto-post blog posts and events to GBP
- [ ] Pull and display Google reviews on site
- [ ] Review response from admin dashboard

### Digital Yearbook

- [ ] End-of-year photo book: auto-generated from curated photos
- [ ] Per-child yearbook page with highlights
- [ ] Print-ready PDF export
- [ ] Digital shareable version for families

### Analytics Dashboard

- [ ] Admin analytics page: traffic, conversions, popular pages, chatbot usage
- [ ] Enrollment funnel visualization
- [ ] Revenue trends and forecasting
- [ ] Parent portal engagement metrics

---

## Dependency Map

```
Phase 1 (Foundation)
  └── Phase 2 (Admissions) — requires: public site, inquiry form, email
       └── Phase 3 (Portal) — requires: auth, Supabase schema, email
            └── Phase 4 (Billing) — requires: family profiles, Stripe
                 └── Phase 5 (Operations) — requires: attendance, compliance tables
  └── Phase 6 (Advanced) — requires: all above + AI, newsletter, events
       └── Phase 7 (Iteration) — requires: stable platform
```

**No phase can start until the previous phase is complete and verified.** The exception is that some Phase 5 compliance work can begin in parallel with Phase 4 billing, since they use different database tables.

---

## Success Metrics by Phase

| Phase | Key Metric | Target |
|---|---|---|
| 1 | Site live, Lighthouse scores, pages indexed | 90+ Lighthouse, 100% pages in Search Console |
| 2 | Tour booking rate from inquiries | 15%+ conversion |
| 3 | Parent portal adoption | 90%+ active within 30 days of launch |
| 4 | Autopay enrollment | 70%+ families on autopay within 60 days |
| 5 | Rebecca's weekly admin hours | ≤ 12 hours (down from ~37) |
| 6 | AI chatbot lead capture | 10%+ of chatbot conversations → inquiry form |
| 7 | Bilingual content coverage | 80%+ of public pages in both languages |
