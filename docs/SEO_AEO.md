# SEO_AEO.md — SEO, AEO & Performance

> Search visibility strategy for traditional search engines, AI answer engines, and local discovery. Read alongside CONTENT_STRATEGY.md for editorial planning and FRONTEND.md for implementation details.

---

## Traditional SEO

### On-Page Requirements (Every Public Page)

- Unique `<title>` tag (50–60 chars) — primary keyword near the beginning
- Unique `<meta name="description">` (150–160 chars) — compelling, includes CTA
- Canonical URL (`<link rel="canonical">`)
- Open Graph tags: `og:title`, `og:description`, `og:image` (1200×630), `og:url`, `og:type`
- Twitter Card meta: `twitter:card` (summary_large_image), `twitter:title`, `twitter:description`, `twitter:image`
- Semantic HTML: proper heading hierarchy (single H1), landmarks (`<main>`, `<nav>`, `<footer>`, `<article>`)
- Alt text on all images — descriptive and specific ("children working with Montessori counting beads in the Marigold classroom" not "kids playing")
- Internal links to 2–3 related pages with descriptive anchor text (never "click here")

### Title Tags & Meta Descriptions

| Page | Title Tag | Meta Description |
|---|---|---|
| Homepage | Little Friends Learning Loft · Montessori Preschool Newburgh NY | Nurturing Montessori preschool at Newburgh JCC. Ages 3–6, outdoor play, art studio, Jewish values. Small classes, big adventures. Schedule a tour. |
| Rebecca's Story | About Rebecca · Little Friends Learning Loft | Meet Rebecca Harrison, founder and lead Montessori teacher at Little Friends Learning Loft in Newburgh NY. AMS-certified, SVA-trained, community-rooted. |
| What is Montessori | What is Montessori? · Little Friends Learning Loft | How Montessori education works at our Newburgh preschool — child-led learning, hands-on discovery, mixed ages, and why it prepares kids for kindergarten. |
| Programs | Programs · Montessori Preschool Ages 3–6 | Primary, Kindergarten, Before & After Care, Summer Camp, and Enrichment programs at Little Friends Learning Loft in Newburgh NY. |
| Admissions | Admissions · Little Friends Learning Loft | How to enroll at our Newburgh Montessori preschool. Tour, apply, join. Rolling admissions for ages 3–6. |
| Contact | Contact · Little Friends Learning Loft | Reach Little Friends Learning Loft at the Newburgh JCC, 290 North St, Newburgh NY 12550. Call, email, or visit. |
| FAQ | FAQ · Montessori Preschool Questions Answered | Answers about enrollment, tuition, daily schedule, kindergarten readiness, and what to expect at Little Friends Learning Loft. |
| Blog Index | Blog · Little Friends Learning Loft | Montessori parenting tips, school news, seasonal activities, and community stories from Little Friends Learning Loft in Newburgh. |

### Technical SEO

- **Rendering:** SSG for all public pages — full HTML served to crawlers, no client-side rendering for content
- **Sitemap:** Auto-generated via `next-sitemap` at `/sitemap.xml`
- **Robots.txt:** Allow `/`, Disallow `/portal/*`, `/admin/*`, `/staff/*`, `/api/*`
- **Images:** `next/image` for everything — responsive srcsets, lazy loading, blur placeholders, AVIF/WebP
- **URLs:** Clean, lowercase, hyphenated — no trailing slashes, no query params for content, no numeric IDs
- **Breadcrumbs:** On all interior pages, with `BreadcrumbList` schema
- **301 Redirects:** Map every old Squarespace URL to its new equivalent (see MIGRATION.md)
- **Mobile-first:** All pages responsive, 375px as primary viewport

### URL Structure

