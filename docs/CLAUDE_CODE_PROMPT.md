# CLAUDE_CODE_PROMPT.md — Kickoff Prompt for Claude Code

> Copy and paste this entire prompt into Claude Code to begin building LFLL.

---

## Prompt

You are building **Little Friends Learning Loft** — a full-stack Next.js web application for a Montessori preschool in Newburgh, NY. This app replaces Squarespace, Brightwheel, Calendly, Google Forms, Google Sheets, Slack photo workflows, Canva newsletter, and Sign-Up Genius with a single unified platform.

### Before you write any code, read these files in order:

**Root files:**
1. `CLAUDE.md` — Single source of truth (start here, references everything else)

**Documentation suite** (`docs/` directory) — Read all 16 documents:

2. `docs/PRD.md` — Product requirements, user personas, feature priority matrix
3. `docs/ARCHITECTURE.md` — System architecture, project structure, data flow
4. `docs/FRONTEND.md` — Page-by-page UI specs, component hierarchy, routing
5. `docs/BACKEND.md` — API route patterns, middleware, Supabase client setup
6. `docs/DESIGN_SYSTEM.md` — Brand colors, typography, motion tokens, component specs
7. `docs/DATA_MODEL.md` — Complete database schema, RLS policies, indexes, triggers
8. `docs/SECURITY.md` — Auth flows, RLS rules, EXIF stripping, COPPA compliance
9. `docs/API_INTEGRATIONS.md` — Stripe, Resend, Twilio, Google Calendar, Google Maps, Sharp
10. `docs/AI_INTEGRATION.md` — Claude API chatbot, prompt engineering, safety rails
11. `docs/ADMIN_DASHBOARD.md` — Rebecca's admin interface, enrollment pipeline, reports
12. `docs/SEO_AEO.md` — Keywords, structured data schemas, llms.txt, AEO formatting, performance targets
13. `docs/CONTENT_STRATEGY.md` — Brand voice, tone, FAQ patterns, blog strategy, email sequences
14. `docs/MIGRATION.md` — Tool-by-tool migration from Squarespace, Brightwheel, Google Sheets, etc.
15. `docs/DEPLOYMENT.md` — Vercel + Supabase config, env variables, CI/CD, third-party service setup
16. `docs/ROADMAP.md` — 7-phase build plan with week-by-week checklists and dependencies

**Read order for Phase 1:** Start with CLAUDE.md → PRD → ARCHITECTURE → DESIGN_SYSTEM → DATA_MODEL → FRONTEND → BACKEND → SECURITY → SEO_AEO → ROADMAP. Reference the others as needed per phase.

### Your first task is Phase 1 — Foundation (Weeks 1–4):

**1. Project Scaffold**
```bash
npx create-next-app@latest lfll --typescript --tailwind --eslint --app --src-dir --import-alias "@/*"
cd lfll
```

**2. Install Dependencies**
```bash
# Core
npm install @supabase/supabase-js @supabase/ssr zustand

# UI
npm install @radix-ui/react-* class-variance-authority clsx tailwind-merge lucide-react

# Animation
npm install framer-motion

# Forms & Validation
npm install react-hook-form @hookform/resolvers zod

# Image Processing
npm install sharp

# Email
npm install resend

# Utilities
npm install date-fns slugify
```

Then initialize shadcn/ui:
```bash
npx shadcn-ui@latest init
```

**3. Configure Tailwind with LFLL Design Tokens**

Extend `tailwind.config.ts` with the brand colors from `docs/DESIGN_SYSTEM.md`:
- Primary Yellow `#F5C518`, Secondary Yellow `#FFF3C4`, Background `#FFFDF7`
- Text colors, accent colors, border colors
- Font families: Fredoka One (headings), Nunito (body), JetBrains Mono (admin data)
- Border radius: `rounded-xl` (12px) default for cards, `rounded-2xl` (16px) for images
- Motion tokens as CSS custom properties

**4. Set Up Supabase**

Create the database schema from `docs/DATA_MODEL.md`. Start with these Phase 1 tables:
- `users`, `families`, `family_members`, `children`, `classrooms`
- `leads`, `tour_bookings`, `email_sequence_progress`
- `blog_posts`, `faqs`, `pages`, `photos`
- `audit_log`, `school_settings`
- All RLS policies per `docs/SECURITY.md`
- Auto-updating timestamp triggers
- Critical indexes for query performance

**5. Build the Public Website**

Follow `docs/FRONTEND.md` and `docs/DESIGN_SYSTEM.md` for component specs. Build these pages:

