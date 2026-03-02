# Product Requirements Document — Little Friends Learning Loft

> **Version:** 2.0 | **Date:** March 2026 | **Author:** Joshua (Developer) for Rebecca Harrison (Founder)

## Executive Summary

Little Friends Learning Loft (LFLL) is a Montessori preschool for ages 3–6 at the Newburgh Jewish Community Center (290 North St, Newburgh, NY 12550). Founded by Rebecca Harrison, the school serves ~50+ children across three classrooms with a summer camp program.

This project replaces the entire digital infrastructure with a single custom web application serving three audiences: the public (prospective families), authenticated parents (enrolled families), and administrators (Rebecca and staff).

### Systems Being Replaced

| Current Tool | Replacement Feature |
|---|---|
| Squarespace | Public website with SEO, AEO, blog, lead capture |
| Brightwheel | Parent comms, attendance, billing, daily reports |
| Calendly | Tour scheduling, consultations, enrichment coordination |
| Google Forms | Before/after care signup, field trip permissions, volunteer signups |
| Google Sheets | Waitlist tracking, tuition add-ons, aftercare billing, work-trade hours, immunization tracking |
| Paper processes | Incident reports, conference forms, enrollment paperwork |
| Slack (photo channel) | Photo management and curation system |
| Canva newsletter | Newsletter builder/distribution |
| Sign-Up Genius | Conference scheduling, volunteer slots |

### Key Success Metrics

- Tour booking rate: 15%+ of inquiry form submissions
- Time from inquiry to first automated follow-up: under 2 minutes
- Parent portal adoption: 90%+ of enrolled families active within 30 days
- Rebecca's weekly admin hours: reduced from ~37 to ~12
- Google review growth: 2+ new reviews per month
- Blog publishing: 2+ posts per month
- AEO visibility: LFLL mentioned when AI engines answer "best preschool in Newburgh NY"

## User Roles

### Public Visitor (Unauthenticated)
Prospective parents researching preschools, extended family, community members interested in events, potential donors, potential employees.

### Parent/Guardian (`parent`)
Enrolled family members. Access to parent portal: dashboard, calendar, children profiles, forms, payments, messaging, signups, resources, notification preferences.

### Staff/Teacher (`staff`)
Lead and assistant teachers. Manage attendance, incident reports, upload photos, communicate with parents, view own training/certs. Cannot see billing.

### Enrichment Teacher (`enrichment`)
External contractors (art, yoga/dance, music, skateboarding). View own schedule, mark attendance, log hours. Limited portal access.

### Administrator (`admin`)
Rebecca Harrison (primary). Full access to everything: finances, enrollment, staff, compliance, CMS, communications. Dashboard with real-time school operations.

### Board Member (`board`)
Read-only access to financial reports, enrollment data, compliance status.

## Critical Non-Negotiables (Rebecca's Requirements)