```
/                                → Homepage (scrolling sections)
/about/rebecca                   → Rebecca's story
/about/montessori                → What is Montessori
/about/jcc                       → Our Home at the JCC
/about/mission                   → Mission & Values
/programs/primary                → Primary Classroom (ages 3–6)
/programs/kindergarten           → Kindergarten Program
/programs/before-after-care      → Extended Day
/programs/summer-camp            → Summer Camp
/programs/enrichment             → Enrichment Classes
/admissions                      → Admissions overview + inquiry form
/admissions/schedule-tour        → Tour booking
/community                       → Events + Blog index
/community/blog                  → Blog
/community/blog/[slug]           → Individual posts
/community/events                → Upcoming events
/careers                         → Employment
/contact                         → Contact + Map
/newburgh-preschool              → Local SEO landing page
/orange-county-montessori        → Regional SEO landing page
/faq                             → Comprehensive FAQ
```

### Local SEO

**Target keywords:**
- "Montessori preschool Newburgh NY" (primary)
- "preschool near me Hudson Valley"
- "best preschool Newburgh"
- "Montessori school Orange County NY"
- "preschool Newburgh JCC"
- "summer camp Newburgh NY"
- "Jewish preschool Hudson Valley"
- "small class size preschool Newburgh"

**Google Business Profile:**
- Claim/verify → Primary category: "Montessori School" → Secondary: "Preschool", "Child Care Agency"
- Optimized listing with weekly posts, 20+ photos, Q&A responses
- NAP consistency: ALWAYS "Little Friends Learning Loft" / "290 North St, Newburgh, NY 12550"
- Review strategy: target 2+ new reviews per month (admin dashboard has review request tool)
- Admin dashboard "Post to GBP" feature for cross-posting blog/event content

**Local Citations (consistent NAP on all):**

| Directory | Priority |
|---|---|
| Google Business Profile | Critical |
| Apple Maps | Critical |
| Bing Places | High |
| Care.com | High |
| Winnie | High |
| GreatSchools | High |
| Yelp | Medium |
| Niche | Medium |
| Newburgh Chamber of Commerce | Medium |
| Hudson Valley Parent | Medium |
| AMS Montessori Directory | Medium |

**Geographic Landing Pages:**
- `/newburgh-preschool` — "Preschool in Newburgh" queries, community context, driving directions, Google Map
- `/orange-county-montessori` — wider regional reach, directions from Middletown, Goshen, Monroe, Warwick

---

## Structured Data (JSON-LD)

### Site-Wide: EducationalOrganization + LocalBusiness

```json
{
  "@context": "https://schema.org",
  "@type": ["EducationalOrganization", "LocalBusiness"],
  "name": "Little Friends Learning Loft",
  "alternateName": "LFLL",
  "url": "https://littlefriendslearningloft.com",
  "logo": "https://littlefriendslearningloft.com/logo.png",
  "image": "https://littlefriendslearningloft.com/images/og-default.jpg",
  "description": "Montessori preschool at the Newburgh JCC combining child-led learning with Jewish values for children ages 3–6.",
  "address": {
    "@type": "PostalAddress",
    "streetAddress": "290 North St",
    "addressLocality": "Newburgh",
    "addressRegion": "NY",
    "postalCode": "12550",
    "addressCountry": "US"
  },
  "geo": {
    "@type": "GeoCoordinates",
    "latitude": 41.5034,
    "longitude": -74.0104
  },
  "telephone": "+1-XXX-XXX-XXXX",
  "email": "hello@littlefriendslearningloft.com",
  "openingHoursSpecification": [
    {
      "@type": "OpeningHoursSpecification",
      "dayOfWeek": ["Monday", "Tuesday", "Wednesday", "Thursday", "Friday"],
      "opens": "08:00",
      "closes": "15:00"
    }
  ],
  "priceRange": "$$",
  "numberOfStudents": 50,
  "educationalFramework": "Montessori",
  "areaServed": ["Newburgh, NY", "Orange County, NY", "Hudson Valley, NY"],
  "founder": {
    "@type": "Person",
    "name": "Rebecca Harrison",
    "hasCredential": "AMS Early Childhood Montessori Primary Certification"
  },
  "memberOf": {
    "@type": "Organization",
    "name": "Newburgh Jewish Community Center"
  },
  "sameAs": ["https://www.instagram.com/littlefriendslearningloft"]
}
```