**Homepage** (`/`) — Scrolling single-page experience:
- Hero with warm gradient overlay, animated heading, "Schedule a Tour" CTA, social proof badge
- "What Makes Us Special" — 4 feature cards with hand-drawn SVG icons
- "A Day at Little Friends" — photo gallery with warm color grading
- "Programs at a Glance" — cards linking to program pages
- "Admissions — Your Path to Little Friends" — 3-step visual stepper with connecting lines
- Community & Events feed
- Our Home at the JCC — photo grid + Google Map embed
- Footer with contact info, quick links, newsletter signup

**About Section** (`/about/*`):
- Rebecca's Story, What is Montessori (AEO-structured Q&A), Our Home at the JCC, Mission & Values

**Programs** (`/programs/*`):
- Primary Classroom, Kindergarten, Enrichment, Before/After Care, Clubs

**Contact** (`/contact`):
- General inquiry form (NOT enrollment) with click-to-call, embedded map

**Static Pages:**
- Privacy Policy, Terms of Use, 404 page (warm + playful)

**6. Implement Design System Components**

Build these reusable components per `docs/DESIGN_SYSTEM.md`:
- `HeroSection` — full-width with gradient overlay and staggered entrance animation
- `StepCard` — numbered circle + title + description + animated connecting line
- `TestimonialCard` — quote + name + yellow accent line
- `FeatureCard` — SVG icon + heading + description
- `PhotoGallery` — masonry layout, lightbox, lazy loading, warm color grade
- `CTABanner` — yellow background with gradient
- `FAQAccordion` — expandable with FAQ schema markup
- `MobileStickyBar` — bottom bar with Call | Book Tour | Chat (appears on scroll)
- `MotionReveal` — Framer Motion wrapper for scroll-triggered animations

**7. Set Up SEO/AEO**

Per `docs/SEO_AEO.md`:
- JSON-LD structured data on every public page (EducationalOrganization, LocalBusiness, BreadcrumbList)
- XML sitemap, robots.txt (exclude portal/admin/staff)
- Meta tags, Open Graph, Twitter Cards
- FAQ schema on embedded FAQ sections
- Question-forward H2 headings for AEO

**8. Image Optimization Pipeline**

Per `docs/ARCHITECTURE.md`:
- Sharp processing: strip EXIF/GPS → resize to 400w/800w/1200w/1600w → WebP/AVIF/JPEG
- Next.js Image component with responsive sizes and blur placeholders
- Hero image preloaded in `<head>`
- Max upload size 5MB, compressed on upload

**9. Basic Admin**

Minimal admin for Phase 1:
- Lead inbox (view form submissions)
- Blog editor (rich text, SEO fields, draft/publish)
- Content editor for public pages
- Photo upload with auto-processing

**10. Mobile Sticky CTA Bar**

High-impact conversion feature:
- Bottom bar: 📞 Call | 📅 Book Tour | 💬 Chat
- Appears after scrolling past hero
- Hides on scroll down (reading), reappears on scroll up
- All public pages

### Key architectural decisions:

- **Public pages are SSG** with ISR for content changes — maximum performance and SEO
- **Blog uses ISR** — static at build, revalidates on new post
- **Portal/admin pages are SSR** with Supabase session validation
- **API Route Handlers** for all mutations
- **Supabase Realtime** for live features (attendance, messages, dashboard)
- **Optimistic UI** for common actions with server reconciliation
- **PWA manifest** from Phase 1 (install to home screen)

### Critical constraints:

- All text on yellow backgrounds uses `#2D2926` — 4.5:1 contrast minimum
- Never use yellow `#F5C518` for text — only for backgrounds, borders, decorative elements
- Every interactive element has a visible focus indicator: `2px solid #5B8DB8` with 2px offset
- All photos auto-strip EXIF/GPS metadata — this is a school with children
- `prefers-reduced-motion` respected globally — replace all animations with simple opacity fades
- Max 3 simultaneous animations in viewport
- Scroll animations trigger once only (`once: true`) — never reset on scroll up
- Touch targets minimum 44x44px
- Max line length 65ch on text blocks
- Loading states use skeleton loaders, never spinners
- Empty states use warm encouraging copy, never "No data found"

### After Phase 1, continue to Phase 2 by re-reading:
- `docs/ROADMAP.md` for Phase 2 scope
- `docs/BACKEND.md` for API route patterns
- `docs/API_INTEGRATIONS.md` for Google Calendar integration
- `docs/ADMIN_DASHBOARD.md` for enrollment pipeline specs

---

*Read `CLAUDE.md` now to begin.*