1. No tuition/fee amounts on the public website — pricing shared only after tour/inquiry
2. No teacher bios on public site — only Rebecca's bio (staff privacy/safety)
3. Children's faces can be shown but never with full names or identifying info
4. School calendar is NEVER public — portal-only (safety)
5. Yellow brand color (#F5C518) must be preserved
6. Fonts must feel "playful" — not corporate
7. Testimonials sprinkled throughout the site, not a dedicated page
8. Step-by-step clarity for every process (enrollment, camp signup, etc.)
9. The site reflects that LFLL is "not just education — it's very social for families and caregivers"
10. Contact form and enrollment inquiry form must be clearly differentiated

## Information Architecture

### Public Website
```
/ (Homepage — scroll-based)
├── /about/rebecca, /about/montessori, /about/jcc, /about/mission
├── /programs/primary, /programs/kindergarten, /programs/enrichment
│   /programs/before-after-care, /programs/clubs
├── /admissions/process, /admissions/tour, /admissions/apply
│   /admissions/faq, /admissions/readiness
├── /summer-camp/overview, /summer-camp/weekly-themes, /summer-camp/register
├── /community/events, /community/workshops, /community/blog
│   /community/press, /community/donate
├── /careers/openings, /careers/apply
├── /contact, /privacy, /terms, /newsletter-signup
```

### Authenticated Portals
```
/portal — Parent Portal
  /dashboard, /calendar, /children, /forms, /payments
  /communication, /signup, /resources, /settings

/admin — Admin Dashboard
  /dashboard, /enrollment, /families, /children, /staff
  /billing, /scheduling, /communications, /content, /events
  /reports, /compliance, /grants, /work-trade, /inventory
  /press, /settings

/staff — Staff Portal
  /dashboard, /classroom, /photos, /incidents, /training
  /messages, /resources
```

## Homepage Sections (Scroll-Based)

1. **Hero** — Full-width photo/video, warm gradient overlay, "Schedule a Tour" CTA, "Trusted by 50+ families since 2018" badge
2. **What Makes Us Special** — 4 feature cards: "Two Teachers, Twelve Children", "Learning by Doing", "An Entire Campus to Explore", "No Rushing Childhood"
3. **A Day at Little Friends** — Photo gallery, day-in-the-life narrative
4. **Programs at a Glance** — Cards linking to each program
5. **Admissions Path** — 3-step visual stepper with animated connecting lines
6. **Community & Events** — Upcoming events, blog previews, email signup
7. **Our Home at the JCC** — Photo grid, JCC partnership note, Google Map
8. **Footer** — Contact info, quick links, newsletter signup, legal

## Parent Portal Features

- First-login onboarding flow (5-step guided setup)
- Family dashboard with Today's Snapshot, Action Items, Quick Actions
- Private school calendar with color-coded events and iCal export
- Per-child profiles with documents, immunizations, special services, reports
- Digital fillable forms with e-signatures and deep-linking from notifications
- Stripe billing: invoices, autopay, add-on payments, year-end statements
- Private 1-to-1 messaging (not visible to all staff like Brightwheel)
- Signups: before/after care, field trips, clubs, volunteers, conferences, consultations
- Parent resources: handbook, supply lists, EIN for tax deductions
- Notification preferences: channels, categories, quiet hours, per-child settings

## Admin Dashboard Features

- Morning Briefing mode (single-scroll daily overview at 7:30am)
- CRM-style enrollment pipeline (Kanban: Inquiry → Tour → Application → Enrolled)
- 7-step automated lead follow-up email sequences
- Family directory with complete records and communication history
- Student records with immunization tracker and 5-year digital archive
- Staff management with OCFS training matrix (10 topic areas, hour tracking)
- Stripe billing center with invoice management, financial dashboard, YoY comparison
- Google Calendar 2-way sync with conflict detection
- Mass communications hub (email, SMS, push, emergency broadcast)
- CMS: blog editor with AEO fields, page editor, photo curation pipeline, newsletter builder
- Reports: enrollment, financial, compliance, auto-generated board reports (PDF)
- Compliance: OCFS training, immunizations, safety drills, ratio monitoring, document retention
- Grant tracker, work-trade program tracker, inventory management, press/PR tools

## Staff Portal Features

- Daily schedule and classroom assignment
- Digital attendance (check in/out with timestamps)
- Photo upload with auto EXIF stripping and curation queue
- Digital incident report form (auto-sends to admin, prints cleanly)
- Personal training dashboard with OCFS progress
- 1-to-1 messaging with parents and admin

## AI Chatbot

- Warm, knowledgeable, conversational personality — like a helpful enrolled parent
- Knows: FAQs, programs, admissions process, hours, location, contact info
- Never shares: pricing, staff personal info, student data
- Deflects tuition questions to tour scheduling
- Routes unanswerable questions to Rebecca
- Bottom-right floating panel, not full-screen overlay
- Quick-reply chips: "What ages?", "Tour times", "Programs", "Talk to Rebecca"
- "Talk to a human" option always visible
- Clearly identifies itself as AI

## Integration Requirements

- **Google Calendar API**: 2-way sync with Rebecca's calendar, building calendar, enrichment calendars
- **Stripe**: Subscriptions (tuition autopay), one-time charges, refunds, webhooks, fee tracking
- **Resend**: Transactional + marketing email with templates and sequences
- **Twilio**: Emergency broadcast, appointment reminders, business phone number
- **Claude API**: Public chatbot + admin content assist (blog drafts, social captions, press releases)
- **Google Maps API**: Embedded map on contact page
- **Sharp**: Photo processing pipeline (compress, EXIF strip, multi-size, WebP/AVIF)

## Accessibility Requirements

- WCAG 2.1 AA compliance
- 4.5:1 contrast ratio minimum on all text
- Yellow (#F5C518) never used for text — backgrounds/borders/decorative only
- Keyboard navigation with logical tab order and focus indicators
- Skip navigation link on every page
- Descriptive alt text on all images
- `prefers-reduced-motion` respected globally
- `aria-live` for form errors, `aria-busy` for loading states
- Print stylesheets for incident reports, invoices, conference reports, board reports
- Bilingual architecture support (Spanish) with `lang` attributes

## Error, Loading & Empty States

- **Loading**: Skeleton loaders matching content shape (not spinners), warm gray tones
- **Error**: Friendly messages with branded illustrations, retry options, never raw errors
- **Empty**: Warm, encouraging copy ("No forms to sign right now — you're all caught up!"), hand-drawn illustration accents, action suggestions
