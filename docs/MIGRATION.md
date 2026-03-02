# MIGRATION.md — Platform Migration Plan

> Strategy for migrating from Squarespace, Brightwheel, and 6 other tools to the unified LFLL application. Covers data extraction, content migration, URL redirects, DNS cutover, and tool-by-tool transition timelines.

---

## Current Tool Inventory

| Tool | What It Does Now | Replacement | Migration Complexity |
|---|---|---|---|
| Squarespace | Public website (5 pages) | Next.js public site | Low — minimal content |
| Brightwheel | Parent communication, attendance, daily reports, billing | Parent portal + admin dashboard | High — active parent data |
| Calendly | Tour scheduling | Built-in tour scheduling (`/admissions/schedule-tour`) | Low — no data to migrate |
| Google Forms | Permission slips, field trips, before/after care signups | Built-in forms system (admin-created, parent-submitted) | Low — recreate forms |
| Google Sheets | Waitlist, tuition tracking, aftercare billing, immunizations | Supabase database with admin dashboard views | Medium — export/import |
| Paper processes | Incident reports, conference forms, enrollment docs | Digital forms in portal + admin | Low — template, no data |
| Slack (photo channel) | Staff photo sharing | Built-in photo management system | Low — new workflow |
| Canva newsletter | Monthly newsletter creation + email | Built-in newsletter builder in admin dashboard | Low — template only |
| Sign-Up Genius | Conference scheduling, volunteer signup | Built-in signup/scheduling in portal | Low — no persistent data |

---

## Phase-by-Phase Migration Timeline

Migration happens gradually — each tool is replaced when its equivalent feature goes live:

| Phase | Week | Tools Replaced | Cutover Action |
|---|---|---|---|
| 1 | 4 | Squarespace | DNS switch, 301 redirects, go live with public site |
| 2 | 8 | Calendly, Sign-Up Genius (partially) | Tour booking live, disable Calendly link |
| 3 | 14 | Google Forms, Paper processes | New forms live in portal, stop using Google Forms |
| 3 | 14 | Slack photo channel | Photo upload in staff tools, archive Slack channel |
| 4 | 18 | Google Sheets (billing) | Stripe billing live, stop manual tracking |
| 5 | 22 | Brightwheel, Google Sheets (compliance) | Full parent portal live, cancel Brightwheel |
| 6 | 26 | Canva newsletter, Sign-Up Genius (fully) | Newsletter builder + event signups live |

**Critical rule:** No tool is decommissioned until its replacement is tested and parents have been onboarded. Run both systems in parallel for 2–4 weeks during transitions.

---

## Squarespace Migration (Phase 1, Week 4)

### Content Inventory

Current Squarespace pages:

| Page | URL | Action |
|---|---|---|
| Home | `/` or `/home` | Replace with new homepage (completely rewritten) |
| What is Montessori | `/what-is-montessori-1` | Replace → `/about/montessori` + 301 redirect |
| FAQs | `/faqs-1` | Replace → `/faq` + 301 redirect (content expanded to 20+ Qs) |
| Contact Us | `/contact-us` | Replace → `/contact` + 301 redirect |
| Summer/Fall 2023 Enrollment | `/summerfall-2023-enrollment` | DELETE (outdated) → 301 redirect to `/admissions` |

### Content to Extract