### Page-Specific Schema

| Page | Schema Types |
|---|---|
| Homepage | `EducationalOrganization` + `LocalBusiness` |
| Contact | `LocalBusiness` (hours, geo, phone) |
| Admissions | `HowTo` (step-by-step enrollment process) |
| FAQ | `FAQPage` (all Q&A pairs) |
| Each Blog Post | `BlogPosting` |
| Each Event | `Event` |
| Each Job Listing | `JobPosting` |
| Tour Video | `VideoObject` |
| All Interior Pages | `BreadcrumbList` |
| Programs | `Course` (for each program) |

### Contextual FAQ Schema

FAQ sections embedded on specific pages get their own `FAQPage` schema:
- Admissions page → 3–4 admissions FAQs
- Programs page → 3–4 curriculum FAQs
- Summer camp page → 3–4 camp FAQs
- Contact page → 2–3 visiting/location FAQs

---

## AEO — Answer Engine Optimization

### Why This Matters

When parents ask ChatGPT, Claude, Perplexity, Google AI, or Siri "What's the best Montessori preschool near Newburgh NY?", LFLL must appear in synthesized answers. This requires content formatted for AI extraction, entity consistency across the web, and authoritative third-party mentions.

### AEO Content Architecture

**Every public page follows these rules:**

1. **Lead with a direct answer** in the first 40–60 words. AI engines extract the first clear statement.
2. **Question-phrased H2/H3 headings** matching how parents actually ask: "What ages does Little Friends accept?" not "Age Range Information"
3. **Each section is standalone** — AI engines extract individual passages, not whole pages
4. **Paragraphs 3–4 lines max** on public pages
5. **Scannable formats:** short paragraphs, occasional tables, clear definitions
6. **Inverted pyramid:** answer first, expand with detail, then supporting evidence

**Example — Bad vs. Good:**

```
❌ "At Little Friends Learning Loft, we believe in the power of experiential 
   learning. Our philosophy draws from Dr. Maria Montessori's methods..." 
   [3 paragraphs later] "...We accept children ages 3 to 6."

✅ "Little Friends Learning Loft accepts children ages 3 to 6. Our Montessori 
   program at the Newburgh JCC combines child-led learning with hands-on 
   discovery. Most families begin between ages 3 and 4."
```

### Entity Consistency (Critical for AI Trust)

AI engines cross-reference claims across the web. Every mention must be consistent:

| Attribute | Canonical Value |
|---|---|
| Name | Little Friends Learning Loft |
| Type | Montessori preschool |
| Ages | 3–6 |
| Location | 290 North St, Newburgh, NY 12550 |
| Host | Newburgh Jewish Community Center (JCC) |
| Founder | Rebecca Harrison |
| Founded | 2018 |
| Class size | ~50 children across 3 classrooms |
| Teacher ratio | 2 teachers per 12 children |

This must match across: website, Google Business Profile, Yelp, Care.com, Winnie, GreatSchools, Instagram bio, Facebook, any press mentions, and all directory listings.

### Third-Party Citation Building

AI engines trust brands that appear across multiple authoritative sources:

- Claim/optimize Google Business Profile with posts, photos, Q&A
- Create/claim listings: Winnie, Care.com, GreatSchools, Niche, Yelp
- Pursue mentions in Hudson Valley publications (Chronogram, Hudson Valley One, Mid Hudson Times)
- Get listed in Montessori directories (AMS, Montessori Foundation)
- Encourage parent blog posts and reviews linking to the site

### AEO-Optimized Page Example: Montessori (`/about/montessori`)

Structure as Q&A that AI engines parse and cite:

- **H2: "What is Montessori education?"** — 2-sentence answer first, then expand
- **H2: "How does Montessori work at Little Friends?"** — LFLL-specific examples, not generic
- **H2: "What does a typical day look like?"** — hour-by-hour narrative
- **H2: "How is Montessori different from traditional preschool?"** — fair comparison table
- **H2: "Will my child be ready for kindergarten?"** — evidence-based answer with outcomes
- Each section independently extractable by AI engines

### llms.txt Implementation

Create `public/llms.txt` — emerging standard for AI search crawlers:

```
# Little Friends Learning Loft

> Montessori preschool at the Newburgh JCC serving children ages 3–6.
> Combines child-led Montessori education with Jewish cultural values
> including tikkun olam (repairing the world). ~50 children across 3
> classrooms with a 2:12 teacher ratio. Founded 2018 by Rebecca Harrison.

## Key Facts
- Ages: 3–6 years old
- Students: ~50 across 3 classrooms
- Teacher ratio: 2:12
- Location: Newburgh JCC, 290 North St, Newburgh, NY 12550
- Schedule: Monday–Friday, 8:00 AM – 3:00 PM (extended day until 5:30)
- Founded: 2018 by Rebecca Harrison (AMS-certified)
- Philosophy: Montessori + Jewish cultural values (open to all faiths)
- Facilities: 3 classrooms, playground, art house, dance studio, gym, library, garden

## Programs
- Primary Montessori (ages 3–6, full day)
- Kindergarten program
- Before & After Care (7:30–8:00 AM, 3:00–5:30 PM)
- Summer camp
- Enrichment: art, yoga/dance, music, skateboarding

## Differentiators
- Only Montessori preschool in Newburgh
- Access to full JCC facilities (gym, art house, dance studio, social hall)
- Nature-based learning with garden program
- Jewish values (tikkun olam, Shabbat, holidays) integrated — secular program, all faiths welcome
- Work-trade program for families who contribute volunteer hours
- Small, community-rooted school

## Enrollment
- Rolling admissions
- Tour required before enrollment
- Book a tour: https://littlefriendslearningloft.com/admissions/schedule-tour
- Inquire: https://littlefriendslearningloft.com/admissions

## Contact
- Website: https://littlefriendslearningloft.com
- Email: hello@littlefriendslearningloft.com
- Instagram: @littlefriendslearningloft
```

---

## Content Clusters (Blog SEO Strategy)

### Pillar 1: "Choosing a Preschool in Newburgh"

| Post | Target Keyword |
|---|---|
| Preschool vs Daycare: What's the Difference? | preschool vs daycare |
| When Should My Child Start Preschool? | when to start preschool |
| Questions to Ask on a Preschool Tour | preschool tour questions |
| How to Prepare Your Child for Preschool | preparing for preschool |
| What Does Preschool Cost in the Hudson Valley? | preschool cost hudson valley |

### Pillar 2: "Understanding Montessori Education"

| Post | Target Keyword |
|---|---|
| What is Montessori? A Parent's Guide | what is montessori |
| Montessori vs Traditional Preschool | montessori vs traditional |
| Benefits of Montessori (Research-Backed) | montessori benefits |
| Montessori Activities You Can Do at Home | montessori activities at home |
| Is Montessori Right for My Child? | is montessori right for my child |

### Pillar 3: "Kindergarten Readiness"

| Post | Target Keyword |
|---|---|
| Kindergarten Readiness Skills Checklist | kindergarten readiness checklist |
| How Montessori Prepares Kids for School | montessori kindergarten prep |
| Social-Emotional Learning in Preschool | social emotional learning preschool |
| When Your Child Struggles with Separation | separation anxiety preschool |

### Publishing Cadence

- **Pre-launch:** 5 cornerstone pages live (home, about, programs, admissions, FAQ)
- **Month 1–3:** 2 blog posts per month (pillar content + supporting)
- **Month 4+:** 1–2 posts per month minimum
- **Ongoing:** Update FAQ quarterly with new questions from parent inquiries and chatbot analytics

