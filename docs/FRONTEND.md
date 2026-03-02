# Frontend Specification — Little Friends Learning Loft

## Rendering Approach

- **Server Components by default** — use `'use client'` only for interactivity (forms, animations, state)
- **Mobile-first** — all layouts start from mobile (375px) and scale up
- **Touch targets minimum 44x44px** — all clickable elements
- **Tailwind for all styling** — no inline styles, no CSS modules
- **shadcn/ui as component base** — customized with LFLL design tokens

## Responsive Breakpoints

```
sm:  640px    (large phones, landscape)
md:  768px    (tablets)
lg:  1024px   (small laptops)
xl:  1280px   (desktops)
2xl: 1536px   (large desktops)
```

## Public Website — Page-by-Page Specs

### Homepage (`/`) — Scroll-Based Sections

**Section 1: Hero**
- Full-width photo or auto-playing muted video loop
- Warm gradient overlay: transparent → yellow-tinted at 10% opacity
- Headline copy (choose one):
  - Option A: "Mud-stained hands. Grass-stained knees. Bright, curious minds."
  - Option B: "They'll paint in the art house. Dance in the studio. Dig in the garden. And learn everything along the way."
  - Option C: "A place where small people do big things — at their own pace."
- Subtitle: "Montessori [learning through play / preschool for ages 3–6] at the Newburgh JCC"
- Primary CTA: "Schedule a Tour" → `/admissions/tour`
- Secondary CTA: "Explore Our School" → smooth scroll to Section 2
- Social proof badge: "Trusted by 50+ families since 2018"
- Subtle downward scroll indicator animation (bouncing chevron)
- **Entrance animation**: heading fades up 600ms → subtitle staggered 150ms → CTA staggered 300ms

**Section 2: What Makes Us Special**
4 feature cards in responsive grid (1-col mobile, 2-col tablet, 4-col desktop):
1. "Two Teachers, Twelve Children" — Every child is known, seen, and heard.
2. "Learning by Doing" — Hands in soil, letters traced in sand, math learned with real objects.
3. "An Entire Campus to Explore" — Gym, art house, dance studio, three playgrounds, garden.
4. "No Rushing Childhood" — Follow each child's natural rhythm.

Each card: hand-drawn SVG icon, heading (Fredoka One), description (Nunito), rounded corners (16px).
Testimonial quote below grid. Cards animate with staggered fade-up on scroll.

**Section 3: A Day at Little Friends**
- Horizontal scrolling photo gallery OR video embed
- Brief warm narrative of a typical day
- Call-out: "Our 3 classrooms serve ages 3–6 in a multi-age Montessori environment"

**Section 4: Programs at a Glance**
- Cards linking to: Primary, Kindergarten, Before/After Care, Summer Camp, Enrichment
- Each card: photo (warm color grade, rounded 16px), brief description
- Contextual testimonial

**Section 5: Admissions Path**
- 3-step visual stepper with animated connecting lines on scroll:
  1. "Tell Us About Your Family" → inquiry form
  2. "Come See Us in Action" → tour booking
  3. "Join the Family" → application
- Each step is clickable → links to relevant page
- Seasonal enrollment status banner
- Testimonial about the enrollment experience

**Section 6: Community & Events**
- Upcoming events feed (next 3 events)
- Blog post previews (latest 2 posts)
- "Join Our Community" email signup form (name, email)
- Photo collage of community moments

**Section 7: Our Home at the JCC**
- Photo grid of JCC spaces LFLL uses
- Brief text about the partnership
- Note: "Little Friends is a secular program housed within the Jewish Community Center"
- Embedded Google Map

**Section 8: Footer**
- Address: 290 North St, Newburgh, NY 12550
- Phone (click-to-call), email
- Social: Instagram, Facebook
- Quick links: Admissions, Programs, Contact, Blog, Careers, Donate
- Newsletter signup
- Privacy Policy, Terms of Use
- "© 2026 Little Friends Learning Loft, LLC. Licensed by NYS OCFS."
- JCC acknowledgment

