# CONTENT_STRATEGY.md — Voice, Copy & Content Plan

> Editorial voice, copy guidelines for every surface, blog strategy, email sequences, notification copy, and content governance. Read alongside SEO_AEO.md for keyword strategy and DESIGN_SYSTEM.md for visual pairing.

---

## Brand Voice

### Personality

Little Friends Learning Loft sounds like **a warm, knowledgeable friend who happens to be an amazing preschool teacher.** Not corporate. Not salesy. Not over-the-top "kid talk." Think: the director you'd trust with your child and also want to have coffee with.

### Voice Attributes

| Attribute | What It Sounds Like | What It Does NOT Sound Like |
|---|---|---|
| Warm | "We'd love to meet your family" | "Submit your application today" |
| Confident | "Our Montessori approach works because..." | "We think Montessori might be helpful..." |
| Honest | "Preschool isn't for every 3-year-old" | "Every child thrives in our program!" |
| Playful | "Mud-stained knees are a sign of a good day" | "Students engage in experiential learning" |
| Community | "Our families become friends" | "Join our exclusive community" |
| Grounded | "We're a small school and we like it that way" | "We're the premier Montessori institution in the region" |

### Tone Shift by Context

| Context | Tone | Example |
|---|---|---|
| Public website | Warm, inviting, confident | "Your child will spend mornings choosing their own work — building, painting, counting, reading — at their own pace." |
| Admissions process | Encouraging, clear, step-by-step | "Here's what happens next: we'll send you a confirmation email with everything you need for your tour." |
| Parent portal | Friendly, efficient, direct | "Tuesday's field trip to the farm is confirmed. Boots and layers recommended." |
| Emergency SMS | Calm, authoritative, actionable | "LFLL: Early dismissal today at 1:00 PM due to weather. Please arrange pickup by 1:00." |
| Admin dashboard | Clean, functional, no-nonsense | UI copy is concise — labels, not sentences. Tooltips for explanation. |
| AI chatbot (public) | Helpful, natural, never pushy | "Great question! Our program is for ages 3–6. Would you like to schedule a tour to see the classrooms?" |

### Words We Use / Words We Avoid

| ✅ Use | ❌ Avoid |
|---|---|
| Children, kids | Students, pupils |
| Families | Clients, customers |
| Classrooms | Facilities |
| Teachers | Educators, staff members |
| Learn, discover, explore | Achieve, optimize, excel |
| Our school | Our institution, our program (when referring to the school itself) |
| Tikkun olam (with translation) | "Jewish values" alone (always explain) |

### Writing Rules

1. **Lead with the parent's question, not your answer.** "Will my child be ready for kindergarten?" not "Our kindergarten readiness outcomes"
2. **Short paragraphs.** 3–4 lines max on public pages. Dense paragraph walls lose parents on mobile.
3. **Specific over vague.** "Two teachers for twelve children" not "small class sizes." "Art house, dance studio, gym, garden, three playgrounds" not "extensive facilities."
4. **Show, don't tell.** Photos + captions > adjectives. "Kids rolling out challah dough on Fridays" > "We celebrate Jewish traditions."
5. **No corporate jargon.** Never: synergy, curated, leverage, holistic approach, innovative solutions, tailored experience.
6. **No exclamation marks in headlines.** One per page maximum in body copy. Warmth comes from word choice, not punctuation.

---

## Public Website Copy

### Homepage Copy Guidelines

Each homepage section needs specific copy:

**Hero Section:**
- Headline: short, warm, active (8 words max). Example: "Where Curious Kids Become Confident Learners"
- Subhead: one sentence, location + differentiator. Example: "Montessori preschool at the Newburgh JCC. Small classes. Big adventures."
- CTA: "Schedule a Tour" (primary), "Learn More" (secondary)
- Social proof badge: "Trusted by 50+ families since 2018"

**Section CTAs (consistent across site):**
- Primary CTA: "Schedule a Tour" → `/admissions/schedule-tour`
- Secondary CTA: "Learn More" → relevant detail page
- Inquiry CTA: "Have Questions?" → `/contact` or opens chatbot