---

## Open Graph Image Strategy

| Page Type | OG Image |
|---|---|
| Homepage | Hero photo with logo overlay (1200×630) |
| About pages | Rebecca photo or classroom photo |
| Programs | Relevant activity photo |
| Blog posts | Post-specific image (AI-assisted caption overlay from admin CMS) |
| Default fallback | Logo on warm yellow `#FFF3C4` background |

Implementation via Next.js `generateMetadata()` on each route:

```typescript
export async function generateMetadata({ params }): Promise<Metadata> {
  const post = await getPost(params.slug);
  return {
    openGraph: {
      title: post.title,
      description: post.excerpt,
      images: [{ url: post.ogImage || '/images/og-default.jpg', width: 1200, height: 630 }],
    },
  };
}
```

---

## Internal Linking Rules

1. Every public page links to 2–3 related pages with descriptive anchor text
2. Homepage links to all primary sections (about, programs, admissions, community, contact)
3. Every blog post links to at least one cornerstone page and one other blog post
4. "Schedule a Tour" CTA appears on every public page (header + contextual)
5. FAQ answers link to relevant detail pages (e.g., tuition Q → admissions page)
6. Footer includes complete navigation to all public pages
7. Breadcrumbs on all interior pages for both UX and structured data

---

## Performance Requirements

### Lighthouse Targets (All Public Pages)

| Category | Target |
|---|---|
| Performance | 90+ |
| Accessibility | 95+ |
| Best Practices | 90+ |
| SEO | 95+ |

### Core Web Vitals

| Metric | Target |
|---|---|
| LCP (Largest Contentful Paint) | < 2.0s |
| INP (Interaction to Next Paint) | < 200ms |
| CLS (Cumulative Layout Shift) | < 0.1 |

### Image Optimization

- AVIF/WebP with JPEG fallback (automatic via `next/image`)
- Responsive sizes: 400w, 800w, 1200w, 1600w
- Lazy loading below the fold
- Hero image preloaded: `<link rel="preload" as="image">`
- Blur placeholder on all images (generated at build via Sharp)

### Font Optimization

- 2 families only: Fredoka One (headings) + Nunito (body)
- Latin subset only (reduces ~60% of font weight)
- Preload critical weights: Nunito 400, 700
- `font-display: swap` on all faces

### Code Optimization

- Code splitting per route (Next.js automatic)
- Tree shaking unused code
- Edge caching on Vercel CDN
- Tailwind CSS purged (< 30KB gzipped)
- JS first load < 100KB gzipped
- No render-blocking scripts on public pages
- Framer Motion animations use `viewport` trigger (no JS on page load)

---

## Monitoring & Measurement

### Tools

| Tool | Purpose | Cost |
|---|---|---|
| Google Search Console | Rankings, impressions, crawl errors, Core Web Vitals | Free |
| Google Analytics 4 | Traffic, behavior, conversion funnels | Free |
| Vercel Analytics | Real User Monitoring, Web Vitals | Free (included) |

### Key Metrics

| Metric | Goal | Review Frequency |
|---|---|---|
| Organic impressions | +50% quarter-over-quarter | Monthly |
| Organic clicks to site | +30% quarter-over-quarter | Monthly |
| "Schedule Tour" conversions from organic | Track absolute count | Monthly |
| Core Web Vitals (all green) | Pass all three metrics | Monthly |
| FAQ page as top landing page | Top 3 organic entries | Monthly |
| Local pack appearance | Appear for "preschool Newburgh" | Weekly |
| AI assistant mentions | Monitor via brand name search | Monthly |
| Blog posts indexed | 100% within 48 hours of publish | Per post |

### Search Console Verification

Add via HTML meta tag during deployment:

```html
<meta name="google-site-verification" content="[verification_code]" />
```

Set in environment variable: `GOOGLE_SITE_VERIFICATION=[code]`