### About Pages

**Rebecca's Story (`/about/rebecca`)**
- Hero photo of Rebecca in classroom/garden
- Rich text narrative (CMS-editable)
- Credentials: BFA from SVA, AMS certification from CMTE at College of New Rochelle
- Personal details: lives in Newburgh with husband and daughters Olive and Lou Eleanour
- Optional video embed
- Parent testimonial about Rebecca

**What is Montessori (`/about/montessori`) — AEO-Structured**
- Q&A structure with question-phrased H2s:
  - "What is Montessori education?"
  - "How does Montessori work at Little Friends?"
  - "What does a typical day look like?"
  - "How is Montessori different from traditional preschool?"
  - "Will my child be ready for kindergarten after Montessori?"
- Each section: direct 2-sentence answer first, then expand
- 3–4 embedded FAQ items with FAQ schema
- Photos of Montessori materials in action

**Our Home at the JCC (`/about/jcc`)**
- Photo gallery of all spaces: 3 classrooms, art house, dance/yoga studio, gym, library, social hall, 3 playgrounds, garden
- Brief description of each space
- JCC partnership note

**Mission & Values (`/about/mission`)**
- Mission statement, core values with visual treatment
- Montessori principles, tikkun olam values
- Multi-age grouping philosophy

### Programs Pages

**Primary Classroom (`/programs/primary`)**
- Multi-age grouping overview (Years 1, 2, 3)
- Daily schedule, enrichment subjects, photo gallery
- "What Your Child Will Learn" section, parent testimonial
- 3 FAQ items with FAQ schema

**Kindergarten (`/programs/kindergarten`)**
- Curriculum highlights, graduation process
- How LFLL prepares for first grade
- Evaluation reports sent to receiving schools

**Enrichment (`/programs/enrichment`)**
- Art, yoga/dance, music, skateboarding (seasonal)
- Each: brief description, schedule, photo

**Before & After Care (`/programs/before-after-care`)**
- Hours, description, signup process (links to portal for enrolled families)
- "Details shared upon enrollment" — no public pricing

**Clubs (`/programs/clubs`)**
- Seasonal offerings, signup process, past examples

### Admissions Pages (Critical Conversion Path)

**Admissions Process (`/admissions/process`)**
- Large visual stepper with HowTo schema markup
- 4 steps: Submit Inquiry → Schedule Tour → Submit Application → Welcome
- Each step: copy, form link, testimonial
- Enrollment status banner, link to readiness guide
- 3–4 embedded FAQ items

**Schedule a Tour (`/admissions/tour`) — Calendar-First**
- Show calendar widget FIRST — browse available slots before any form
- Tour types: Individual, Group (seasonal)
- After selecting slot → collect: parent name, child age, how they heard about LFLL
- Confirmation email with directions, parking, what to expect, what to wear
- Syncs to Rebecca's Google Calendar

**Application (`/admissions/apply`)**
- Multi-step form with progress indicator
- Fields: parent info, child info, enrollment year, program interest, before/after care, source, siblings, open questions
- Submission → lead pipeline entry, automated email sequence starts
- Dedicated confirmation/thank you page (not just a toast)

**FAQ (`/admissions/faq`) — AEO-Optimized**
- Accordion with FAQ schema on every Q&A pair
- Natural conversational question format
- Critical FAQs: religious affiliation, ratio, financial aid, hours, curriculum, allergies, COVID policy, waitlist, supplies
- Searchable, categorized, CMS-editable

**School Readiness (`/admissions/readiness`)**
- Developmental indicators as warm, encouraging checklist
- "Every child develops at their own pace — these are guides, not requirements"

### Other Public Pages

**Summer Camp** — Overview, weekly themes, registration (multi-step with Stripe payment). Camp pricing IS shown publicly.

**Events** — Eventbrite replacement: cards with RSVP, ticketed events via Stripe, Event schema.