**Testimonial Guidelines:**
- Sprinkled throughout the site, not on a dedicated page (Rebecca's preference)
- Format: quote + parent first name + child's classroom. Example: *"Our daughter didn't want to go to school anywhere else." — Maria, Sunflower Room parent*
- 1–2 testimonials per public page section
- Rotate quarterly

### FAQ Copy

Minimum 20 questions organized by category. Each follows the AEO pattern:

```
Q: What ages does Little Friends accept?
A: We accept children ages 3 to 6. Most families join between ages 3 and 4, 
   though we welcome children up to kindergarten age. Readiness for social 
   interaction matters more than a specific birthday. [Link: Learn about our 
   admissions process →]
```

**FAQ Categories:**
- About Our School (6–8 questions)
- Enrollment & Tuition (5–7 questions)
- Daily Schedule & Curriculum (4–6 questions)
- Safety & Policies (3–5 questions)
- Summer Camp (3–4 questions)

---

## Blog Content Strategy

### Content Types

| Type | Frequency | Purpose | Length |
|---|---|---|---|
| Pillar article | 1/quarter | SEO backbone, comprehensive topic coverage | 1,200–1,800 words |
| Supporting post | 2/month | Internal linking to pillars, long-tail keywords | 600–1,000 words |
| School news | As needed | Events, seasonal updates, enrollment announcements | 300–500 words |
| Community story | 1/month | Parent perspective, teacher spotlight, alumni update | 400–800 words |

### Blog Authoring Workflow (Admin Dashboard)

1. Rebecca (or staff) creates draft in admin CMS
2. AI content assist generates SEO title suggestions, meta description, and tag recommendations
3. Rebecca reviews, edits, adds photos from the photo library
4. Publishes → auto-generates sitemap entry, social meta, and optional GBP cross-post
5. Newsletter subscribers notified if "Include in newsletter" is toggled

### Blog Post Template (CMS Fields)

```typescript
interface BlogPost {
  title: string;              // H1, displayed
  slug: string;               // Auto-generated from title, editable
  excerpt: string;            // 1–2 sentences, used for cards and meta
  body: RichText;             // Markdown-based rich text editor
  coverImage: string;         // Required — from photo library or upload
  coverImageAlt: string;      // Required — descriptive alt text
  author: 'rebecca' | 'staff';
  category: BlogCategory;     // 'montessori' | 'school-news' | 'community' | 'parenting' | 'seasonal'
  tags: string[];             // Free-form tags for filtering
  seoTitle?: string;          // Override auto-generated title tag
  seoDescription?: string;    // Override auto-generated meta description
  publishedAt: Date;          // Schedule or publish immediately
  includeInNewsletter: boolean;
  status: 'draft' | 'published' | 'archived';
}
```

---

## Email Sequences

### Lead Nurture (Post-Inquiry, Pre-Tour)

Triggered when someone submits the inquiry form but hasn't booked a tour yet.

| Email | Timing | Subject | Content |
|---|---|---|---|
| 1 | Immediate | "Thanks for reaching out, [Name]" | Warm acknowledgment, next steps, link to schedule tour, Rebecca's photo |
| 2 | Day 3 | "A peek inside our classrooms" | 3 classroom photos, brief daily schedule overview, "still interested?" soft CTA |
| 3 | Day 7 | "What parents say about Little Friends" | 2–3 testimonials, "Book your tour" CTA |
| 4 | Day 14 | "Questions about Montessori?" | Link to FAQ, link to /about/montessori, "We're here to help" |
| 5 | Day 30 | "We'd still love to meet you" | Final gentle nudge, direct reply to Rebecca, no pressure |

**Sequence stops if:** tour is booked, parent replies to any email, parent unsubscribes.

### Post-Tour Follow-Up

| Email | Timing | Subject | Content |
|---|---|---|---|
| 1 | Same day | "Great meeting you today, [Name]" | Personalized (Rebecca sends via admin dashboard), recap of what they saw, application link |
| 2 | Day 3 | "Your child's first day at Little Friends" | What a typical first week looks like, separation tips, "Ready to apply?" |
| 3 | Day 7 | "Any questions after your tour?" | FAQ link, direct reply to Rebecca |

### Enrolled Family Onboarding

| Email | Timing | Subject | Content |
|---|---|---|---|
| 1 | Enrollment confirmed | "Welcome to the Little Friends family!" | What happens next, portal login, required forms checklist |
| 2 | 1 week before start | "Getting ready for [Child]'s first day" | Practical prep tips, what to bring, drop-off procedures, daily schedule |
| 3 | Day before start | "See you tomorrow!" | Classroom name, teacher names, parking, what to expect |
| 4 | Day 3 | "How's it going?" | Check-in, link to resources for separation anxiety, "message us anytime" |
| 5 | Week 2 | "You're part of the community now" | Invite to parent portal features, upcoming events, Facebook group (if applicable) |

### Operational Notifications (Parent Portal)

| Trigger | Channel | Copy Style |
|---|---|---|
| Daily report posted | Email + push | "[Child] had a great day — see the daily report" |
| Tuition payment due | Email | "Your [Month] tuition is due on [Date]. Pay now →" |
| Payment received | Email | "Payment received — thank you, [Name]." |
| Form needs attention | Email + push | "Action needed: [Form Name] for [Child]" |
| Event RSVP open | Email | "[Event Name] on [Date] — RSVP for your family" |
| School closure | SMS + Email + push | "LFLL closed [Date] due to [Reason]. No before/after care." |
| Emergency | SMS (immediate) | "LFLL ALERT: [Message]. [Action required]." |

---

## Newsletter Strategy

### Format

- Monthly email newsletter via Resend
- Built from admin dashboard (template with drag/drop sections)
- Sections: Classroom highlights, upcoming events, parent tip, community spotlight, enrollment update

### Audience Segments

| Segment | Content Focus |
|---|---|
| Current families | School news, events, portal reminders |
| Prospective families | Blog posts, testimonials, enrollment updates |
| Community (non-parent subscribers) | Events, community stories, volunteer opportunities |
| Alumni families | Yearly update, events, referral requests |

### Email Design Rules

- Warm off-white `#FFFDF7` background (matches site)
- Yellow `#F5C518` accent header bar
- Fredoka One headings, Nunito body (web-safe fallbacks: Verdana, Georgia)
- Maximum 600px width
- Single-column layout for mobile
- Every email includes: unsubscribe link, school address, "reply to Rebecca" option

---

## AI Chatbot Copy Guidelines

The public chatbot (see AI_INTEGRATION.md) uses Claude Haiku with a system prompt that enforces the brand voice:

### Chatbot Personality

- Warm, helpful, conversational
- Answers questions about the school, programs, enrollment, visiting
- Never reveals tuition/pricing — redirects to scheduling a tour or inquiry form
- Never reveals teacher names (except Rebecca) — staff privacy
- Never reveals school calendar — portal only
- Provides factual information from the approved knowledge base

### Chatbot Copy Patterns

```
Greeting: "Hi there! I'm here to answer questions about Little Friends 
Learning Loft. What can I help you with?"

When asked about tuition: "We'd love to share tuition details with you — 
we find it works best to discuss during a tour so we can show you everything 
that's included. Would you like to schedule a visit?"

When asked about something outside scope: "That's a great question — 
I'm not sure about that one. Rebecca would be the best person to ask. 
Want me to help you reach her?"

Closing: "Thanks for your interest in Little Friends! If you think of 
more questions, I'm always here."
```

---

## Content Governance

### Who Creates What

| Content Type | Creator | Reviewer |
|---|---|---|
| Public website copy | Joshua (initial), Rebecca (ongoing via CMS) | Rebecca approves all |
| Blog posts | Rebecca or staff via admin dashboard | Rebecca before publish |
| Email sequences | Joshua (initial templates) | Rebecca approves copy |
| Newsletter | Rebecca via admin dashboard | Self-publish |
| Chatbot knowledge base | Joshua (initial), Rebecca (updates) | Test before deploy |
| Parent portal notifications | System-generated from templates | Joshua writes templates |
| SMS emergency alerts | Rebecca via admin dashboard | No review needed (emergency) |

### Content Review Cadence

| Content | Review Frequency |
|---|---|
| Public website copy | Quarterly — check for outdated info |
| FAQ page | Quarterly — add new questions from chatbot analytics and parent inquiries |
| Blog posts | Never edit after publish (add editor's notes if correction needed) |
| Email sequences | Annually — refresh testimonials and links |
| Chatbot knowledge base | Monthly — add new Q&A from chatbot logs |
| Google Business Profile | Weekly — post photo or update |

### Photo Usage Rules

- Children's faces: OK with signed media consent on file
- No full names paired with identifiable photos — first names only
- No identifying details (home addresses, car plates, family medical info)
- Staff photos: Rebecca only on public site (staff preference for privacy)
- All photos run through Sharp pipeline: strip EXIF, resize, format-convert
- Photo credits not required (Rebecca and staff capture all)
- Public-facing galleries: curated by admin, not auto-published
- Parent portal galleries: automatic from daily photo uploads, visible only to authenticated parents

---

## i18n Preparation (Phase 7)

- All hardcoded copy extracted to locale files from Phase 1
- Default locale: `en`
- Future locale: `es` (significant Spanish-speaking population in Newburgh)
- CMS supports per-field translations (blog title in English + Spanish)
- Chatbot can detect Spanish input and respond in Spanish (Claude is multilingual)
- Email templates will have Spanish variants
- Critical forms (enrollment, emergency contact, permission slips) bilingual first