- Any text content worth preserving (minimal — most will be rewritten)
- Images: download all original-resolution images from Squarespace media library
- No structured data to extract (Squarespace had none)
- No blog posts (didn't exist)
- No forms (used Google Forms externally)

### Squarespace Export Steps

1. **Export content:** Squarespace Settings → Advanced → Import/Export → Export (generates XML)
2. **Download images:** Squarespace doesn't include full-res images in export — manually download from media library or use a scraping tool
3. **Export analytics:** Note current traffic levels in Google Analytics before cutover (baseline)
4. **Document existing URLs:** Screenshot Google Search Console showing indexed pages

### DNS Cutover

1. **Before cutover:** Deploy new site to Vercel, test at preview URL
2. **Update DNS:** Point `littlefriendslearningloft.com` to Vercel
   - Remove Squarespace DNS records
   - Add Vercel's DNS records (A record + CNAME for www)
   - Vercel auto-provisions SSL certificate
3. **Verify:** Site loads at production URL, SSL is active, all pages render
4. **Monitor:** Watch Google Search Console for crawl errors over next 2 weeks

### 301 Redirect Map

Configure in `next.config.js`:

```typescript
// next.config.ts
const nextConfig = {
  async redirects() {
    return [
      // Squarespace URL → New URL
      { source: '/home', destination: '/', permanent: true },
      { source: '/what-is-montessori-1', destination: '/about/montessori', permanent: true },
      { source: '/faqs-1', destination: '/faq', permanent: true },
      { source: '/contact-us', destination: '/contact', permanent: true },
      { source: '/summerfall-2023-enrollment', destination: '/admissions', permanent: true },
      // Catch common Squarespace URL patterns
      { source: '/new-page', destination: '/', permanent: true },
      { source: '/new-page-1', destination: '/', permanent: true },
      // Trailing slash normalization
      { source: '/:path+/', destination: '/:path+', permanent: true },
    ];
  },
};
```

### SEO Continuity

- Submit updated sitemap to Google Search Console immediately after cutover
- Request indexing of all new URLs via Search Console
- Monitor "Coverage" report for errors
- Expect 1–2 week dip in impressions during re-indexing (normal)
- New site will recover and surpass old rankings quickly due to vastly more content + proper SEO

---

## Brightwheel Migration (Phase 5, Week 22)

This is the most complex migration — Brightwheel contains active family data.

### Data to Extract from Brightwheel

| Data Type | Export Method | Destination in LFLL |
|---|---|---|
| Child profiles | CSV export or manual | `children` table in Supabase |
| Parent/guardian contact info | CSV export | `parents` + `family_members` tables |
| Attendance records (historical) | CSV export | `attendance_records` table (archived) |
| Billing history | CSV export | `payment_history` table (archived, reference only) |
| Daily reports (historical) | No bulk export — screenshots or manual | Optional: archive in Supabase storage as PDFs |
| Photos | Manual download | Re-upload to Supabase storage |
| Messages | No export available | Do not migrate — fresh start |

### Brightwheel Export Process

1. **Settings → Program → Export Data** — download all available CSVs
2. **Child-by-child:** Export individual child portfolios if needed
3. **Billing:** Export transaction history from billing section
4. **Photos:** Download in bulk from activity feed (may require manual work)

### Data Import Script

Create a Supabase seed script for Brightwheel data:

```typescript
// supabase/seed-from-brightwheel.ts
import { createClient } from '@supabase/supabase-js';
import Papa from 'papaparse';
import fs from 'fs';

const supabase = createClient(process.env.SUPABASE_URL!, process.env.SUPABASE_SERVICE_ROLE_KEY!);

async function importChildren() {
  const csv = fs.readFileSync('exports/brightwheel-children.csv', 'utf-8');
  const { data } = Papa.parse(csv, { header: true });
  
  for (const row of data) {
    await supabase.from('children').insert({
      first_name: row['First Name'],
      last_name: row['Last Name'],
      date_of_birth: row['DOB'],
      classroom_id: mapClassroom(row['Classroom']),
      enrollment_status: 'active',
      // ... map all fields
    });
  }
}

// Run all imports
async function main() {
  await importChildren();
  await importFamilies();
  await importAttendanceHistory();
  console.log('Brightwheel import complete');
}
```

### Parent Transition Plan

This is the most sensitive migration — parents are actively using Brightwheel daily.

**4 Weeks Before Cutover:**
- Email all families: "We're moving to a new, better system built just for Little Friends"
- Highlight what's improving (one login for everything, modern interface, more features)
- Provide timeline

**2 Weeks Before:**
- Send portal invitations — each family gets a personalized magic link to set up their account
- Include step-by-step screenshots
- Offer 1:1 help sessions (drop-off or pickup time)

**1 Week Before:**
- "Parallel run" — both Brightwheel and new portal active
- Staff posts daily reports in BOTH systems
- Parents can try the new portal while Brightwheel still works

**Cutover Day:**
- Brightwheel is archived (not deleted — maintain read access for 90 days)
- All communication, attendance, billing moves to new system
- Staff uses new system exclusively
- "Tech support" available all week via Rebecca + chatbot

**1 Week After:**
- Check-in email: "How's the new system? Reply with any questions."
- Monitor portal adoption metrics in admin dashboard
- Goal: 90%+ parent portal adoption within 30 days

---

## Google Sheets Migration (Phase 4–5)

### Spreadsheets to Migrate

| Sheet | Data | Destination |
|---|---|---|
| Waitlist | Family names, child info, inquiry date, notes, status | `enrollment_pipeline` table |
| Tuition Tracking | Family, amount, payment dates, balance | Stripe subscriptions + `invoices` table |
| Aftercare Billing | Daily signups, hours, billing | Before/after care module + Stripe |
| Immunization Tracking | Child, vaccine, date, provider, document | `immunization_records` table + Supabase storage |
| Supply Inventory | Items, quantities, budget | `inventory` table in admin dashboard |

### Import Approach

1. Export each Google Sheet as CSV
2. Clean data (normalize names, fix dates, remove duplicates)
3. Run import script that maps columns to Supabase table schema
4. Verify data in admin dashboard
5. Archive Google Sheets (don't delete — reference for discrepancies)

---

## Calendly Migration (Phase 2, Week 8)

Simple transition — no data migration needed.

1. Build tour scheduling feature (`/admissions/schedule-tour`)
2. Integrate with Google Calendar (Rebecca's calendar for availability)
3. Test booking flow end-to-end
4. Update all links pointing to Calendly → new tour page
5. Update Google Business Profile "Book" button URL
6. Update Instagram bio link
7. Cancel Calendly subscription

---

## Google Forms Migration (Phase 3, Week 14)

1. Inventory all active Google Forms (before/after care, field trips, permission slips, volunteer)
2. Recreate each form in the admin dashboard form builder
3. Test form submission → notification → data storage flow
4. Replace Google Form links in parent communications
5. Archive Google Forms (keep for reference)

---

## Paper Process Digitization (Phase 3–5)

| Paper Form | Digital Replacement | Phase |
|---|---|---|
| Incident report | Staff incident form → admin review → parent notification | 3 |
| Conference notes | Teacher conference form → parent portal summary | 5 |
| Enrollment packet | Multi-step digital enrollment form | 2 |
| Permission slips | Per-event digital form in portal | 3 |
| Emergency contact cards | Digital emergency contacts in child profile | 3 |
| Medication authorization | Digital form + Supabase storage for uploads | 3 |

---

## Risk Mitigation

| Risk | Mitigation |
|---|---|
| Parents resist change | 4-week communication plan, parallel run, 1:1 support |
| Data loss during migration | Export everything BEFORE cutover, archive don't delete, test imports |
| SEO traffic dip | 301 redirects, immediate sitemap submission, monitor Search Console |
| Brightwheel billing mid-cycle | Time cutover for start of billing cycle (1st of month) |
| Staff resistance | Train staff 2 weeks before cutover, provide printed quick-reference guides |
| Google Forms responses during transition | Set form descriptions to "This form is being replaced — use [new link]" |

---

## Post-Migration Checklist

After ALL tools have been replaced (end of Phase 6):

- [ ] All Squarespace URLs redirect correctly (spot-check 10 URLs)
- [ ] Google Search Console shows no critical crawl errors
- [ ] All Brightwheel data imported and verified by Rebecca
- [ ] All families have active portal accounts
- [ ] Portal adoption > 90%
- [ ] Stripe billing active for all enrolled families
- [ ] All Google Forms archived with redirect notices
- [ ] All Google Sheets archived
- [ ] Calendly subscription cancelled
- [ ] Brightwheel subscription cancelled (after 90-day archive period)
- [ ] Canva subscription cancelled (if newsletter builder replaces it)
- [ ] Sign-Up Genius no longer needed
- [ ] Instagram bio link updated to new site
- [ ] Google Business Profile URLs all updated
- [ ] All directory listings (Care.com, Winnie, etc.) updated with new URL