**Blog** — CMS-powered, rich text, categories, AEO fields, BlogPosting schema, RSS feed.

**Donate** — Stripe: one-time + recurring, fund options, tax info, Amazon wishlist links.

**Careers** — Job listings with JobPosting schema, digital application form (multi-step), applications stored in admin.

**Contact** — General questions only (NOT enrollment). Form: name, email, message. Map, hours, social links.

## Navigation Components

### Glassmorphism Sticky Navbar
```css
backdrop-filter: blur(12px);
background: rgba(255, 253, 247, 0.8); /* warm off-white at 80% */
```
- LFLL logo (left)
- Minimal links: Programs, Admissions, About, Community, Contact
- "Book a Tour" CTA button (right)
- Mobile: hamburger → full-screen drawer

### Mobile Sticky CTA Bar
- Bottom bar: 📞 Call | 📅 Book Tour | 💬 Chat
- Appears after scrolling past hero
- Hides on scroll down (user is reading), reappears on scroll up
- Animation: slide up/down, 200ms, smooth easing
- `position: fixed; bottom: 0;` with proper z-index

## Form Design Patterns

### Multi-Step Forms (Stepper)
- Progress indicator at top showing current step
- Save-as-you-go (form data persists between steps)
- Back/Next navigation
- Step validation before advancing
- `aria-current="step"` on active step

### Form Field Behavior
- Labels always visible (not placeholder-only)
- Focus: border transitions to brand yellow + subtle glow (`box-shadow: 0 0 0 3px rgba(245,197,24,0.2)`)
- Errors: red border + message linked via `aria-describedby`
- Required: visual indicator + `aria-required="true"`

### Confirmation Pages
Every form submission gets a purpose-built thank you page:
- **Tour**: directions, parking, what to expect, what to wear, calendar event
- **Inquiry**: "We'll be in touch within 24 hours" + video/photos
- **Contact**: "We'll get back to you soon" + links to FAQ, programs

## Portal & Dashboard UI Patterns

### Parent Portal Layout
- Sidebar navigation (collapsible on mobile)
- Welcome banner with family name
- Today's Snapshot card (priority top position)
- Action Items (urgent cards)
- Quick Actions grid

### Admin Dashboard Layout
- Sidebar with grouped navigation sections
- Morning Briefing section (scrollable, no clicking required)
- Real-time operations below
- Progressive disclosure — surface important actions first, details on drill-down

### Emotional Design (Portal Only)
- Celebration micro-animations: confetti on last form submitted, checkmark on payment
- Seasonal background color shifts: amber in fall, green in spring, yellow in summer
- Warm encouraging empty states
- Skeleton loaders (not spinners)

## Image Handling

- All images: `next/image` component
- Above-fold: `priority={true}`, `loading="eager"`
- Below-fold: `loading="lazy"` (default)
- Hero: `<link rel="preload" as="image">` in head
- Sizes: responsive `sizes` attribute matching layout
- Photo treatment: slightly warm color grade, rounded corners (16px)
- Gallery: masonry layout, lightbox on click, hover scale (1.03) + shadow
- Parallax on hero/full-bleed photos: 0.3 ratio

## Accessibility Checklist (Per Page)

- [ ] Semantic heading hierarchy (h1 → h2 → h3, no skips)
- [ ] Skip navigation link
- [ ] All images have descriptive alt text
- [ ] Decorative images: `alt=""` + `aria-hidden="true"`
- [ ] All form inputs have associated `<label>`
- [ ] Focus indicators visible on all interactive elements
- [ ] Tab order matches visual order
- [ ] Modal dialogs trap focus
- [ ] Escape closes modals/dropdowns
- [ ] Loading states: `aria-busy="true"`
- [ ] Errors: `aria-live="polite"`
- [ ] Color contrast: 4.5:1 minimum
- [ ] Touch targets: 44x44px minimum
- [ ] `prefers-reduced-motion` respected
